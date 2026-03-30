# Chapter 9: Observation Handler Bridge — Connect Tracing to Micrometer Observation

> **What you'll build**: Four classes (1 interface + 3 implementations) that bridge Micrometer
> Observation's lifecycle to tracing spans — so that every `Observation.start()` automatically
> creates, tags, scopes, and ends a span. This is how Spring Boot auto-configures tracing for
> HTTP, messaging, and database observations.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// Wire tracing into the Observation API
ObservationRegistry registry = ObservationRegistry.create();
registry.observationConfig()
    .observationHandler(new DefaultTracingObservationHandler(tracer));

// Now every Observation automatically creates a span
Observation observation = Observation.start("my-operation", registry);
try (Observation.Scope scope = observation.openScope()) {
    // span is automatically in scope here
    observation.lowCardinalityKeyValue("result", "success");
} finally {
    observation.stop();  // span is named, tagged, and ended here
}
```

**Why this matters**: Up to now, tracing required explicit span creation — calling
`tracer.nextSpan().name("x").start()` and managing scope manually. The Observation API
is Micrometer's *instrumentation* abstraction — it captures "something happened" without
knowing whether to create metrics, spans, or logs. The handlers we build here are the
bridge that says "when an observation starts, create a span."

This is the pattern Spring Boot uses to auto-instrument HTTP clients, servers, messaging,
and database operations. A single `@Observed` annotation or framework integration creates
an observation, and these handlers silently create the corresponding trace spans.

### The Three Handlers

```
TracingObservationHandler<T>          ← Base interface with TracingContext
├── DefaultTracingObservationHandler  ← Local spans (db queries, method calls)
├── PropagatingSenderTracingOH<T>     ← Outgoing requests (HTTP client, producer)
└── PropagatingReceiverTracingOH<T>   ← Incoming requests (HTTP server, consumer)
```

**Why three?** The lifecycle of a span depends on whether it's:
1. **Local** — create a child of the current span, no propagation needed
2. **Outgoing** — create a CLIENT/PRODUCER span and *inject* trace context into the carrier
3. **Incoming** — *extract* trace context from the carrier and create a SERVER/CONSUMER span

### The Lifecycle Bridge

```
Observation lifecycle          TracingObservationHandler           Span lifecycle
─────────────────              ──────────────────────              ──────────────
onStart(context)          →    create span, store in TracingCtx    span.start()
onScopeOpened(context)    →    maybeScope(span.context())          put in ThreadLocal
onEvent(event, context)   →    span.event(eventName)               record event
onError(context)          →    span.error(throwable)               record error
onScopeClosed(context)    →    scope.close()                       restore previous
onStop(context)           →    name, tag, end span                 span.end()
```

**Critical design choice**: The span name and tags are set at `onStop`, not `onStart`.
This is because the observation context accumulates key-values throughout its lifecycle —
the contextual name might come from HTTP response parsing, and tags are added progressively.

---

## 2. Client Usage & Tests

### 2a. Testing Local Observations

```java
@Test
void shouldCreateSpan_WhenObservationStartedAndStopped() {
    ObservationRegistry registry = ObservationRegistry.create();
    registry.observationConfig()
            .observationHandler(new DefaultTracingObservationHandler(tracer));

    Observation observation = Observation.start("my-operation", registry);
    observation.stop();

    SimpleSpan span = tracer.onlySpan();
    assertThat(span.getName()).isEqualTo("my-operation");
    assertThat(span.isStarted()).isTrue();
    assertThat(span.isEnded()).isTrue();
}
```

### 2b. Testing Outgoing Request (Sender)

```java
@Test
void shouldCreateSenderSpan_AndInjectTraceContext() {
    ObservationRegistry registry = ObservationRegistry.create();
    registry.observationConfig()
            .observationHandler(
                new PropagatingSenderTracingObservationHandler<>(tracer, propagator));

    Map<String, String> headers = new HashMap<>();
    SenderContext<Map<String, String>> senderCtx =
            new SenderContext<>((carrier, key, value) -> carrier.put(key, value), Kind.CLIENT);
    senderCtx.setCarrier(headers);

    Observation observation = Observation.start("http.client", () -> senderCtx, registry);
    observation.lowCardinalityKeyValue("http.method", "GET");
    observation.stop();

    SimpleSpan span = tracer.onlySpan();
    assertThat(span.getKind()).isEqualTo(Span.Kind.CLIENT);

    // Trace context was injected into the carrier
    assertThat(headers.get("X-B3-TraceId")).isEqualTo(span.context().traceId());
}
```

### 2c. Testing Incoming Request (Receiver)

```java
@Test
void shouldExtractTraceContext_AndCreateServerSpan() {
    ObservationRegistry registry = ObservationRegistry.create();
    registry.observationConfig()
            .observationHandler(
                new PropagatingReceiverTracingObservationHandler<>(tracer, propagator));

    Map<String, String> incomingHeaders = new HashMap<>();
    incomingHeaders.put("X-B3-TraceId", "abc123");
    incomingHeaders.put("X-B3-SpanId", "def456");

    ReceiverContext<Map<String, String>> receiverCtx =
            new ReceiverContext<>((carrier, key) -> carrier.get(key), Kind.SERVER);
    receiverCtx.setCarrier(incomingHeaders);

    Observation observation = Observation.start("http.server", () -> receiverCtx, registry);
    observation.stop();

    SimpleSpan span = tracer.onlySpan();
    assertThat(span.getKind()).isEqualTo(Span.Kind.SERVER);
    assertThat(span.context().traceId()).isEqualTo("abc123");
    assertThat(span.context().parentId()).isEqualTo("def456");
}
```

### 2d. Testing Composite Handler Routing

In Spring Boot, all three handlers are registered in a `FirstMatchingCompositeObservationHandler`.
The propagating handlers match `SenderContext`/`ReceiverContext`, and the default handler catches
everything else:

```java
registry.observationConfig()
    .observationHandler(
        new ObservationHandler.FirstMatchingCompositeObservationHandler(
            new PropagatingSenderTracingObservationHandler<>(tracer, propagator),
            new PropagatingReceiverTracingObservationHandler<>(tracer, propagator),
            new DefaultTracingObservationHandler(tracer)));
```

---

## 3. Implementing the Call Chain — Building Each Layer Top-Down

### 3a. TracingObservationHandler — The Base Interface

This is the heart of the bridge. It's an interface with one abstract method (`getTracer()`)
and many default methods that implement the observation-to-span lifecycle mapping.

**The TracingContext inner class**:

The key challenge: how do you carry span state between separate lifecycle callbacks
(`onStart`, `onScopeOpened`, `onStop`)? The solution is `TracingContext` — a simple POJO
stored inside `Observation.Context` via `computeIfAbsent`:

```java
class TracingContext {
    private Span span;
    private CurrentTraceContext.Scope scope;
    // getters, setters, convenience setSpanAndScope
}
```

This uses the observation context as a key-value store, with the class object itself as the key.
Every lifecycle callback retrieves the same `TracingContext` via:

```java
default TracingContext getTracingContext(T context) {
    return context.computeIfAbsent(TracingContext.class, clazz -> new TracingContext());
}
```

**Default lifecycle methods**:

```java
// onScopeOpened: put span's trace context into thread-local scope
default void onScopeOpened(T context) {
    TracingContext tracingContext = getTracingContext(context);
    Span span = tracingContext.getSpan();
    if (span == null) return;
    TraceContext traceContext = span.context();
    CurrentTraceContext.Scope newScope = getTracer().currentTraceContext().maybeScope(traceContext);
    tracingContext.setScope(newScope);
}

// onScopeClosed: restore previous trace context
default void onScopeClosed(T context) {
    CurrentTraceContext.Scope scope = getTracingContext(context).getScope();
    if (scope != null) {
        scope.close();
        getTracingContext(context).setScope(null);
    }
}

// onError: record error on span
default void onError(T context) {
    if (context.getError() != null) {
        getRequiredSpan(context).error(context.getError());
    }
}

// onEvent: record event on span
default void onEvent(Observation.Event event, T context) {
    getRequiredSpan(context).event(event.getContextualName());
}
```

**Helper methods**:

- `tagSpan(context, span)` — iterates all key-values, converts to span tags
- `getSpanName(context)` — prefers contextual name over generic name
- `getParentSpan(context)` — resolves parent span from parent observation's TracingContext
- `supportsContext(context)` — defaults to accepting any non-null context

### 3b. DefaultTracingObservationHandler — Local Spans

The simplest handler. Only implements `onStart` and `onStop`:

```java
public class DefaultTracingObservationHandler
        implements TracingObservationHandler<Observation.Context> {

    private final Tracer tracer;

    @Override
    public void onStart(Observation.Context context) {
        Span parentSpan = getParentSpan(context);
        Span childSpan = (parentSpan != null)
                ? this.tracer.nextSpan(parentSpan)
                : this.tracer.nextSpan();
        childSpan.start();
        getTracingContext(context).setSpan(childSpan);
    }

    @Override
    public void onStop(Observation.Context context) {
        Span span = getRequiredSpan(context);
        span.name(getSpanName(context));  // set name at stop, not start
        tagSpan(context, span);           // apply all accumulated key-values
        endSpan(context, span);           // end the span
    }
}
```

### 3c. PropagatingSenderTracingObservationHandler — Outgoing

This handler bridges two propagation systems. The observation layer has its own
`Propagator.Setter` (in `io.micrometer.observation.transport`), and the tracing layer has
its own (in `io.simpletracing.propagation`). The handler adapts between them with a lambda:

```java
@Override
public void onStart(T context) {
    Span.Builder builder = this.tracer.spanBuilder();
    builder.kind(Span.Kind.valueOf(context.getKind().name()));  // Kind bridge

    if (parentSpan != null) {
        builder.setParent(parentSpan.context());
    }

    Span childSpan = builder.start();

    // Bridge: observation Setter → tracing Setter
    SenderContext<Object> senderContext = (SenderContext<Object>) context;
    io.micrometer.observation.transport.Propagator.Setter<Object> observationSetter =
            senderContext.getSetter();
    this.propagator.inject(childSpan.context(), senderContext.getCarrier(),
            (c, key, value) -> observationSetter.set(c, key, value));

    getTracingContext(context).setSpan(childSpan);
}
```

The `(c, key, value) -> observationSetter.set(c, key, value)` lambda is the Adapter pattern
at its most concise — one lambda bridges two ecosystems.

**`supportsContext`** returns `context instanceof SenderContext` to restrict to sender observations only.

### 3d. PropagatingReceiverTracingObservationHandler — Incoming

The mirror of the sender. Instead of injecting, it extracts:

```java
@Override
public void onStart(T context) {
    ReceiverContext<Object> receiverContext = (ReceiverContext<Object>) context;
    io.micrometer.observation.transport.Propagator.Getter<Object> observationGetter =
            receiverContext.getGetter();

    // Bridge: observation Getter → tracing Getter
    Span.Builder extractedBuilder = this.propagator.extract(
            receiverContext.getCarrier(),
            (c, key) -> observationGetter.get(c, key));

    extractedBuilder.kind(Span.Kind.valueOf(context.getKind().name()));

    Span span = extractedBuilder.start();
    getTracingContext(context).setSpan(span);
}
```

**`supportsContext`** returns `context instanceof ReceiverContext`.

---

## 4. Try It Yourself

1. **Add a `customizeSenderSpan` hook**: The real framework has an empty `customizeSenderSpan(T context, Span span)`
   method that subclasses override to add custom tags. Add this hook to the sender handler's `onStop`.

2. **Support `onScopeReset`**: The real framework's `onScopeReset` drains all nested scopes.
   Add this method to `TracingObservationHandler` — it should close the current scope and call
   `maybeScope(null)` to clear the thread-local.

3. **Per-thread scope tracking**: The real `TracingContext` uses a `ConcurrentHashMap<Thread, Scope>`
   instead of a single `Scope` field, supporting observations that open/close scopes on different
   threads. Upgrade `TracingContext` to use this pattern.

---

## 5. Why This Works

### The Observation Context as State Carrier
The `Observation.Context` is a `ConcurrentHashMap`-backed key-value store. By using
`computeIfAbsent(TracingContext.class, ...)`, we get a single shared `TracingContext` instance
across all lifecycle callbacks without any coupling between handlers. The class object itself
is the key — no string constants, no possibility of collision.

### Deferred Naming and Tagging
Setting the span name and tags at `onStop` (not `onStart`) is a crucial pattern. In a real
HTTP server, the observation starts before the request is processed — the contextual name
("GET /users/123") isn't known until the response is being built. By deferring, the span
captures the final, most accurate data.

### The Setter/Getter Bridge
The observation layer and tracing layer both define `Setter<C>` and `Getter<C>` interfaces.
They have identical shapes but are different types in different packages. The handlers bridge
them with a lambda — this is the Adapter pattern at its most minimal. One line of code bridges
two ecosystems without either needing to know about the other.

### supportsContext Discrimination
`DefaultTracingObservationHandler.supportsContext` returns `true` for any context.
The propagating handlers restrict to `SenderContext` / `ReceiverContext`. When registered in a
`FirstMatchingCompositeObservationHandler`, more specific handlers take priority. This is a
Strategy pattern where the context type determines which strategy executes.

---

## 6. What We Enhanced

| Area | Before Feature 9 | After Feature 9 |
|------|-------------------|-----------------|
| Span creation | Manual: `tracer.nextSpan().start()` | Automatic: `Observation.start()` creates a span |
| Scope management | Manual: `try (SpanInScope ws = ...)` | Automatic: `observation.openScope()` scopes the span |
| Tagging | Manual: `span.tag("k", "v")` | Automatic: `observation.lowCardinalityKeyValue(...)` |
| Propagation (send) | Manual: `propagator.inject(...)` | Automatic: Sender handler injects on start |
| Propagation (recv) | Manual: `propagator.extract(...)` | Automatic: Receiver handler extracts on start |
| Error recording | Manual: `span.error(e)` | Automatic: `observation.error(e)` records on span |
| **External dep** | None | `micrometer-observation:1.15.0` added to API module |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplifications |
|------------------|----------------------|---------------------|
| `TracingObservationHandler` | `io.micrometer.tracing.handler.TracingObservationHandler` | Single `Scope` field vs. per-thread `ConcurrentHashMap<Thread, Scope>`; no `RevertingScope` nesting; no baggage scope wrapping; simplified `getParentSpan` (no `currentSpan()` comparison) |
| `TracingContext` | inner class of same | Single scope vs. per-thread map |
| `DefaultTracingObservationHandler` | `io.micrometer.tracing.handler.DefaultTracingObservationHandler` | Faithful — same `onStart`/`onStop` structure |
| `PropagatingSenderTracingObservationHandler` | `io.micrometer.tracing.handler.PropagatingSenderTracingObservationHandler` | No `customizeSenderSpan` hook; no URI parsing for remote IP/port |
| `PropagatingReceiverTracingObservationHandler` | `io.micrometer.tracing.handler.PropagatingReceiverTracingObservationHandler` | No `customizeExtractedSpan`/`customizeReceiverSpan` hooks; no URI parsing |

### Source File References

Real source files in `micrometer-tracing/src/main/java/io/micrometer/tracing/handler/`:
- `TracingObservationHandler.java` — the full interface with `RevertingScope`, per-thread scopes, baggage integration
- `DefaultTracingObservationHandler.java` — local span handler
- `PropagatingSenderTracingObservationHandler.java` — sender with `createSenderSpan()` and `customizeSenderSpan()` hooks
- `PropagatingReceiverTracingObservationHandler.java` — receiver with `customizeExtractedSpan()` and `customizeReceiverSpan()` hooks
- `RevertingScope.java` — package-private scope nesting helper

---

## 8. Complete Code

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/handler/TracingObservationHandler.java`

```java
package io.simpletracing.handler;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationHandler;
import io.micrometer.observation.ObservationView;
import io.simpletracing.CurrentTraceContext;
import io.simpletracing.Span;
import io.simpletracing.TraceContext;
import io.simpletracing.Tracer;

public interface TracingObservationHandler<T extends Observation.Context> extends ObservationHandler<T> {

    Tracer getTracer();

    class TracingContext {

        private Span span;

        private CurrentTraceContext.Scope scope;

        public Span getSpan() {
            return this.span;
        }

        public void setSpan(Span span) {
            this.span = span;
        }

        public CurrentTraceContext.Scope getScope() {
            return this.scope;
        }

        public void setScope(CurrentTraceContext.Scope scope) {
            this.scope = scope;
        }

        public void setSpanAndScope(Span span, CurrentTraceContext.Scope scope) {
            this.span = span;
            this.scope = scope;
        }

    }

    @Override
    default void onScopeOpened(T context) {
        TracingContext tracingContext = getTracingContext(context);
        Span span = tracingContext.getSpan();
        if (span == null) {
            return;
        }
        TraceContext traceContext = span.context();
        CurrentTraceContext.Scope newScope = getTracer().currentTraceContext().maybeScope(traceContext);
        tracingContext.setScope(newScope);
    }

    @Override
    default void onScopeClosed(T context) {
        TracingContext tracingContext = getTracingContext(context);
        CurrentTraceContext.Scope scope = tracingContext.getScope();
        if (scope != null) {
            scope.close();
            tracingContext.setScope(null);
        }
    }

    @Override
    default void onError(T context) {
        if (context.getError() != null) {
            getRequiredSpan(context).error(context.getError());
        }
    }

    @Override
    default void onEvent(Observation.Event event, T context) {
        getRequiredSpan(context).event(event.getContextualName());
    }

    default void tagSpan(T context, Span span) {
        context.getAllKeyValues().forEach(keyValue -> {
            if ("error".equalsIgnoreCase(keyValue.getKey())) {
                span.error(new RuntimeException(keyValue.getValue()));
            }
            else {
                span.tag(keyValue.getKey(), keyValue.getValue());
            }
        });
    }

    default String getSpanName(T context) {
        String contextualName = context.getContextualName();
        if (contextualName != null && !contextualName.isBlank()) {
            return contextualName;
        }
        return context.getName();
    }

    default TracingContext getTracingContext(T context) {
        return context.computeIfAbsent(TracingContext.class, clazz -> new TracingContext());
    }

    default Span getRequiredSpan(T context) {
        TracingContext tracingContext = getTracingContext(context);
        Span span = tracingContext.getSpan();
        if (span == null) {
            throw new IllegalStateException("Span is not set in TracingContext");
        }
        return span;
    }

    default Span getParentSpan(T context) {
        ObservationView parentObservation = context.getParentObservation();
        if (parentObservation == null) {
            return null;
        }
        Observation.ContextView parentContextView = parentObservation.getContextView();
        TracingContext parentTracingContext = parentContextView.get(TracingContext.class);
        if (parentTracingContext != null) {
            return parentTracingContext.getSpan();
        }
        return null;
    }

    default void endSpan(T context, Span span) {
        span.end();
    }

    @Override
    default boolean supportsContext(Observation.Context context) {
        return context != null;
    }

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/handler/DefaultTracingObservationHandler.java`

```java
package io.simpletracing.handler;

import io.micrometer.observation.Observation;
import io.simpletracing.Span;
import io.simpletracing.Tracer;

public class DefaultTracingObservationHandler implements TracingObservationHandler<Observation.Context> {

    private final Tracer tracer;

    public DefaultTracingObservationHandler(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public void onStart(Observation.Context context) {
        Span parentSpan = getParentSpan(context);
        Span childSpan;
        if (parentSpan != null) {
            childSpan = this.tracer.nextSpan(parentSpan);
        }
        else {
            childSpan = this.tracer.nextSpan();
        }
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

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/handler/PropagatingSenderTracingObservationHandler.java`

```java
package io.simpletracing.handler;

import io.micrometer.observation.Observation;
import io.micrometer.observation.transport.SenderContext;
import io.simpletracing.Span;
import io.simpletracing.Tracer;
import io.simpletracing.propagation.Propagator;

public class PropagatingSenderTracingObservationHandler<T extends SenderContext<?>>
        implements TracingObservationHandler<T> {

    private final Tracer tracer;

    private final Propagator propagator;

    public PropagatingSenderTracingObservationHandler(Tracer tracer, Propagator propagator) {
        this.tracer = tracer;
        this.propagator = propagator;
    }

    @SuppressWarnings("unchecked")
    @Override
    public void onStart(T context) {
        Span parentSpan = getParentSpan(context);
        Span.Builder builder = this.tracer.spanBuilder();

        builder.kind(Span.Kind.valueOf(context.getKind().name()));

        if (parentSpan != null) {
            builder.setParent(parentSpan.context());
        }

        String remoteServiceName = context.getRemoteServiceName();
        if (remoteServiceName != null && !remoteServiceName.isBlank()) {
            builder.remoteServiceName(remoteServiceName);
        }

        Span childSpan = builder.start();

        SenderContext<Object> senderContext = (SenderContext<Object>) context;
        Object carrier = senderContext.getCarrier();
        if (carrier != null) {
            io.micrometer.observation.transport.Propagator.Setter<Object> observationSetter =
                    senderContext.getSetter();
            this.propagator.inject(childSpan.context(), carrier,
                    (c, key, value) -> observationSetter.set(c, key, value));
        }

        getTracingContext(context).setSpan(childSpan);
    }

    @Override
    public void onStop(T context) {
        Span span = getRequiredSpan(context);
        tagSpan(context, span);
        span.name(getSpanName(context));
        endSpan(context, span);
    }

    @Override
    public boolean supportsContext(Observation.Context context) {
        return context instanceof SenderContext;
    }

    @Override
    public Tracer getTracer() {
        return this.tracer;
    }

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/handler/PropagatingReceiverTracingObservationHandler.java`

```java
package io.simpletracing.handler;

import io.micrometer.observation.Observation;
import io.micrometer.observation.transport.ReceiverContext;
import io.simpletracing.Span;
import io.simpletracing.Tracer;
import io.simpletracing.propagation.Propagator;

public class PropagatingReceiverTracingObservationHandler<T extends ReceiverContext<?>>
        implements TracingObservationHandler<T> {

    private final Tracer tracer;

    private final Propagator propagator;

    public PropagatingReceiverTracingObservationHandler(Tracer tracer, Propagator propagator) {
        this.tracer = tracer;
        this.propagator = propagator;
    }

    @SuppressWarnings("unchecked")
    @Override
    public void onStart(T context) {
        ReceiverContext<Object> receiverContext = (ReceiverContext<Object>) context;
        Object carrier = receiverContext.getCarrier();

        io.micrometer.observation.transport.Propagator.Getter<Object> observationGetter =
                receiverContext.getGetter();

        Span.Builder extractedBuilder = this.propagator.extract(carrier,
                (c, key) -> observationGetter.get(c, key));

        extractedBuilder.kind(Span.Kind.valueOf(context.getKind().name()));

        String remoteServiceName = context.getRemoteServiceName();
        if (remoteServiceName != null && !remoteServiceName.isBlank()) {
            extractedBuilder.remoteServiceName(remoteServiceName);
        }

        Span span = extractedBuilder.start();
        getTracingContext(context).setSpan(span);
    }

    @Override
    public void onStop(T context) {
        Span span = getRequiredSpan(context);
        tagSpan(context, span);
        span.name(getSpanName(context));
        endSpan(context, span);
    }

    @Override
    public boolean supportsContext(Observation.Context context) {
        return context instanceof ReceiverContext;
    }

    @Override
    public Tracer getTracer() {
        return this.tracer;
    }

}
```

### `[MODIFIED] simple-tracing-api/build.gradle`

```groovy
// Core API module — defines the tracing facade interfaces
// Maps to: micrometer-tracing in the real repository
//
// This module has NO implementation — only interfaces and annotations.
// Concrete implementations live in simple-tracing-test (in-memory)
// and simple-tracing-bridge-brave (Brave adapter).

dependencies {
    // Observation API — needed for TracingObservationHandler (Feature 9)
    api 'io.micrometer:micrometer-observation:1.15.0'

    // AOP Alliance — needed for annotation-based tracing (Feature 10)
    // Uncomment when implementing Feature 10:
    // api 'aopalliance:aopalliance:1.0'
    // implementation 'org.aspectj:aspectjweaver:1.9.22.1'
}
```

### `[NEW] simple-tracing-test/src/test/java/io/simpletracing/test/simple/ObservationHandlerTest.java`

```java
package io.simpletracing.test.simple;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationHandler;
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.transport.Kind;
import io.micrometer.observation.transport.ReceiverContext;
import io.micrometer.observation.transport.SenderContext;
import io.simpletracing.Span;
import io.simpletracing.TraceContext;
import io.simpletracing.handler.DefaultTracingObservationHandler;
import io.simpletracing.handler.PropagatingReceiverTracingObservationHandler;
import io.simpletracing.handler.PropagatingSenderTracingObservationHandler;
import io.simpletracing.handler.TracingObservationHandler;
import io.simpletracing.propagation.Propagator;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ObservationHandlerTest {

    private final SimpleTracer tracer = new SimpleTracer();

    private final SimplePropagator propagator = new SimplePropagator(tracer);

    @AfterEach
    void cleanup() {
        SimpleTracer.resetCurrentSpan();
    }

    // Tests for Default, Sender, Receiver, Composite, supportsContext, TracingContext
    // (see full source in src/test/java/io/simpletracing/test/simple/ObservationHandlerTest.java)

    private static class SimplePropagator implements Propagator {

        private final SimpleTracer tracer;

        SimplePropagator(SimpleTracer tracer) {
            this.tracer = tracer;
        }

        @Override
        public List<String> fields() {
            return List.of("X-B3-TraceId", "X-B3-SpanId", "X-B3-ParentSpanId", "X-B3-Sampled");
        }

        @Override
        public <C> void inject(TraceContext context, C carrier, Setter<C> setter) {
            setter.set(carrier, "X-B3-TraceId", context.traceId());
            setter.set(carrier, "X-B3-SpanId", context.spanId());
            if (context.parentId() != null) {
                setter.set(carrier, "X-B3-ParentSpanId", context.parentId());
            }
            if (context.sampled() != null) {
                setter.set(carrier, "X-B3-Sampled", context.sampled() ? "1" : "0");
            }
        }

        @Override
        public <C> Span.Builder extract(C carrier, Getter<C> getter) {
            String traceId = getter.get(carrier, "X-B3-TraceId");
            if (traceId == null) {
                return Span.Builder.NOOP;
            }
            String spanId = getter.get(carrier, "X-B3-SpanId");

            SimpleTraceContext extractedContext = new SimpleTraceContext();
            extractedContext.setTraceId(traceId);
            extractedContext.setSpanId(spanId);

            String sampled = getter.get(carrier, "X-B3-Sampled");
            if (sampled != null) {
                extractedContext.setSampled(
                        "1".equals(sampled) || "true".equalsIgnoreCase(sampled));
            }

            return this.tracer.spanBuilder().setParent(extractedContext);
        }

    }

}
```
