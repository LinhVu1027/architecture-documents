# Chapter 4: The ObservationContext — Domain-Specific Data Carrier

## What You'll Learn
- Why you create custom Context subclasses instead of using `Observation.Context` directly
- The transport context hierarchy: `SenderContext`, `ReceiverContext`, `RequestReplySenderContext`
- How Context serves double duty: data carrier for conventions AND propagation bridge for tracing
- How to design a Context for Zalobot SDK operations

---

## 4.1 The Problem: Conventions Need Data

In Chapter 3, our `DefaultZaloBotClientObservationConvention` called:

```java
context.getMethodPath()   // "sendMessage"
context.isSuccess()       // true/false
context.getExceptionName() // "none" or "ZaloBotApiException"
context.getRequestUrl()   // "https://bot-api.zaloplatforms.com/bot.../sendMessage"
```

But `Observation.Context` doesn't have these methods! It only has generic facilities:

```java
// Observation.Context provides:
context.getName()                    // observation name
context.getError()                   // Throwable
context.getLowCardinalityKeyValues() // tags (set by convention)
context.put(key, value)              // generic map-like storage
```

We need a **typed subclass** that carries domain-specific data.

## 4.2 Choosing the Right Base Class

Micrometer provides a hierarchy of context base classes based on the communication pattern:

```
Observation.Context                    ← Generic (no transport)
├── SenderContext<C>                   ← Outgoing fire-and-forget
│   └── RequestReplySenderContext<Req,Res>  ← Outgoing request-reply
├── ReceiverContext<C>                 ← Incoming fire-and-forget
│   └── RequestReplyReceiverContext<Req,Res> ← Incoming request-reply
```

Decision matrix:

| Your Operation | Direction | Has Response? | Use |
|----------------|-----------|---------------|-----|
| Zalobot client → Zalo API (sendMessage) | Outgoing | Yes (API response) | `RequestReplySenderContext` |
| Zalobot client → Zalo API (getUpdates) | Outgoing | Yes (API response) | `RequestReplySenderContext` |
| Kafka producer → topic | Outgoing | No (fire-and-forget) | `SenderContext` |
| Kafka consumer ← topic | Incoming | No (fire-and-forget) | `ReceiverContext` |
| HTTP server ← client request | Incoming | Yes (HTTP response) | `RequestReplyReceiverContext` |
| Listener processing an update | Local | No (internal) | `Observation.Context` |

For Zalobot SDK, we have two main contexts:

1. **API Client calls** → `RequestReplySenderContext` (outgoing request-reply)
2. **Listener update processing** → `Observation.Context` (local processing, no transport)

## 4.3 Why Transport Contexts Matter for Tracing

The transport contexts (`SenderContext`, `ReceiverContext`) carry a **Propagator** — a function that injects or extracts trace headers:

```java
// SenderContext: INJECT trace headers into outgoing carrier
public class SenderContext<C> extends Observation.Context {
    private final Propagator.Setter<C> setter;  // (carrier, key, value) -> inject

    public SenderContext(Propagator.Setter<C> setter) {
        this.setter = setter;
    }
}

// ReceiverContext: EXTRACT trace headers from incoming carrier
public class ReceiverContext<C> extends Observation.Context {
    private final Propagator.Getter<C> getter;  // (carrier, key) -> extract

    public ReceiverContext(Propagator.Getter<C> getter) {
        this.getter = getter;
    }
}
```

When the `PropagatingSenderTracingObservationHandler` sees a `SenderContext`, it uses the setter to inject trace IDs into the outgoing HTTP headers:

```
┌──────────────────┐         ┌──────────────────┐
│ Zalobot Client   │  HTTP   │   Zalo API       │
│                  │ ──────► │                   │
│ SenderContext    │         │                   │
│  .setter injects │         │                   │
│  traceparent:    │         │                   │
│  00-abc-def-01   │         │                   │
└──────────────────┘         └──────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - **Why Propagator is in the Context, not the Handler**: The handler is generic (works for HTTP, Kafka, gRPC). The context is specific (knows how to set HTTP headers vs Kafka headers vs gRPC metadata). By putting the propagator in the context, the same tracing handler works for all transports.
> - **Spring Kafka example**: `KafkaRecordSenderContext` injects trace headers into Kafka message headers via `record.headers().add(key, value.getBytes())`. `KafkaRecordReceiverContext` extracts them via `carrier.headers().lastHeader(key)`. Same handler, different propagation logic.
> ─────────────────────────────────────────────────────

## 4.4 Designing the Zalobot Client Context

For the Zalobot HTTP client, we use `RequestReplySenderContext` because each API call is an outgoing request with a response:

```java
/**
 * {@link Observation.Context} for Zalo Bot API client HTTP requests.
 *
 * <p>Extends {@link RequestReplySenderContext} to support distributed
 * tracing header propagation into outgoing HTTP requests.
 */
public class ZaloBotClientContext
        extends RequestReplySenderContext<ClientHttpRequest, ClientHttpResponse> {

    private final String methodPath;
    private final HttpMethod httpMethod;
    private String exceptionName = KeyValue.NONE_VALUE;
    private boolean success = true;

    public ZaloBotClientContext(String methodPath, HttpMethod httpMethod) {
        // Propagator.Setter: injects trace headers into ClientHttpRequest headers
        super((request, key, value) -> {
            if (request != null) {
                request.getHeaders().put(key, value);
            }
        });
        this.methodPath = methodPath;
        this.httpMethod = httpMethod;
        setRemoteServiceName("Zalo Bot API");
    }

    public String getMethodPath() {
        return this.methodPath;
    }

    public HttpMethod getHttpMethod() {
        return this.httpMethod;
    }

    public boolean isSuccess() {
        return this.success;
    }

    public void setSuccess(boolean success) {
        this.success = success;
    }

    public String getExceptionName() {
        return this.exceptionName;
    }

    public void setExceptionName(String exceptionName) {
        this.exceptionName = exceptionName;
    }

    public String getRequestUrl() {
        ClientHttpRequest request = getCarrier();
        return request != null ? request.getURI().toString() : "unknown";
    }
}
```

Key design decisions:
1. **`methodPath`** is set at construction (known before the request executes)
2. **`success`/`exceptionName`** are set during execution (known after response)
3. **`setCarrier(request)`** is called at the integration point to enable trace propagation
4. **`setResponse(response)`** is called after HTTP execution for response-based tags

## 4.5 Designing the Listener Processing Context

For update processing in the listener, there's no transport — it's local processing:

```java
/**
 * {@link Observation.Context} for Zalo Bot listener update processing.
 *
 * <p>Extends {@link Observation.Context} directly since listener processing
 * is a local operation without network propagation.
 */
public class ZaloBotListenerContext extends Observation.Context {

    private final String listenerId;
    private final GetUpdatesResult update;

    public ZaloBotListenerContext(String listenerId, GetUpdatesResult update) {
        this.listenerId = listenerId;
        this.update = update;
    }

    public String getListenerId() {
        return this.listenerId;
    }

    public GetUpdatesResult getUpdate() {
        return this.update;
    }

    public String getEventName() {
        return this.update.eventName() != null ? this.update.eventName() : "unknown";
    }
}
```

## 4.6 How Spring Kafka and Spring Framework Do It

### Spring Kafka — KafkaRecordSenderContext

```java
// Source: KafkaRecordSenderContext.java
public class KafkaRecordSenderContext extends SenderContext<ProducerRecord<?, ?>> {
    private final String beanName;
    private final ProducerRecord<?, ?> record;

    public KafkaRecordSenderContext(ProducerRecord<?, ?> record, String beanName,
            Supplier<String> clusterId) {
        // Propagator.Setter: inject trace headers into Kafka headers
        super((carrier, key, value) -> {
            Headers headers = record.headers();
            headers.remove(key);
            headers.add(key, value == null ? null : value.getBytes(StandardCharsets.UTF_8));
        });
        setCarrier(record);
        this.beanName = beanName;
        this.record = record;
        setRemoteServiceName("Apache Kafka" + (cluster != null ? ": " + cluster : ""));
    }
}
```

### Spring Framework — ClientRequestObservationContext (RestClient)

```java
// Source: spring-web/ClientRequestObservationContext.java
public class ClientRequestObservationContext
        extends RequestReplySenderContext<ClientHttpRequest, ClientHttpResponse> {

    private String uriTemplate;

    public ClientRequestObservationContext(ClientHttpRequest request) {
        super(ClientRequestObservationContext::setRequestHeader);
        setCarrier(request);
    }

    private static void setRequestHeader(ClientHttpRequest request, String name, String value) {
        request.getHeaders().set(name, value);
    }
}
```

### Pattern Comparison

| Aspect | Spring Kafka (Producer) | Spring Framework (RestClient) | Zalobot (Client) |
|--------|------------------------|------------------------------|-------------------|
| Base class | `SenderContext<ProducerRecord>` | `RequestReplySenderContext<Req,Res>` | `RequestReplySenderContext<Req,Res>` |
| Propagator target | Kafka message headers | HTTP request headers | HTTP request headers |
| Domain data | `beanName`, `topic` | `uriTemplate` | `methodPath`, `httpMethod` |
| Remote service | `"Apache Kafka: cluster"` | — (not set) | `"Zalo Bot API"` |

## 4.7 Connection to Real Source

| Concept | Source File |
|---------|------------|
| `SenderContext` | `micrometer/micrometer-observation/.../transport/SenderContext.java` |
| `ReceiverContext` | `micrometer/micrometer-observation/.../transport/ReceiverContext.java` |
| `RequestReplySenderContext` | `micrometer/micrometer-observation/.../transport/RequestReplySenderContext.java` |
| `Propagator.Setter` | `micrometer/micrometer-observation/.../transport/Propagator.java` |
| `KafkaRecordSenderContext` | `spring-kafka/.../support/micrometer/KafkaRecordSenderContext.java` |
| `KafkaRecordReceiverContext` | `spring-kafka/.../support/micrometer/KafkaRecordReceiverContext.java` |
| `ClientRequestObservationContext` | `spring-framework/spring-web/.../client/observation/ClientRequestObservationContext.java` |
| `ServerRequestObservationContext` | `spring-framework/spring-web/.../server/observation/ServerRequestObservationContext.java` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Custom Context** | Typed subclass of `Observation.Context` carrying domain-specific data for conventions |
| **Transport Contexts** | `SenderContext`/`ReceiverContext` carry propagation functions for distributed tracing |
| **RequestReplySenderContext** | For outgoing request-reply operations (HTTP client → API) |
| **Propagator.Setter** | Lambda `(carrier, key, value)` that injects trace headers into the transport |
| **setRemoteServiceName** | Tells tracing what the remote endpoint is (appears in dependency graphs) |
| **setCarrier/setResponse** | Store the actual request/response objects for propagation and tag extraction |

**Next: Chapter 5** — The `ObservationDocumentation` enum: self-documenting observation definitions that serve as both documentation and factory.
