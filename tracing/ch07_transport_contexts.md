# Chapter 7: Transport Contexts

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have spans, a tracer, propagation, and baggage — but everything happens within a single process | No way to tell the Observation API "this observation represents a cross-process call that needs trace context injected/extracted" | Build observation context types that carry the carrier object and propagation strategy, bridging the observation world to the tracing world |

---

## 7.1 The Integration Point

This feature is a **foundation for the Observation-Tracing bridge**. There is no existing file to modify — instead, we're building the data types that Features 8 and 9 will consume. The integration point is the act of extending `Observation.Context`:

```java
public class SenderContext<C> extends Observation.Context {
    private final Propagator.Setter<C> setter;
    private final Kind kind;
    private C carrier;
    // ...
}
```

By extending `Observation.Context`, these types plug into the observation lifecycle. Any `ObservationHandler` can check `context instanceof SenderContext` to know it's dealing with an outbound transport operation, or `context instanceof ReceiverContext` for inbound.

Two key decisions here:

1. **Why extend `Observation.Context` rather than create standalone types?** Because the observation handlers (Feature 8-9) need to receive these via the standard `onStart(T context)` callback. The handler's `supportsContext()` method uses `instanceof` to activate only for transport observations.
2. **Why carry a `Setter`/`Getter` strategy inside the context?** The context must be self-describing — whoever creates the observation knows HOW to write to their carrier (HTTP headers? Kafka record headers?), and they encode that knowledge as a `Setter` or `Getter` lambda. The handler doesn't need to know the carrier type.

This connects **Observation lifecycle** ↔ **Trace context propagation**. To make it work, we need to build:
- `Kind` — enum classifying the communication role (CLIENT/SERVER/PRODUCER/CONSUMER)
- `Propagator` — observation-level interface with `Setter<C>` and `Getter<C>` (distinct from the tracing-level `Propagator`)
- `SenderContext<C>` — for outgoing communication (inject)
- `ReceiverContext<C>` — for incoming communication (extract)

## 7.2 Kind Enum

**New file:** `src/main/java/dev/linhvu/tracing/transport/Kind.java`

```java
package dev.linhvu.tracing.transport;

public enum Kind {

    /** Server-side handling of an RPC or other remote request. */
    SERVER,

    /** Client-side wrapper around an RPC or other remote request. */
    CLIENT,

    /** Producer sending a message to a broker. */
    PRODUCER,

    /** Consumer receiving a message from a broker. */
    CONSUMER

}
```

Two communication paradigms in four values: CLIENT/SERVER for synchronous request/response, PRODUCER/CONSUMER for asynchronous messaging. The key difference: CLIENT/SERVER have a direct latency relationship (the client waits for the server), while PRODUCER/CONSUMER do not (fire-and-forget).

## 7.3 Transport-Level Propagator

**New file:** `src/main/java/dev/linhvu/tracing/transport/Propagator.java`

```java
package dev.linhvu.tracing.transport;

public interface Propagator {

    interface Setter<C> {
        void set(C carrier, String key, String value);
    }

    interface Getter<C> {
        String get(C carrier, String key);
    }

}
```

This is the **observation-level** propagator — it only defines how to access carrier fields. Compare with the **tracing-level** `dev.linhvu.tracing.propagation.Propagator` from Feature 5, which has `inject()`, `extract()`, and `fields()` methods. The two layers are connected by Feature 9's propagating handlers.

Both `Setter` and `Getter` are functional interfaces (single abstract method), enabling method references like `Map::put` and `Map::get`.

## 7.4 SenderContext

**New file:** `src/main/java/dev/linhvu/tracing/transport/SenderContext.java`

```java
package dev.linhvu.tracing.transport;

import dev.linhvu.micrometer.observation.Observation;
import java.util.Objects;

public class SenderContext<C> extends Observation.Context {

    private final Propagator.Setter<C> setter;
    private final Kind kind;
    private C carrier;
    private String remoteServiceName;
    private String remoteServiceAddress;

    public SenderContext(Propagator.Setter<C> setter, Kind kind) {
        this.setter = Objects.requireNonNull(setter, "Setter must be set");
        this.kind = Objects.requireNonNull(kind, "Kind must be set");
    }

    public SenderContext(Propagator.Setter<C> setter) {
        this(setter, Kind.PRODUCER);
    }

    public C getCarrier() { return this.carrier; }
    public void setCarrier(C carrier) { this.carrier = carrier; }
    public Propagator.Setter<C> getSetter() { return this.setter; }
    public Kind getKind() { return this.kind; }
    public String getRemoteServiceName() { return this.remoteServiceName; }
    public void setRemoteServiceName(String remoteServiceName) { this.remoteServiceName = remoteServiceName; }
    public String getRemoteServiceAddress() { return this.remoteServiceAddress; }
    public void setRemoteServiceAddress(String remoteServiceAddress) { this.remoteServiceAddress = remoteServiceAddress; }
}
```

The single-arg constructor defaults to `Kind.PRODUCER` — messaging is the more common fire-and-forget pattern. For HTTP client calls, use the two-arg constructor with `Kind.CLIENT`.

The carrier is **not** set at construction time. It's set later via `setCarrier()` because in many frameworks the carrier object (e.g., the HTTP request) is created after the observation context.

## 7.5 ReceiverContext

**New file:** `src/main/java/dev/linhvu/tracing/transport/ReceiverContext.java`

```java
package dev.linhvu.tracing.transport;

import dev.linhvu.micrometer.observation.Observation;
import java.util.Objects;

public class ReceiverContext<C> extends Observation.Context {

    private final Propagator.Getter<C> getter;
    private final Kind kind;
    private C carrier;
    private String remoteServiceName;
    private String remoteServiceAddress;

    public ReceiverContext(Propagator.Getter<C> getter, Kind kind) {
        this.getter = Objects.requireNonNull(getter, "Getter must be set");
        this.kind = Objects.requireNonNull(kind, "Kind must be set");
    }

    public ReceiverContext(Propagator.Getter<C> getter) {
        this(getter, Kind.CONSUMER);
    }

    public C getCarrier() { return this.carrier; }
    public void setCarrier(C carrier) { this.carrier = carrier; }
    public Propagator.Getter<C> getGetter() { return this.getter; }
    public Kind getKind() { return this.kind; }
    public String getRemoteServiceName() { return this.remoteServiceName; }
    public void setRemoteServiceName(String remoteServiceName) { this.remoteServiceName = remoteServiceName; }
    public String getRemoteServiceAddress() { return this.remoteServiceAddress; }
    public void setRemoteServiceAddress(String remoteServiceAddress) { this.remoteServiceAddress = remoteServiceAddress; }
}
```

The mirror of `SenderContext` — defaults to `Kind.CONSUMER`, holds a `Getter` instead of a `Setter`.

## 7.6 Try It Yourself

<details>
<summary>Challenge: Create a SenderContext for an HTTP client call that injects headers into a Map</summary>

Think about: What `Kind` should you use for a synchronous HTTP client call? How would you express `Map::put` as a `Propagator.Setter`?

```java
// Create the setter — Map::put matches the Setter<Map<String, String>> signature
Propagator.Setter<Map<String, String>> setter = Map::put;

// Use Kind.CLIENT for synchronous request/response
SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

// The carrier is the HTTP headers map
Map<String, String> headers = new HashMap<>();
context.setCarrier(headers);
context.setRemoteServiceName("user-service");

// Now a propagating handler can use context.getSetter()
// to inject trace context into context.getCarrier()
```

</details>

<details>
<summary>Challenge: Create a ReceiverContext for a Kafka consumer that reads from record headers</summary>

Think about: Kafka `ConsumerRecord` headers are accessed differently than HTTP headers. How do you express that as a `Propagator.Getter`?

```java
// Simulate Kafka record headers as a Map
Map<String, String> kafkaHeaders = Map.of(
    "traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
);

// The getter reads from the headers map
Propagator.Getter<Map<String, String>> getter = Map::get;

// Use Kind.CONSUMER (or just the single-arg constructor which defaults to CONSUMER)
ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter);

context.setCarrier(kafkaHeaders);
context.setRemoteServiceName("order-service");

// Now a propagating handler can use context.getGetter()
// to extract trace context from context.getCarrier()
String traceparent = context.getGetter().get(context.getCarrier(), "traceparent");
// → "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
```

</details>

## 7.7 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/transport/KindTest.java`

```java
class KindTest {

    @Test
    void shouldHaveFourValues() {
        assertThat(Kind.values()).hasSize(4);
    }

    @Test
    void shouldContainSynchronousKinds() {
        assertThat(Kind.valueOf("CLIENT")).isEqualTo(Kind.CLIENT);
        assertThat(Kind.valueOf("SERVER")).isEqualTo(Kind.SERVER);
    }

    @Test
    void shouldContainAsynchronousKinds() {
        assertThat(Kind.valueOf("PRODUCER")).isEqualTo(Kind.PRODUCER);
        assertThat(Kind.valueOf("CONSUMER")).isEqualTo(Kind.CONSUMER);
    }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/transport/TransportPropagatorTest.java`

```java
class TransportPropagatorTest {

    @Test
    void shouldSetValueOnCarrier_WhenSetterInvoked() { ... }

    @Test
    void shouldGetValueFromCarrier_WhenGetterInvoked() { ... }

    @Test
    void shouldReturnNull_WhenGetterKeyNotPresent() { ... }

    @Test
    void shouldBeUsableAsLambda_WhenSetterIsLambda() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/transport/SenderContextTest.java`

```java
class SenderContextTest {

    @Test
    void shouldExtendsObservationContext() { ... }

    @Test
    void shouldStoreSetterAndKind_WhenConstructedWithBothArgs() { ... }

    @Test
    void shouldDefaultToProducer_WhenConstructedWithSetterOnly() { ... }

    @Test
    void shouldThrowNpe_WhenSetterIsNull() { ... }

    @Test
    void shouldThrowNpe_WhenKindIsNull() { ... }

    @Test
    void shouldReturnNullCarrier_WhenNotSet() { ... }

    @Test
    void shouldStoreAndReturnCarrier_WhenSet() { ... }

    @Test
    void shouldStoreRemoteServiceName_WhenSet() { ... }

    @Test
    void shouldStoreRemoteServiceAddress_WhenSet() { ... }

    @Test
    void shouldInheritObservationContextCapabilities() { ... }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/transport/ReceiverContextTest.java`

```java
class ReceiverContextTest {

    @Test
    void shouldExtendObservationContext() { ... }

    @Test
    void shouldStoreGetterAndKind_WhenConstructedWithBothArgs() { ... }

    @Test
    void shouldDefaultToConsumer_WhenConstructedWithGetterOnly() { ... }

    @Test
    void shouldThrowNpe_WhenGetterIsNull() { ... }

    @Test
    void shouldThrowNpe_WhenKindIsNull() { ... }

    @Test
    void shouldReturnNullCarrier_WhenNotSet() { ... }

    @Test
    void shouldStoreAndReturnCarrier_WhenSet() { ... }

    @Test
    void shouldStoreRemoteServiceName_WhenSet() { ... }

    @Test
    void shouldStoreRemoteServiceAddress_WhenSet() { ... }

    @Test
    void shouldInheritObservationContextCapabilities() { ... }
}
```

**Run:** `./gradlew test` — expected: all 254 tests pass (including prior features' tests)

---

## 7.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why two Propagator interfaces?** The observation module defines `transport.Propagator.Setter/Getter` — these are carrier access strategies that know nothing about tracing. The tracing module defines `propagation.Propagator` with `inject()`/`extract()` — these know about `TraceContext` and `Span.Builder`. Feature 9's propagating handlers bridge the gap: they take the transport-level setter/getter FROM the context and pass it TO the tracing-level propagator. This separation means the observation module has zero dependency on tracing — it just says "here's how to read/write my carrier," and the tracing layer decides what to read/write.
> - **When this separation is overkill:** If you're building a system where tracing is the ONLY consumer of propagation (no metrics, no logging), having two propagator interfaces adds indirection without benefit. The two-layer design pays off when you have multiple observation handlers that each use the carrier differently.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why is the carrier mutable and set separately from construction?** In many frameworks, the carrier (e.g., the HTTP request object) doesn't exist when the observation starts. Consider Spring's `RestTemplate`: the observation is created BEFORE the request is built. The context is constructed with a setter/kind, the observation starts (handlers fire `onStart`), and THEN the carrier is set before `onStop`. This two-phase initialization is a deliberate design choice, not an oversight.
> - **Trade-off:** A nullable carrier means handlers must null-check it. In practice, the propagating handlers (Feature 9) access the carrier in `onStop()` when it's guaranteed to be set.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why default to PRODUCER/CONSUMER rather than CLIENT/SERVER?** The single-arg constructors default to the messaging paradigm because messaging is the simpler, more common case for fire-and-forget communication. The CLIENT/SERVER pattern implies a request/response exchange with latency correlation, which requires the more explicit two-arg constructor. This makes the common case easy and the specific case possible.
> -----------------------------------------------------------

## 7.9 What We Enhanced

| Aspect | Before (ch05) | Current (ch07) | Real Framework |
|--------|---------------|----------------|----------------|
| Cross-process context | `Propagator` can inject/extract `TraceContext` into carriers, but has no integration with the Observation API | `SenderContext` and `ReceiverContext` carry the carrier + propagation strategy inside `Observation.Context`, enabling observation handlers to automatically propagate | `SenderContext.java:30` / `ReceiverContext.java:31` — identical pattern, plus `@Nullable` annotations and `RequestReply*Context` variants |
| Communication kind | `Span.Kind` enum exists but is only used when manually creating spans | `transport.Kind` enum provides the same classification for observation-driven transport contexts, bridging observation-level kind to span-level kind | `Kind.java:24` — same four values, same semantics |

## 7.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `transport.Kind` | `io.micrometer.observation.transport.Kind` | `Kind.java:24` | Identical — same four enum constants |
| `transport.Propagator.Setter<C>` | `io.micrometer.observation.transport.Propagator.Setter<C>` | `Propagator.java:47` | Real adds `@Nullable` on carrier parameter |
| `transport.Propagator.Getter<C>` | `io.micrometer.observation.transport.Propagator.Getter<C>` | `Propagator.java:76` | Real adds `getAll()` default method for multi-valued keys (`Propagator.java:106`) |
| `SenderContext<C>` | `io.micrometer.observation.transport.SenderContext<C>` | `SenderContext.java:30` | Real adds `@Nullable` annotations throughout |
| `ReceiverContext<C>` | `io.micrometer.observation.transport.ReceiverContext<C>` | `ReceiverContext.java:31` | Real adds `@Nullable` annotations throughout |
| Not implemented | `RequestReplySenderContext` / `RequestReplyReceiverContext` | N/A | Request-reply variants for bidirectional messaging (out of scope) |

## 7.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/transport/Kind.java` [NEW]

```java
package dev.linhvu.tracing.transport;

/**
 * Classifies the role a service plays in a cross-process communication.
 *
 * <p>Two distinct communication paradigms:
 * <ul>
 *   <li>{@link #CLIENT}/{@link #SERVER} — synchronous request/response (RPC, HTTP)</li>
 *   <li>{@link #PRODUCER}/{@link #CONSUMER} — asynchronous messaging (no direct latency relationship)</li>
 * </ul>
 *
 * <p>Used by {@link SenderContext} and {@link ReceiverContext} to indicate
 * what kind of span the tracing handler should create.
 */
public enum Kind {

    /** Server-side handling of an RPC or other remote request. */
    SERVER,

    /** Client-side wrapper around an RPC or other remote request. */
    CLIENT,

    /** Producer sending a message to a broker. */
    PRODUCER,

    /** Consumer receiving a message from a broker. */
    CONSUMER

}
```

#### File: `src/main/java/dev/linhvu/tracing/transport/Propagator.java` [NEW]

```java
package dev.linhvu.tracing.transport;

/**
 * Observation-level propagation abstractions — defines how to read/write
 * propagation fields on a carrier object.
 *
 * <p>This is the <b>transport-layer</b> propagator from the observation module.
 * It only defines the carrier access strategy ({@link Setter}/{@link Getter}),
 * NOT the inject/extract logic. The tracing-layer
 * {@link dev.linhvu.tracing.propagation.Propagator} is the one that
 * actually performs injection and extraction using these accessors.
 *
 * <p>{@link SenderContext} stores a {@link Setter} and
 * {@link ReceiverContext} stores a {@link Getter} — the propagating
 * observation handlers (Feature 9) use them to bridge between the two layers.
 */
public interface Propagator {

    /**
     * Writes a propagation field into a carrier.
     *
     * <p>Implementations should be stateless and can be saved as constants
     * to avoid runtime allocations. For example, a setter for HTTP headers
     * would be {@code HttpURLConnection::addRequestProperty}.
     *
     * @param <C> carrier type (e.g., {@code HttpURLConnection}, {@code Map<String, String>})
     */
    interface Setter<C> {

        /**
         * Sets a propagation field on the carrier.
         *
         * @param carrier the carrier to write to (may be null to facilitate lambda usage)
         * @param key     the header/field name
         * @param value   the header/field value
         */
        void set(C carrier, String key, String value);
    }

    /**
     * Reads a propagation field from a carrier.
     *
     * <p>Implementations should be stateless and can be saved as constants
     * to avoid runtime allocations.
     *
     * @param <C> carrier type
     */
    interface Getter<C> {

        /**
         * Returns the first value of the given propagation key, or null if absent.
         *
         * @param carrier the carrier to read from
         * @param key     the header/field name
         * @return the value, or null
         */
        String get(C carrier, String key);
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/transport/SenderContext.java` [NEW]

```java
package dev.linhvu.tracing.transport;

import dev.linhvu.micrometer.observation.Observation;

import java.util.Objects;

/**
 * An {@link Observation.Context} for outgoing (sender-side) cross-process communication.
 *
 * <p>Carries the carrier object, a {@link Propagator.Setter} for writing
 * propagation fields into the carrier, the communication {@link Kind},
 * and optional remote service metadata.
 *
 * <p>When a propagating observation handler receives this context, it knows
 * to <b>inject</b> trace context into the carrier before the message is sent.
 *
 * @param <C> the type of the carrier (e.g., HTTP request, message headers)
 */
public class SenderContext<C> extends Observation.Context {

    private final Propagator.Setter<C> setter;

    private final Kind kind;

    private C carrier;

    private String remoteServiceName;

    private String remoteServiceAddress;

    /**
     * Creates a sender context with the given setter and kind.
     *
     * @param setter the strategy for writing propagation fields into the carrier
     * @param kind   the communication kind (CLIENT or PRODUCER)
     */
    public SenderContext(Propagator.Setter<C> setter, Kind kind) {
        this.setter = Objects.requireNonNull(setter, "Setter must be set");
        this.kind = Objects.requireNonNull(kind, "Kind must be set");
    }

    /**
     * Creates a sender context with the given setter and default kind {@link Kind#PRODUCER}.
     *
     * @param setter the strategy for writing propagation fields into the carrier
     */
    public SenderContext(Propagator.Setter<C> setter) {
        this(setter, Kind.PRODUCER);
    }

    /** Returns the carrier, or null if not yet set. */
    public C getCarrier() {
        return this.carrier;
    }

    /** Sets the carrier object that propagation fields will be written into. */
    public void setCarrier(C carrier) {
        this.carrier = carrier;
    }

    /** Returns the setter strategy for writing propagation fields. */
    public Propagator.Setter<C> getSetter() {
        return this.setter;
    }

    /** Returns the communication kind. */
    public Kind getKind() {
        return this.kind;
    }

    /** Returns the remote service name, or null if not set. */
    public String getRemoteServiceName() {
        return this.remoteServiceName;
    }

    /** Sets the name of the remote service being called. */
    public void setRemoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
    }

    /** Returns the remote service address, or null if not set. */
    public String getRemoteServiceAddress() {
        return this.remoteServiceAddress;
    }

    /** Sets the address of the remote service being called. */
    public void setRemoteServiceAddress(String remoteServiceAddress) {
        this.remoteServiceAddress = remoteServiceAddress;
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/transport/ReceiverContext.java` [NEW]

```java
package dev.linhvu.tracing.transport;

import dev.linhvu.micrometer.observation.Observation;

import java.util.Objects;

/**
 * An {@link Observation.Context} for incoming (receiver-side) cross-process communication.
 *
 * <p>Carries the carrier object, a {@link Propagator.Getter} for reading
 * propagation fields from the carrier, the communication {@link Kind},
 * and optional remote service metadata.
 *
 * <p>When a propagating observation handler receives this context, it knows
 * to <b>extract</b> trace context from the carrier to continue a distributed trace.
 *
 * @param <C> the type of the carrier (e.g., HTTP request, message headers)
 */
public class ReceiverContext<C> extends Observation.Context {

    private final Propagator.Getter<C> getter;

    private final Kind kind;

    private C carrier;

    private String remoteServiceName;

    private String remoteServiceAddress;

    /**
     * Creates a receiver context with the given getter and kind.
     *
     * @param getter the strategy for reading propagation fields from the carrier
     * @param kind   the communication kind (SERVER or CONSUMER)
     */
    public ReceiverContext(Propagator.Getter<C> getter, Kind kind) {
        this.getter = Objects.requireNonNull(getter, "Getter must be set");
        this.kind = Objects.requireNonNull(kind, "Kind must be set");
    }

    /**
     * Creates a receiver context with the given getter and default kind {@link Kind#CONSUMER}.
     *
     * @param getter the strategy for reading propagation fields from the carrier
     */
    public ReceiverContext(Propagator.Getter<C> getter) {
        this(getter, Kind.CONSUMER);
    }

    /** Returns the carrier, or null if not yet set. */
    public C getCarrier() {
        return this.carrier;
    }

    /** Sets the carrier object that propagation fields will be read from. */
    public void setCarrier(C carrier) {
        this.carrier = carrier;
    }

    /** Returns the getter strategy for reading propagation fields. */
    public Propagator.Getter<C> getGetter() {
        return this.getter;
    }

    /** Returns the communication kind. */
    public Kind getKind() {
        return this.kind;
    }

    /** Returns the remote service name, or null if not set. */
    public String getRemoteServiceName() {
        return this.remoteServiceName;
    }

    /** Sets the name of the remote service that sent the data. */
    public void setRemoteServiceName(String remoteServiceName) {
        this.remoteServiceName = remoteServiceName;
    }

    /** Returns the remote service address, or null if not set. */
    public String getRemoteServiceAddress() {
        return this.remoteServiceAddress;
    }

    /** Sets the address of the remote service that sent the data. */
    public void setRemoteServiceAddress(String remoteServiceAddress) {
        this.remoteServiceAddress = remoteServiceAddress;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/transport/KindTest.java` [NEW]

```java
package dev.linhvu.tracing.transport;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class KindTest {

    @Test
    void shouldHaveFourValues() {
        assertThat(Kind.values()).hasSize(4);
    }

    @Test
    void shouldContainSynchronousKinds() {
        assertThat(Kind.valueOf("CLIENT")).isEqualTo(Kind.CLIENT);
        assertThat(Kind.valueOf("SERVER")).isEqualTo(Kind.SERVER);
    }

    @Test
    void shouldContainAsynchronousKinds() {
        assertThat(Kind.valueOf("PRODUCER")).isEqualTo(Kind.PRODUCER);
        assertThat(Kind.valueOf("CONSUMER")).isEqualTo(Kind.CONSUMER);
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/transport/TransportPropagatorTest.java` [NEW]

```java
package dev.linhvu.tracing.transport;

import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class TransportPropagatorTest {

    @Test
    void shouldSetValueOnCarrier_WhenSetterInvoked() {
        // Arrange
        Propagator.Setter<Map<String, String>> setter = Map::put;
        Map<String, String> carrier = new HashMap<>();

        // Act
        setter.set(carrier, "traceparent", "00-abc-def-01");

        // Assert
        assertThat(carrier).containsEntry("traceparent", "00-abc-def-01");
    }

    @Test
    void shouldGetValueFromCarrier_WhenGetterInvoked() {
        // Arrange
        Propagator.Getter<Map<String, String>> getter = Map::get;
        Map<String, String> carrier = Map.of("traceparent", "00-abc-def-01");

        // Act
        String value = getter.get(carrier, "traceparent");

        // Assert
        assertThat(value).isEqualTo("00-abc-def-01");
    }

    @Test
    void shouldReturnNull_WhenGetterKeyNotPresent() {
        // Arrange
        Propagator.Getter<Map<String, String>> getter = Map::get;
        Map<String, String> carrier = Map.of();

        // Act
        String value = getter.get(carrier, "traceparent");

        // Assert
        assertThat(value).isNull();
    }

    @Test
    void shouldBeUsableAsLambda_WhenSetterIsLambda() {
        // Arrange — demonstrates that Setter is a functional interface
        StringBuilder sb = new StringBuilder();
        Propagator.Setter<StringBuilder> setter = (c, key, value) -> c.append(key).append("=").append(value);

        // Act
        setter.set(sb, "traceId", "abc123");

        // Assert
        assertThat(sb.toString()).isEqualTo("traceId=abc123");
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/transport/SenderContextTest.java` [NEW]

```java
package dev.linhvu.tracing.transport;

import dev.linhvu.micrometer.observation.Observation;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SenderContextTest {

    private final Propagator.Setter<Map<String, String>> setter = Map::put;

    @Test
    void shouldExtendsObservationContext() {
        // Arrange & Act
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

        // Assert
        assertThat(context).isInstanceOf(Observation.Context.class);
    }

    @Test
    void shouldStoreSetterAndKind_WhenConstructedWithBothArgs() {
        // Arrange & Act
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

        // Assert
        assertThat(context.getSetter()).isSameAs(setter);
        assertThat(context.getKind()).isEqualTo(Kind.CLIENT);
    }

    @Test
    void shouldDefaultToProducer_WhenConstructedWithSetterOnly() {
        // Arrange & Act
        SenderContext<Map<String, String>> context = new SenderContext<>(setter);

        // Assert
        assertThat(context.getKind()).isEqualTo(Kind.PRODUCER);
    }

    @Test
    void shouldThrowNpe_WhenSetterIsNull() {
        assertThatThrownBy(() -> new SenderContext<>(null, Kind.CLIENT))
                .isInstanceOf(NullPointerException.class)
                .hasMessage("Setter must be set");
    }

    @Test
    void shouldThrowNpe_WhenKindIsNull() {
        assertThatThrownBy(() -> new SenderContext<>(setter, null))
                .isInstanceOf(NullPointerException.class)
                .hasMessage("Kind must be set");
    }

    @Test
    void shouldReturnNullCarrier_WhenNotSet() {
        // Arrange
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

        // Assert
        assertThat(context.getCarrier()).isNull();
    }

    @Test
    void shouldStoreAndReturnCarrier_WhenSet() {
        // Arrange
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);
        Map<String, String> carrier = new HashMap<>();

        // Act
        context.setCarrier(carrier);

        // Assert
        assertThat(context.getCarrier()).isSameAs(carrier);
    }

    @Test
    void shouldStoreRemoteServiceName_WhenSet() {
        // Arrange
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

        // Act
        context.setRemoteServiceName("user-service");

        // Assert
        assertThat(context.getRemoteServiceName()).isEqualTo("user-service");
    }

    @Test
    void shouldStoreRemoteServiceAddress_WhenSet() {
        // Arrange
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

        // Act
        context.setRemoteServiceAddress("http://user-service:8080");

        // Assert
        assertThat(context.getRemoteServiceAddress()).isEqualTo("http://user-service:8080");
    }

    @Test
    void shouldInheritObservationContextCapabilities() {
        // Arrange — verifies inherited name/key-value functionality
        SenderContext<Map<String, String>> context = new SenderContext<>(setter, Kind.CLIENT);

        // Act
        context.setName("http.client.requests");

        // Assert
        assertThat(context.getName()).isEqualTo("http.client.requests");
    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/transport/ReceiverContextTest.java` [NEW]

```java
package dev.linhvu.tracing.transport;

import dev.linhvu.micrometer.observation.Observation;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ReceiverContextTest {

    private final Propagator.Getter<Map<String, String>> getter = Map::get;

    @Test
    void shouldExtendObservationContext() {
        // Arrange & Act
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);

        // Assert
        assertThat(context).isInstanceOf(Observation.Context.class);
    }

    @Test
    void shouldStoreGetterAndKind_WhenConstructedWithBothArgs() {
        // Arrange & Act
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);

        // Assert
        assertThat(context.getGetter()).isSameAs(getter);
        assertThat(context.getKind()).isEqualTo(Kind.SERVER);
    }

    @Test
    void shouldDefaultToConsumer_WhenConstructedWithGetterOnly() {
        // Arrange & Act
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter);

        // Assert
        assertThat(context.getKind()).isEqualTo(Kind.CONSUMER);
    }

    @Test
    void shouldThrowNpe_WhenGetterIsNull() {
        assertThatThrownBy(() -> new ReceiverContext<>(null, Kind.SERVER))
                .isInstanceOf(NullPointerException.class)
                .hasMessage("Getter must be set");
    }

    @Test
    void shouldThrowNpe_WhenKindIsNull() {
        assertThatThrownBy(() -> new ReceiverContext<>(getter, null))
                .isInstanceOf(NullPointerException.class)
                .hasMessage("Kind must be set");
    }

    @Test
    void shouldReturnNullCarrier_WhenNotSet() {
        // Arrange
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);

        // Assert
        assertThat(context.getCarrier()).isNull();
    }

    @Test
    void shouldStoreAndReturnCarrier_WhenSet() {
        // Arrange
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);
        Map<String, String> carrier = Map.of("traceparent", "00-abc-def-01");

        // Act
        context.setCarrier(carrier);

        // Assert
        assertThat(context.getCarrier()).isSameAs(carrier);
    }

    @Test
    void shouldStoreRemoteServiceName_WhenSet() {
        // Arrange
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);

        // Act
        context.setRemoteServiceName("order-service");

        // Assert
        assertThat(context.getRemoteServiceName()).isEqualTo("order-service");
    }

    @Test
    void shouldStoreRemoteServiceAddress_WhenSet() {
        // Arrange
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);

        // Act
        context.setRemoteServiceAddress("http://order-service:8080");

        // Assert
        assertThat(context.getRemoteServiceAddress()).isEqualTo("http://order-service:8080");
    }

    @Test
    void shouldInheritObservationContextCapabilities() {
        // Arrange — verifies inherited name/key-value functionality
        ReceiverContext<Map<String, String>> context = new ReceiverContext<>(getter, Kind.SERVER);

        // Act
        context.setName("http.server.requests");

        // Assert
        assertThat(context.getName()).isEqualTo("http.server.requests");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`Kind`** | Enum classifying communication role: CLIENT/SERVER (sync RPC) or PRODUCER/CONSUMER (async messaging) |
| **`transport.Propagator`** | Observation-level interface defining carrier access strategy via `Setter<C>` and `Getter<C>` — distinct from the tracing-level `propagation.Propagator` |
| **`SenderContext<C>`** | `Observation.Context` subclass for outgoing calls — carries the carrier, setter, kind, and remote service metadata |
| **`ReceiverContext<C>`** | `Observation.Context` subclass for incoming calls — carries the carrier, getter, kind, and remote service metadata |
| **Carrier-agnostic generics** | The type parameter `<C>` lets any object serve as a carrier (HTTP request, Kafka headers, gRPC metadata) without the framework knowing the specific type |

**Next: Chapter 8 — TracingObservationHandler (Local Spans)** — The bridge that automatically creates and manages spans from observation lifecycle events, using `Observation.Context` (and its transport subclasses) to drive span creation
