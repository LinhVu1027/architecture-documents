# Chapter 9: Propagating Observation Handlers

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `DefaultTracingObservationHandler` creates spans from observations, but only for local/in-process operations. `SenderContext`/`ReceiverContext` carry carrier objects and propagation accessors, and `Propagator` can inject/extract trace context — but nothing connects them. | When an observation represents cross-process communication (an HTTP call, a message publish), the handler doesn't inject or extract trace context — distributed traces are broken across service boundaries. | Build two specialized observation handlers: one that **injects** trace context into outbound carriers (sender) and one that **extracts** trace context from inbound carriers (receiver), enabling automatic distributed trace propagation through the Observation API. |

---

## 9.1 The Integration Point

The integration point for this feature is the `ObservationRegistry` — specifically, how the three tracing handlers are registered together. The sender and receiver handlers must be wrapped in a `FirstMatchingCompositeObservationHandler` alongside the default handler so that only **one** handler fires per observation:

```
┌─ FirstMatchingCompositeObservationHandler ──────────────────┐
│                                                              │
│  SenderContext?  ──→  PropagatingSenderTracingObsHandler     │
│  ReceiverContext? ──→ PropagatingReceiverTracingObsHandler   │
│  anything else?  ──→  DefaultTracingObservationHandler       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```java
registry.observationConfig()
    .observationHandler(
        new ObservationHandler.FirstMatchingCompositeObservationHandler(
            new PropagatingSenderTracingObservationHandler<>(tracer, propagator),
            new PropagatingReceiverTracingObservationHandler<>(tracer, propagator),
            new DefaultTracingObservationHandler(tracer)));
```

**Direction:** This integration point connects the **Observation lifecycle** to the **Propagation layer**. To make it work, we need to build:
- `PropagatingSenderTracingObservationHandler` — creates a span and injects its context into the outbound carrier
- `PropagatingReceiverTracingObservationHandler` — extracts context from the inbound carrier and creates a child span

Two key decisions here:

1. **Why `FirstMatchingCompositeObservationHandler`?** Without it, `DefaultTracingObservationHandler`'s inherited `supportsContext()` returns `true` for ALL contexts (including `SenderContext`), causing duplicate span creation. The composite ensures mutual exclusion.
2. **Why order matters?** The sender and receiver handlers are registered before the default handler. `FirstMatchingCompositeObservationHandler` picks the first match, so specific handlers must come before the catch-all.

## 9.2 The Sender Handler

The sender handler activates when the observation's context is a `SenderContext`. It creates a span with the right `Kind`, then **injects** the span's trace context into the carrier (e.g., HTTP headers) so the downstream service can extract it.

**New file:** `src/main/java/dev/linhvu/tracing/handler/PropagatingSenderTracingObservationHandler.java`

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.propagation.Propagator;
import dev.linhvu.tracing.transport.SenderContext;

@SuppressWarnings({"rawtypes", "unchecked"})
public class PropagatingSenderTracingObservationHandler<T extends SenderContext>
        implements TracingObservationHandler<T> {

    private final Tracer tracer;
    private final Propagator propagator;

    public PropagatingSenderTracingObservationHandler(Tracer tracer, Propagator propagator) {
        this.tracer = tracer;
        this.propagator = propagator;
    }

    @Override
    public void onStart(T context) {
        Span childSpan = createSenderSpan(context);
        // Inject trace context into the carrier so the receiver can extract it
        this.propagator.inject(childSpan.context(), context.getCarrier(),
                (carrier, key, value) -> context.getSetter().set(carrier, key, value));
        getTracingContext(context).setSpan(childSpan);
    }

    public Span createSenderSpan(T context) {
        Span parentSpan = getParentSpan(context);
        Span.Builder builder = getTracer().spanBuilder()
                .kind(Span.Kind.valueOf(context.getKind().name()));
        if (parentSpan != null) {
            builder = builder.setParent(parentSpan.context());
        }
        if (context.getRemoteServiceName() != null) {
            builder = builder.remoteServiceName(context.getRemoteServiceName());
        }
        if (context.getRemoteServiceAddress() != null) {
            try {
                java.net.URI uri = java.net.URI.create(context.getRemoteServiceAddress());
                builder = builder.remoteIpAndPort(uri.getHost(), uri.getPort());
            }
            catch (Exception ex) {
                // Silently ignore malformed URIs in the simplified version
            }
        }
        return builder.start();
    }

    @Override
    public void onStop(T context) {
        Span span = getRequiredSpan(context);
        tagSpan(context, span);
        String name = context.getContextualName() != null
                ? context.getContextualName() : context.getName();
        if (name != null) {
            span.name(name);
        }
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

The critical line is `onStart`: it creates the span, then immediately calls `propagator.inject()` to write the `traceparent` header into the carrier. The lambda `(carrier, key, value) -> context.getSetter().set(carrier, key, value)` bridges the tracing-layer `Propagator.Setter` to the transport-layer `Propagator.Setter` — these are two different interfaces in two different packages.

## 9.3 The Receiver Handler

The mirror image of the sender — it activates for `ReceiverContext` and **extracts** trace context from the inbound carrier to continue the distributed trace.

**New file:** `src/main/java/dev/linhvu/tracing/handler/PropagatingReceiverTracingObservationHandler.java`

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.propagation.Propagator;
import dev.linhvu.tracing.transport.ReceiverContext;

@SuppressWarnings({"rawtypes", "unchecked"})
public class PropagatingReceiverTracingObservationHandler<T extends ReceiverContext>
        implements TracingObservationHandler<T> {

    private final Tracer tracer;
    private final Propagator propagator;

    public PropagatingReceiverTracingObservationHandler(Tracer tracer, Propagator propagator) {
        this.tracer = tracer;
        this.propagator = propagator;
    }

    @Override
    public void onStart(T context) {
        // Bridge the transport-layer Getter to the tracing-layer Getter
        dev.linhvu.tracing.transport.Propagator.Getter<Object> transportGetter = context.getGetter();
        Span.Builder extractedSpan = this.propagator.extract(context.getCarrier(),
                (carrier, key) -> transportGetter.get(carrier, key));

        extractedSpan.kind(Span.Kind.valueOf(context.getKind().name()));
        if (context.getRemoteServiceName() != null) {
            extractedSpan.remoteServiceName(context.getRemoteServiceName());
        }
        if (context.getRemoteServiceAddress() != null) {
            try {
                java.net.URI uri = java.net.URI.create(context.getRemoteServiceAddress());
                extractedSpan = extractedSpan.remoteIpAndPort(uri.getHost(), uri.getPort());
            }
            catch (Exception ex) {
                // Silently ignore malformed URIs in the simplified version
            }
        }
        getTracingContext(context).setSpan(extractedSpan.start());
    }

    @Override
    public void onStop(T context) {
        Span span = getRequiredSpan(context);
        tagSpan(context, span);
        String name = context.getContextualName() != null
                ? context.getContextualName() : context.getName();
        if (name != null) {
            span.name(name);
        }
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

The key difference from the sender: `onStart` calls `propagator.extract()` instead of `propagator.inject()`. The `extract()` method returns a `Span.Builder` pre-configured with the extracted parent context — starting it creates a child span that continues the distributed trace.

## 9.4 Try It Yourself

<details>
<summary>Challenge: Implement the sender handler's onStart method</summary>

Given a `SenderContext` with a carrier and setter, write the `onStart` method that:
1. Creates a span with the correct kind
2. Injects the trace context into the carrier
3. Stores the span in the `TracingContext`

```java
@Override
public void onStart(T context) {
    Span childSpan = createSenderSpan(context);
    this.propagator.inject(childSpan.context(), context.getCarrier(),
            (carrier, key, value) -> context.getSetter().set(carrier, key, value));
    getTracingContext(context).setSpan(childSpan);
}
```

The trick is the lambda that bridges between the two `Setter` interfaces — the tracing-layer `Propagator.Setter` and the transport-layer `Propagator.Setter`.

</details>

<details>
<summary>Challenge: Wire the handlers with FirstMatchingCompositeObservationHandler</summary>

Register the three tracing handlers so that `SenderContext` observations use the sender handler, `ReceiverContext` observations use the receiver handler, and everything else uses the default handler:

```java
registry.observationConfig()
    .observationHandler(
        new ObservationHandler.FirstMatchingCompositeObservationHandler(
            new PropagatingSenderTracingObservationHandler<>(tracer, propagator),
            new PropagatingReceiverTracingObservationHandler<>(tracer, propagator),
            new DefaultTracingObservationHandler(tracer)));
```

The order matters: specific handlers must come before the catch-all `DefaultTracingObservationHandler`.

</details>

## 9.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/handler/PropagatingSenderTracingObservationHandlerTest.java`

```java
@Test
void shouldCreateSpanWithClientKind_WhenStarted() {
    Map<String, String> carrier = new HashMap<>();
    SenderContext<Map<String, String>> context = new SenderContext<>(Map::put, Kind.CLIENT);
    context.setCarrier(carrier);
    context.setName("http.client");

    handler.onStart(context);

    SimpleSpan span = tracer.lastSpan();
    assertThat(span.getKind()).isEqualTo(Span.Kind.CLIENT);
}

@Test
void shouldInjectTraceparentHeader_WhenStarted() {
    Map<String, String> carrier = new HashMap<>();
    SenderContext<Map<String, String>> context = new SenderContext<>(Map::put, Kind.CLIENT);
    context.setCarrier(carrier);
    context.setName("http.client");

    handler.onStart(context);

    assertThat(carrier).containsKey("traceparent");
    assertThat(carrier.get("traceparent")).startsWith("00-");
}
```

**New file:** `src/test/java/dev/linhvu/tracing/handler/PropagatingReceiverTracingObservationHandlerTest.java`

```java
@Test
void shouldExtractParentContext_WhenTraceparentPresent() {
    Map<String, String> carrier = new HashMap<>();
    carrier.put("traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01");
    ReceiverContext<Map<String, String>> context = new ReceiverContext<>(Map::get, Kind.SERVER);
    context.setCarrier(carrier);
    context.setName("http.server");

    handler.onStart(context);

    SimpleSpan span = tracer.lastSpan();
    assertThat(span.context().traceId()).isEqualTo("4bf92f3577b34da6a3ce929d0e0e4736");
    assertThat(span.context().parentId()).isEqualTo("00f067aa0ba902b7");
}

@Test
void shouldCreateRootSpan_WhenNoTraceparentPresent() {
    Map<String, String> carrier = new HashMap<>();
    ReceiverContext<Map<String, String>> context = new ReceiverContext<>(Map::get, Kind.SERVER);
    context.setCarrier(carrier);
    context.setName("http.server");

    handler.onStart(context);

    SimpleSpan span = tracer.lastSpan();
    assertThat(span.context().traceId()).isNotEmpty();
    assertThat(span.context().parentId()).isEmpty();
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/PropagatingHandlerIntegrationTest.java`

```java
@Test
void shouldFormDistributedTrace_WhenSenderAndReceiverConnected() {
    // === SERVICE A: sends an outbound HTTP call ===
    Map<String, String> wireHeaders = new HashMap<>();
    SenderContext<Map<String, String>> senderCtx = new SenderContext<>(Map::put, Kind.CLIENT);
    senderCtx.setCarrier(wireHeaders);
    senderCtx.setRemoteServiceName("service-b");

    Observation senderObs = Observation.createNotStarted("http.client", () -> senderCtx, registry)
            .contextualName("GET /api/users")
            .start();
    senderObs.stop();

    SimpleSpan senderSpan = tracer.getSpans().getFirst();

    // === SERVICE B: receives the same request ===
    ReceiverContext<Map<String, String>> receiverCtx = new ReceiverContext<>(Map::get, Kind.SERVER);
    receiverCtx.setCarrier(wireHeaders);

    Observation receiverObs = Observation.createNotStarted("http.server", () -> receiverCtx, registry)
            .contextualName("POST /api/users")
            .start();
    receiverObs.stop();

    SimpleSpan receiverSpan = tracer.getSpans().getLast();

    // Both spans share the same traceId
    assertThat(receiverSpan.context().traceId())
            .isEqualTo(senderSpan.context().traceId());
    // The receiver's parent is the sender's span
    assertThat(receiverSpan.context().parentId())
            .isEqualTo(senderSpan.context().spanId());
    // Correct kinds
    assertThat(senderSpan.getKind()).isEqualTo(Span.Kind.CLIENT);
    assertThat(receiverSpan.getKind()).isEqualTo(Span.Kind.SERVER);
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 9.6 Why This Works

> ★ **Insight** -------------------------------------------
> **The Getter/Setter Bridge Pattern** — The sender and receiver handlers must bridge between two separate `Propagator` interfaces: the observation-layer one (in `transport/Propagator.java`, just `Setter`/`Getter`) and the tracing-layer one (in `propagation/Propagator.java`, with `inject`/`extract` logic). This double-propagator design keeps the Observation API completely decoupled from the Tracing API — libraries like Spring MVC only depend on `micrometer-observation` and provide a `SenderContext` with a `Setter`. They never import any tracing class. The bridge lambda `(carrier, key, value) -> context.getSetter().set(carrier, key, value)` is tiny but architecturally significant.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **FirstMatchingCompositeObservationHandler as a Routing Table** — The `DefaultTracingObservationHandler` inherits a `supportsContext()` that returns `true` for ALL contexts. Without `FirstMatchingCompositeObservationHandler`, it would fire alongside sender/receiver handlers, creating duplicate spans. The composite acts as a routing table: `SenderContext` → sender handler, `ReceiverContext` → receiver handler, anything else → default handler. This is exactly how Spring Boot auto-configures tracing — it wraps all `TracingObservationHandler` implementations in a `FirstMatchingCompositeObservationHandler`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Inject vs. Extract Symmetry** — The sender and receiver handlers are mirror images. The sender creates a span then injects its context (`span → headers`). The receiver extracts a context then creates a span (`headers → span`). Both produce a span with the correct `Kind` and remote service metadata. Together they form the two halves of W3C trace propagation: one service writes `traceparent: 00-{traceId}-{spanId}-01` into HTTP headers, and the next service reads it back — all triggered automatically by the Observation lifecycle.
> -----------------------------------------------------------

## 9.7 What We Enhanced

| Aspect | Before (ch08) | Current (this chapter) | Real Framework |
|--------|---------------|----------------------|----------------|
| **Context type support** | Only local `Observation.Context` (no kind, no propagation) | `SenderContext` triggers injection, `ReceiverContext` triggers extraction | Same pattern — `PropagatingSenderTracingObservationHandler.java:57-62` |
| **Trace propagation** | Traces stayed within one process | Traces cross process boundaries via `traceparent` header injection/extraction | Same — uses `Propagator.inject()` and `Propagator.extract()` |
| **Handler routing** | Single handler for all observations | `FirstMatchingCompositeObservationHandler` routes to the correct handler by context type | Same — Spring Boot wraps all tracing handlers in `FirstMatchingCompositeObservationHandler` |
| **Span kind** | All spans had no kind (local) | Sender creates CLIENT/PRODUCER spans, receiver creates SERVER/CONSUMER spans | Same — kind is mapped from `transport.Kind` via `Span.Kind.valueOf()` |

## 9.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `PropagatingSenderTracingObservationHandler` | `PropagatingSenderTracingObservationHandler` | `PropagatingSenderTracingObservationHandler.java:36` | Real version has `customizeSenderSpan()` extension hook and internal logging |
| `PropagatingReceiverTracingObservationHandler` | `PropagatingReceiverTracingObservationHandler` | `PropagatingReceiverTracingObservationHandler.java:37` | Real version has `customizeExtractedSpan()` (pre-start) and `customizeReceiverSpan()` (post-start) hooks, plus `@Nullable` annotations |
| `supportsContext(instanceof SenderContext)` | Same | `PropagatingSenderTracingObservationHandler.java:121` | Identical pattern |
| Lambda setter bridge | Lambda setter bridge | `PropagatingSenderTracingObservationHandler.java:59-60` | Real version also uses a lambda to bridge |
| Lambda getter bridge | Anonymous inner class | `PropagatingReceiverTracingObservationHandler.java:60-70` | Real version uses an anonymous inner class with both `get()` and `getAll()` methods |
| No URI parsing logging | `InternalLogger.warn()` on malformed URI | `PropagatingSenderTracingObservationHandler.java:84-86` | Real version logs a warning instead of silently ignoring |

## 9.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/handler/PropagatingSenderTracingObservationHandler.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.propagation.Propagator;
import dev.linhvu.tracing.transport.SenderContext;

/**
 * A {@link TracingObservationHandler} for <b>outgoing</b> cross-process communication
 * (e.g., sending HTTP requests or publishing messages).
 *
 * <p>On {@link #onStart}, this handler:
 * <ol>
 *   <li>Creates a new span with the appropriate {@link Span.Kind}
 *       (mapped from the transport {@link dev.linhvu.tracing.transport.Kind})</li>
 *   <li><b>Injects</b> the span's trace context into the outbound carrier
 *       via {@link Propagator#inject} — this is how the downstream service
 *       receives the tracing headers</li>
 *   <li>Stores the span in the {@link TracingContext} for later lifecycle events</li>
 * </ol>
 *
 * <p>Only activates when the observation's context is a {@link SenderContext}.
 *
 * @param <T> a sender context type carrying the outbound carrier
 */
@SuppressWarnings({"rawtypes", "unchecked"})
public class PropagatingSenderTracingObservationHandler<T extends SenderContext>
        implements TracingObservationHandler<T> {

    private final Tracer tracer;

    private final Propagator propagator;

    public PropagatingSenderTracingObservationHandler(Tracer tracer, Propagator propagator) {
        this.tracer = tracer;
        this.propagator = propagator;
    }

    @Override
    public void onStart(T context) {
        Span childSpan = createSenderSpan(context);
        // Inject trace context into the carrier so the receiver can extract it
        this.propagator.inject(childSpan.context(), context.getCarrier(),
                (carrier, key, value) -> context.getSetter().set(carrier, key, value));
        getTracingContext(context).setSpan(childSpan);
    }

    /**
     * Creates the sender span with the appropriate kind, parent, and remote
     * service metadata. Public so subclasses can override the span creation logic.
     */
    public Span createSenderSpan(T context) {
        Span parentSpan = getParentSpan(context);
        Span.Builder builder = getTracer().spanBuilder()
                .kind(Span.Kind.valueOf(context.getKind().name()));
        if (parentSpan != null) {
            builder = builder.setParent(parentSpan.context());
        }
        if (context.getRemoteServiceName() != null) {
            builder = builder.remoteServiceName(context.getRemoteServiceName());
        }
        if (context.getRemoteServiceAddress() != null) {
            try {
                java.net.URI uri = java.net.URI.create(context.getRemoteServiceAddress());
                builder = builder.remoteIpAndPort(uri.getHost(), uri.getPort());
            }
            catch (Exception ex) {
                // Silently ignore malformed URIs in the simplified version
            }
        }
        return builder.start();
    }

    @Override
    public void onStop(T context) {
        Span span = getRequiredSpan(context);
        tagSpan(context, span);
        String name = context.getContextualName() != null
                ? context.getContextualName() : context.getName();
        if (name != null) {
            span.name(name);
        }
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

#### File: `src/main/java/dev/linhvu/tracing/handler/PropagatingReceiverTracingObservationHandler.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.propagation.Propagator;
import dev.linhvu.tracing.transport.ReceiverContext;

/**
 * A {@link TracingObservationHandler} for <b>incoming</b> cross-process communication
 * (e.g., receiving HTTP requests or consuming messages).
 *
 * <p>On {@link #onStart}, this handler:
 * <ol>
 *   <li><b>Extracts</b> trace context from the inbound carrier
 *       via {@link Propagator#extract} — this reads the tracing headers
 *       that the upstream sender injected</li>
 *   <li>Creates a new span as a child of the extracted parent context,
 *       with the appropriate {@link Span.Kind}</li>
 *   <li>Stores the span in the {@link TracingContext} for later lifecycle events</li>
 * </ol>
 *
 * <p>Only activates when the observation's context is a {@link ReceiverContext}.
 *
 * @param <T> a receiver context type carrying the inbound carrier
 */
@SuppressWarnings({"rawtypes", "unchecked"})
public class PropagatingReceiverTracingObservationHandler<T extends ReceiverContext>
        implements TracingObservationHandler<T> {

    private final Tracer tracer;

    private final Propagator propagator;

    public PropagatingReceiverTracingObservationHandler(Tracer tracer, Propagator propagator) {
        this.tracer = tracer;
        this.propagator = propagator;
    }

    @Override
    public void onStart(T context) {
        // Bridge the transport-layer Getter to the tracing-layer Getter
        dev.linhvu.tracing.transport.Propagator.Getter<Object> transportGetter = context.getGetter();
        Span.Builder extractedSpan = this.propagator.extract(context.getCarrier(),
                (carrier, key) -> transportGetter.get(carrier, key));

        extractedSpan.kind(Span.Kind.valueOf(context.getKind().name()));
        if (context.getRemoteServiceName() != null) {
            extractedSpan.remoteServiceName(context.getRemoteServiceName());
        }
        if (context.getRemoteServiceAddress() != null) {
            try {
                java.net.URI uri = java.net.URI.create(context.getRemoteServiceAddress());
                extractedSpan = extractedSpan.remoteIpAndPort(uri.getHost(), uri.getPort());
            }
            catch (Exception ex) {
                // Silently ignore malformed URIs in the simplified version
            }
        }
        getTracingContext(context).setSpan(extractedSpan.start());
    }

    @Override
    public void onStop(T context) {
        Span span = getRequiredSpan(context);
        tagSpan(context, span);
        String name = context.getContextualName() != null
                ? context.getContextualName() : context.getName();
        if (name != null) {
            span.name(name);
        }
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

#### File: `settings.gradle` [MODIFIED]

```groovy
rootProject.name = 'simple-tracing'

includeBuild '../simple-context-propagation'
includeBuild '../simple-micrometer'
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/handler/PropagatingSenderTracingObservationHandlerTest.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.propagation.W3CTraceContextPropagator;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import dev.linhvu.tracing.transport.Kind;
import dev.linhvu.tracing.transport.SenderContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link PropagatingSenderTracingObservationHandler}.
 */
class PropagatingSenderTracingObservationHandlerTest {

    private SimpleTracer tracer;
    private W3CTraceContextPropagator propagator;
    private PropagatingSenderTracingObservationHandler<SenderContext<Map<String, String>>> handler;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        propagator = new W3CTraceContextPropagator(tracer);
        handler = new PropagatingSenderTracingObservationHandler<>(tracer, propagator);
    }

    @Test
    void shouldCreateSpanWithClientKind_WhenStarted() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        SenderContext<Map<String, String>> context = new SenderContext<>(
                Map::put, Kind.CLIENT);
        context.setCarrier(carrier);
        context.setName("http.client");

        // Act
        handler.onStart(context);

        // Assert
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.getKind()).isEqualTo(Span.Kind.CLIENT);
    }

    @Test
    void shouldInjectTraceparentHeader_WhenStarted() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        SenderContext<Map<String, String>> context = new SenderContext<>(
                Map::put, Kind.CLIENT);
        context.setCarrier(carrier);
        context.setName("http.client");

        // Act
        handler.onStart(context);

        // Assert — traceparent header was injected
        assertThat(carrier).containsKey("traceparent");
        String traceparent = carrier.get("traceparent");
        assertThat(traceparent).startsWith("00-");
        assertThat(traceparent.split("-")).hasSize(4);
    }

    @Test
    void shouldCreateChildSpan_WhenParentExists() {
        // Arrange — put a parent span in scope
        SimpleSpan parentSpan = tracer.nextSpan().start();
        try (var ws = tracer.withSpan(parentSpan)) {
            Map<String, String> carrier = new HashMap<>();
            SenderContext<Map<String, String>> context = new SenderContext<>(
                    Map::put, Kind.CLIENT);
            context.setCarrier(carrier);
            context.setName("http.client");

            // Act
            handler.onStart(context);

            // Assert
            SimpleSpan childSpan = tracer.lastSpan();
            assertThat(childSpan.context().traceId())
                    .isEqualTo(parentSpan.context().traceId());
            assertThat(childSpan.context().parentId())
                    .isEqualTo(parentSpan.context().spanId());
        }
    }

    @Test
    void shouldSetRemoteServiceName_WhenProvided() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        SenderContext<Map<String, String>> context = new SenderContext<>(
                Map::put, Kind.CLIENT);
        context.setCarrier(carrier);
        context.setName("http.client");
        context.setRemoteServiceName("user-service");

        // Act
        handler.onStart(context);

        // Assert
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.getRemoteServiceName()).isEqualTo("user-service");
    }

    @Test
    void shouldNameAndEndSpan_WhenStopped() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        SenderContext<Map<String, String>> context = new SenderContext<>(
                Map::put, Kind.CLIENT);
        context.setCarrier(carrier);
        context.setName("http.client");
        context.setContextualName("GET /api/users");
        handler.onStart(context);

        // Act
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("GET /api/users");
        assertThat(span.getEndTimestamp().toEpochMilli()).isGreaterThan(0);
    }

    @Test
    void shouldSupportSenderContext() {
        SenderContext<Map<String, String>> senderCtx = new SenderContext<>(
                Map::put, Kind.CLIENT);
        assertThat(handler.supportsContext(senderCtx)).isTrue();
    }

    @Test
    void shouldNotSupportNonSenderContext() {
        assertThat(handler.supportsContext(new dev.linhvu.micrometer.observation.Observation.Context()))
                .isFalse();
    }

    @Test
    void shouldReturnTracer() {
        assertThat(handler.getTracer()).isSameAs(tracer);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/handler/PropagatingReceiverTracingObservationHandlerTest.java` [NEW]

```java
package dev.linhvu.tracing.handler;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.propagation.W3CTraceContextPropagator;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import dev.linhvu.tracing.transport.Kind;
import dev.linhvu.tracing.transport.ReceiverContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link PropagatingReceiverTracingObservationHandler}.
 */
class PropagatingReceiverTracingObservationHandlerTest {

    private SimpleTracer tracer;
    private W3CTraceContextPropagator propagator;
    private PropagatingReceiverTracingObservationHandler<ReceiverContext<Map<String, String>>> handler;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        propagator = new W3CTraceContextPropagator(tracer);
        handler = new PropagatingReceiverTracingObservationHandler<>(tracer, propagator);
    }

    @Test
    void shouldExtractParentContext_WhenTraceparentPresent() {
        // Arrange — simulate incoming request with traceparent header
        Map<String, String> carrier = new HashMap<>();
        carrier.put("traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01");
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(
                Map::get, Kind.SERVER);
        context.setCarrier(carrier);
        context.setName("http.server");

        // Act
        handler.onStart(context);

        // Assert — span should be a child of the extracted parent
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.context().traceId()).isEqualTo("4bf92f3577b34da6a3ce929d0e0e4736");
        assertThat(span.context().parentId()).isEqualTo("00f067aa0ba902b7");
        assertThat(span.context().spanId()).isNotEqualTo("00f067aa0ba902b7");
    }

    @Test
    void shouldCreateRootSpan_WhenNoTraceparentPresent() {
        // Arrange — no traceparent header
        Map<String, String> carrier = new HashMap<>();
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(
                Map::get, Kind.SERVER);
        context.setCarrier(carrier);
        context.setName("http.server");

        // Act
        handler.onStart(context);

        // Assert — should create a new root trace
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.context().traceId()).isNotEmpty();
        assertThat(span.context().parentId()).isEmpty();
    }

    @Test
    void shouldSetServerKind_WhenStarted() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        carrier.put("traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01");
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(
                Map::get, Kind.SERVER);
        context.setCarrier(carrier);
        context.setName("http.server");

        // Act
        handler.onStart(context);

        // Assert
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.getKind()).isEqualTo(Span.Kind.SERVER);
    }

    @Test
    void shouldSetRemoteServiceName_WhenProvided() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(
                Map::get, Kind.SERVER);
        context.setCarrier(carrier);
        context.setName("http.server");
        context.setRemoteServiceName("gateway-service");

        // Act
        handler.onStart(context);

        // Assert
        SimpleSpan span = tracer.lastSpan();
        assertThat(span.getRemoteServiceName()).isEqualTo("gateway-service");
    }

    @Test
    void shouldNameAndEndSpan_WhenStopped() {
        // Arrange
        Map<String, String> carrier = new HashMap<>();
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(
                Map::get, Kind.SERVER);
        context.setCarrier(carrier);
        context.setName("http.server");
        context.setContextualName("POST /api/orders");
        handler.onStart(context);

        // Act
        handler.onStop(context);

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("POST /api/orders");
        assertThat(span.getEndTimestamp().toEpochMilli()).isGreaterThan(0);
    }

    @Test
    void shouldSupportReceiverContext() {
        ReceiverContext<Map<String, String>> receiverCtx = new ReceiverContext<>(
                Map::get, Kind.SERVER);
        assertThat(handler.supportsContext(receiverCtx)).isTrue();
    }

    @Test
    void shouldNotSupportNonReceiverContext() {
        assertThat(handler.supportsContext(new dev.linhvu.micrometer.observation.Observation.Context()))
                .isFalse();
    }

    @Test
    void shouldReturnTracer() {
        assertThat(handler.getTracer()).isSameAs(tracer);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/PropagatingHandlerIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.micrometer.observation.KeyValue;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationHandler;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.handler.DefaultTracingObservationHandler;
import dev.linhvu.tracing.handler.PropagatingReceiverTracingObservationHandler;
import dev.linhvu.tracing.handler.PropagatingSenderTracingObservationHandler;
import dev.linhvu.tracing.propagation.W3CTraceContextPropagator;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import dev.linhvu.tracing.transport.Kind;
import dev.linhvu.tracing.transport.ReceiverContext;
import dev.linhvu.tracing.transport.SenderContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying the full sender/receiver propagation flow
 * through the ObservationRegistry:
 * <ol>
 *   <li>Sender injects trace context into carrier headers</li>
 *   <li>Receiver extracts the same context and creates a child span</li>
 *   <li>Both share the same traceId, forming a distributed trace</li>
 * </ol>
 */
class PropagatingHandlerIntegrationTest {

    private SimpleTracer tracer;
    private ObservationRegistry registry;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        W3CTraceContextPropagator propagator = new W3CTraceContextPropagator(tracer);
        registry = ObservationRegistry.create();
        // Wrap all tracing handlers in FirstMatchingCompositeObservationHandler.
        // This ensures only ONE handler fires per observation:
        //   - SenderContext → PropagatingSenderTracingObservationHandler
        //   - ReceiverContext → PropagatingReceiverTracingObservationHandler
        //   - anything else → DefaultTracingObservationHandler
        registry.observationConfig()
                .observationHandler(
                        new ObservationHandler.FirstMatchingCompositeObservationHandler(
                                new PropagatingSenderTracingObservationHandler<>(tracer, propagator),
                                new PropagatingReceiverTracingObservationHandler<>(tracer, propagator),
                                new DefaultTracingObservationHandler(tracer)));
    }

    @Test
    void shouldInjectTraceparent_WhenSenderObservationRuns() {
        // Arrange
        Map<String, String> httpHeaders = new HashMap<>();
        SenderContext<Map<String, String>> context = new SenderContext<>(Map::put, Kind.CLIENT);
        context.setCarrier(httpHeaders);

        // Act
        Observation.createNotStarted("http.client", () -> context, registry)
                .lowCardinalityKeyValue("http.method", "GET")
                .start()
                .stop();

        // Assert
        assertThat(httpHeaders).containsKey("traceparent");
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getKind()).isEqualTo(Span.Kind.CLIENT);
        assertThat(span.getTags()).containsEntry("http.method", "GET");
    }

    @Test
    void shouldExtractAndContinueTrace_WhenReceiverObservationRuns() {
        // Arrange — simulate an incoming request with a traceparent header
        Map<String, String> incomingHeaders = new HashMap<>();
        incomingHeaders.put("traceparent",
                "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01");
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(Map::get, Kind.SERVER);
        context.setCarrier(incomingHeaders);

        // Act
        Observation.createNotStarted("http.server", () -> context, registry)
                .start()
                .stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.context().traceId()).isEqualTo("4bf92f3577b34da6a3ce929d0e0e4736");
        assertThat(span.context().parentId()).isEqualTo("00f067aa0ba902b7");
        assertThat(span.getKind()).isEqualTo(Span.Kind.SERVER);
    }

    @Test
    void shouldFormDistributedTrace_WhenSenderAndReceiverConnected() {
        // === SERVICE A: sends an outbound HTTP call ===
        Map<String, String> wireHeaders = new HashMap<>();
        SenderContext<Map<String, String>> senderCtx = new SenderContext<>(Map::put, Kind.CLIENT);
        senderCtx.setCarrier(wireHeaders);
        senderCtx.setRemoteServiceName("service-b");

        Observation senderObs = Observation.createNotStarted("http.client", () -> senderCtx, registry)
                .contextualName("GET /api/users")
                .start();
        senderObs.stop();

        SimpleSpan senderSpan = tracer.getSpans().getFirst();

        // === SERVICE B: receives the same request ===
        // In reality this runs in a different JVM, but we simulate with the same tracer
        ReceiverContext<Map<String, String>> receiverCtx = new ReceiverContext<>(Map::get, Kind.SERVER);
        receiverCtx.setCarrier(wireHeaders);

        Observation receiverObs = Observation.createNotStarted("http.server", () -> receiverCtx, registry)
                .contextualName("POST /api/users")
                .start();
        receiverObs.stop();

        SimpleSpan receiverSpan = tracer.getSpans().getLast();

        // Assert — both spans share the same traceId
        assertThat(receiverSpan.context().traceId())
                .isEqualTo(senderSpan.context().traceId());

        // The receiver's parent is the sender's span
        assertThat(receiverSpan.context().parentId())
                .isEqualTo(senderSpan.context().spanId());

        // Different span IDs
        assertThat(receiverSpan.context().spanId())
                .isNotEqualTo(senderSpan.context().spanId());

        // Correct kinds
        assertThat(senderSpan.getKind()).isEqualTo(Span.Kind.CLIENT);
        assertThat(receiverSpan.getKind()).isEqualTo(Span.Kind.SERVER);

        // Sender recorded remote service name
        assertThat(senderSpan.getRemoteServiceName()).isEqualTo("service-b");
    }

    @Test
    void shouldUseDefaultHandler_WhenPlainContextUsed() {
        // Act — plain context (no SenderContext/ReceiverContext)
        Observation.createNotStarted("local.op", registry)
                .start()
                .stop();

        // Assert — default handler creates a local span (no kind)
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getKind()).isNull();
    }

    @Test
    void shouldRecordError_WhenSenderObservationFails() {
        // Arrange
        Map<String, String> headers = new HashMap<>();
        SenderContext<Map<String, String>> context = new SenderContext<>(Map::put, Kind.CLIENT);
        context.setCarrier(headers);

        // Act
        Observation obs = Observation.createNotStarted("http.client", () -> context, registry).start();
        obs.error(new RuntimeException("connection refused"));
        obs.stop();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getError()).hasMessage("connection refused");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **PropagatingSenderTracingObservationHandler** | Observation handler that creates a span and **injects** its trace context into an outbound carrier (e.g., HTTP headers) |
| **PropagatingReceiverTracingObservationHandler** | Observation handler that **extracts** trace context from an inbound carrier and creates a child span |
| **Getter/Setter bridge** | Lambda that adapts the transport-layer `Propagator.Setter`/`Getter` to the tracing-layer `Propagator.Setter`/`Getter` |
| **FirstMatchingCompositeObservationHandler** | Routes observations to exactly one handler — prevents duplicate span creation when multiple handlers match |
| **`supportsContext(instanceof SenderContext)`** | Type-based routing that ensures only the correct handler processes each observation |

**Next: Chapter 10 — Annotation-Based Tracing** — Add `@NewSpan` and `@ContinueSpan` annotations with a `SpanAspect` proxy, enabling declarative tracing without manual `Tracer` API calls.
