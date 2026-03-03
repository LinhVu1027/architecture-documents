# Chapter 6: The Integration Point — Wiring Observations Into Your Code

## What You'll Learn
- How to instrument `DefaultZaloBotClient.exchangeInternal()` with observations
- How to instrument `ZaloUpdateListenerContainer.ProcessingLoop` with observations
- The `ObservationRegistry.NOOP` default pattern for backward compatibility
- How to expose the registry as a configurable property
- The exact pattern used by Spring Kafka's `KafkaTemplate` and Spring Framework's `RestClient`

---

## 6.1 The Integration Pattern

Every Spring project follows the same integration pattern:

```
1. Add ObservationRegistry field (default: NOOP)
2. Add optional custom ObservationConvention field (default: null)
3. Provide setter methods for both
4. At the operation point:
   a. Create context
   b. Create observation via enum.observation(custom, default, contextSupplier, registry)
   c. observation.start()
   d. try (Scope) { doWork() }
   e. catch { observation.error(ex) }
   f. finally { observation.stop() }
```

## 6.2 Instrumenting the Zalo Bot Client

### Step 1: Add Fields and Setters to `DefaultZaloBotClient`

```java
final class DefaultZaloBotClient implements ZaloBotClient {

    private final ZaloBotUrl url;
    private final String botToken;
    private final ClientHttpRequestFactory clientHttpRequestFactory;
    private final JsonMapper jsonMapper;
    private final DefaultZaloBotClientBuilder builder;

    // NEW: Observation support
    private ObservationRegistry observationRegistry = ObservationRegistry.NOOP;
    private ZaloBotClientObservationConvention observationConvention;

    // ... constructor ...

    public void setObservationRegistry(ObservationRegistry observationRegistry) {
        Assert.notNull(observationRegistry, "'observationRegistry' must not be null");
        this.observationRegistry = observationRegistry;
    }

    public void setObservationConvention(ZaloBotClientObservationConvention convention) {
        this.observationConvention = convention;
    }
```

### Step 2: Instrument `exchangeInternal()`

Here's the **before** and **after**:

**BEFORE** (current code):
```java
private <N> ZaloApiResponse<N> exchangeInternal(HttpMethod method, String methodPath,
        Map<String, String> headers, Object body, Class<N> clazz) {

    URI uri = buildUri(methodPath);
    ClientHttpRequest request = this.clientHttpRequestFactory.createRequest(uri, method);
    // ... write headers and body ...
    try (ClientHttpResponse response = request.execute()) {
        // ... deserialize and handle errors ...
        return apiResponse;
    }
    catch (IOException e) {
        throw new ZaloBotClientException("HTTP request failed", e);
    }
}
```

**AFTER** (with observation):
```java
private <N> ZaloApiResponse<N> exchangeInternal(HttpMethod method, String methodPath,
        Map<String, String> headers, Object body, Class<N> clazz) {

    // 1. Create context
    ZaloBotClientContext observationContext =
            new ZaloBotClientContext(methodPath, method);

    // 2. Create observation from documented enum
    Observation observation = ZaloBotClientObservation.API_REQUEST.observation(
            this.observationConvention,
            ZaloBotClientObservation.DefaultZaloBotClientObservationConvention.INSTANCE,
            () -> observationContext,
            this.observationRegistry);

    // 3. Start observation
    observation.start();

    try (Observation.Scope scope = observation.openScope()) {
        // 4. Build and execute HTTP request
        URI uri = buildUri(methodPath);
        ClientHttpRequest request = this.clientHttpRequestFactory.createRequest(uri, method);

        // Set carrier for trace propagation (enables header injection)
        observationContext.setCarrier(request);

        Map<String, String> requestHeaders = request.getHeaders();
        requestHeaders.putAll(headers);

        if (body != null) {
            try {
                byte[] bodyBytes = this.jsonMapper.writeValueAsBytes(body);
                OutputStream outputStream = request.getBody();
                outputStream.write(bodyBytes);
                outputStream.flush();
            }
            catch (IOException e) {
                throw new ZaloBotSerializationException("Failed to serialize request body", e);
            }
        }

        try (ClientHttpResponse response = request.execute()) {
            // 5. Store response for tag extraction
            observationContext.setResponse(response);

            int httpStatus = response.getStatusCode();
            JavaType javaType = this.jsonMapper.getTypeFactory()
                    .constructParametricType(ZaloApiResponse.class, clazz);
            ZaloApiResponse<N> apiResponse = this.jsonMapper.readValue(
                    response.getBody(), javaType);

            if (apiResponse != null && !apiResponse.ok()) {
                observationContext.setSuccess(false);
                ZaloErrorCode code = ZaloErrorCode.fromCode(apiResponse.errorCode());
                String description = code.getDescription();
                if (code.isAuthenticationError()) {
                    throw new ZaloBotAuthenticationException(httpStatus,
                            apiResponse.errorCode(), description);
                }
                if (code.isRequestTimeout()) {
                    throw new ZaloBotRequestTimeoutException(httpStatus,
                            apiResponse.errorCode(), description);
                }
                throw new ZaloBotApiException(httpStatus, apiResponse.errorCode(), description);
            }
            return apiResponse;
        }
    }
    catch (ZaloBotException e) {
        // 6. Record error
        observationContext.setSuccess(false);
        observation.error(e);
        throw e;
    }
    catch (IOException e) {
        observationContext.setSuccess(false);
        observation.error(e);
        throw new ZaloBotClientException("HTTP request failed: " + e.getMessage(), e);
    }
    finally {
        // 7. Always stop (records duration)
        observation.stop();
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - **`setCarrier(request)` placement matters**: It must be called AFTER the request is created but BEFORE `request.execute()`. The `PropagatingSenderTracingObservationHandler` uses the carrier's setter to inject trace headers. If you set the carrier too late, trace headers won't be propagated. Spring Framework's `DefaultRestClient` sets the carrier at context construction time because it has the request builder earlier.
> - **Why `observationContext.setSuccess(false)` before `observation.error(e)`**: The convention reads `context.isSuccess()` in `onStop()` to determine the `outcome` tag. The `error()` call only stores the exception; it doesn't automatically set success/failure. This is consistent with Spring Framework's approach where `context.setConnectionAborted(true)` is set independently of `observation.error()`.
> ─────────────────────────────────────────────────────

## 6.3 Instrumenting the Listener Container

### Step 1: Add Fields to `ContainerProperties`

```java
public class ContainerProperties {
    // ... existing fields ...

    private ObservationRegistry observationRegistry = ObservationRegistry.NOOP;
    private ZaloBotListenerObservationConvention observationConvention;

    public ObservationRegistry getObservationRegistry() {
        return this.observationRegistry;
    }

    public void setObservationRegistry(ObservationRegistry observationRegistry) {
        Assert.notNull(observationRegistry, "'observationRegistry' must not be null");
        this.observationRegistry = observationRegistry;
    }

    public ZaloBotListenerObservationConvention getObservationConvention() {
        return this.observationConvention;
    }

    public void setObservationConvention(
            ZaloBotListenerObservationConvention observationConvention) {
        this.observationConvention = observationConvention;
    }
}
```

### Step 2: Instrument `ProcessingLoop`

**BEFORE:**
```java
public void run() {
    while (isRunning() || !queue.isEmpty()) {
        try {
            GetUpdatesResult update = queue.poll(1, TimeUnit.SECONDS);
            if (update != null) {
                listener.onUpdate(update);
            }
        }
        catch (Exception e) {
            errorHandler.handleError(e, ZaloUpdateListenerContainer.this);
        }
    }
}
```

**AFTER:**
```java
private final class ProcessingLoop implements Runnable {

    private final BlockingQueue<GetUpdatesResult> queue;
    private final UpdateListener listener;
    private final ErrorHandler errorHandler;
    private final ObservationRegistry observationRegistry;
    private final ZaloBotListenerObservationConvention observationConvention;

    ProcessingLoop(BlockingQueue<GetUpdatesResult> queue,
            UpdateListener listener,
            ErrorHandler errorHandler,
            ObservationRegistry observationRegistry,
            ZaloBotListenerObservationConvention observationConvention) {
        this.queue = queue;
        this.listener = listener;
        this.errorHandler = errorHandler;
        this.observationRegistry = observationRegistry;
        this.observationConvention = observationConvention;
    }

    @Override
    public void run() {
        while (isRunning() || !queue.isEmpty()) {
            try {
                GetUpdatesResult update = queue.poll(1, TimeUnit.SECONDS);
                if (update != null) {
                    processUpdate(update);
                }
            }
            catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
            catch (Exception e) {
                errorHandler.handleError(e, ZaloUpdateListenerContainer.this);
            }
        }
    }

    private void processUpdate(GetUpdatesResult update) {
        Observation observation = ZaloBotListenerObservation.LISTENER_OBSERVATION
            .observation(
                this.observationConvention,
                ZaloBotListenerObservation.DefaultZaloBotListenerObservationConvention.INSTANCE,
                () -> new ZaloBotListenerContext("default", update),
                this.observationRegistry);

        observation.start();
        try (Observation.Scope scope = observation.openScope()) {
            this.listener.onUpdate(update);
        }
        catch (Exception e) {
            observation.error(e);
            this.errorHandler.handleError(e, ZaloUpdateListenerContainer.this);
        }
        finally {
            observation.stop();
        }
    }
}
```

## 6.4 How Spring Kafka Does It (Side-by-Side)

**Spring Kafka's `KafkaTemplate.observeSend()`** (`KafkaTemplate.java:819-837`):

```java
private CompletableFuture<SendResult<K, V>> observeSend(ProducerRecord<K, V> producerRecord) {
    Observation observation = KafkaTemplateObservation.TEMPLATE_OBSERVATION.observation(
        this.observationConvention,
        DefaultKafkaTemplateObservationConvention.INSTANCE,
        () -> new KafkaRecordSenderContext(producerRecord, this.beanName, this::clusterId),
        this.observationRegistry);

    observation.start();
    try {
        try (Observation.Scope ignored = observation.openScope()) {
            return doSend(producerRecord, observation);
        }
    }
    catch (RuntimeException ex) {
        if (observation.getContext().getError() == null) {
            observation.error(ex);
            observation.stop();
        }
        throw ex;
    }
}
```

**Spring Kafka's `KafkaMessageListenerContainer`** (`KafkaMessageListenerContainer.java:2845-2850`):

```java
Observation observation = KafkaListenerObservation.LISTENER_OBSERVATION.observation(
    this.containerProperties.getObservationConvention(),
    DefaultKafkaListenerObservationConvention.INSTANCE,
    () -> new KafkaRecordReceiverContext(cRecord, getListenerId(), getClientId(),
        this.consumerGroupId, this::clusterId),
    this.observationRegistry);

observation.start();
Observation.Scope observationScope = observation.openScope();
try {
    invokeOnMessage(cRecord);
}
catch (RuntimeException e) {
    observation.error(e);
}
finally {
    observation.stop();
    observationScope.close();
}
```

### Pattern Comparison

| Aspect | Spring Kafka (Producer) | Spring Kafka (Listener) | Zalobot (Client) | Zalobot (Listener) |
|--------|------------------------|------------------------|-------------------|---------------------|
| Enum constant | `TEMPLATE_OBSERVATION` | `LISTENER_OBSERVATION` | `API_REQUEST` | `LISTENER_OBSERVATION` |
| Context type | `KafkaRecordSenderContext` | `KafkaRecordReceiverContext` | `ZaloBotClientContext` | `ZaloBotListenerContext` |
| Convention storage | Field on KafkaTemplate | ContainerProperties | Field on client | ContainerProperties |
| Registry storage | Field on KafkaTemplate | ContainerProperties | Field on client | ContainerProperties |
| Scope pattern | try-with-resources | Manual open/close | try-with-resources | try-with-resources |

## 6.5 The Builder Integration

The `ObservationRegistry` should also be settable via the builder:

```java
public class DefaultZaloBotClientBuilder implements ZaloBotClient.Builder {
    // ... existing fields ...

    private ObservationRegistry observationRegistry = ObservationRegistry.NOOP;
    private ZaloBotClientObservationConvention observationConvention;

    @Override
    public Builder observationRegistry(ObservationRegistry registry) {
        Assert.notNull(registry, "'observationRegistry' must not be null");
        this.observationRegistry = registry;
        return this;
    }

    @Override
    public Builder observationConvention(ZaloBotClientObservationConvention convention) {
        this.observationConvention = convention;
        return this;
    }

    @Override
    public ZaloBotClient build() {
        DefaultZaloBotClient client = new DefaultZaloBotClient(
                url, botToken, clientHttpRequestFactory, jsonMapper, this);
        client.setObservationRegistry(this.observationRegistry);
        client.setObservationConvention(this.observationConvention);
        return client;
    }
}
```

## 6.6 Backward Compatibility

Because `ObservationRegistry.NOOP` is the default, **existing users are not affected**:

```java
// Existing code — works exactly as before (NOOP = zero overhead)
ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .build();

// New code — opt-in to observation
ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .observationRegistry(registry)
    .build();
```

> ★ **Insight** ─────────────────────────────────────
> - **NOOP as default is non-negotiable for libraries**: Spring Kafka defaults to `ObservationRegistry.NOOP`. Spring Framework does the same. If a library required an `ObservationRegistry`, it would force a dependency on `micrometer-observation` at runtime even for users who don't want metrics. NOOP means the dependency is optional — present on the classpath when users opt in via Spring Boot Actuator.
> - **Convention field is `@Nullable`**: When null, the `observation()` factory method falls through to the default convention. This is the common case — most users won't provide custom conventions.
> ─────────────────────────────────────────────────────

## 6.7 Connection to Real Source

| Concept | Source |
|---------|--------|
| KafkaTemplate observation fields | `KafkaTemplate.java:154-158` — `observationEnabled`, `observationConvention`, `observationRegistry` |
| KafkaTemplate.observeSend() | `KafkaTemplate.java:819-837` — observation creation and lifecycle |
| KafkaTemplate.setObservationRegistry() | `KafkaTemplate.java:464-467` |
| ContainerProperties observation fields | `ContainerProperties.java:285-309` |
| KafkaMessageListenerContainer observation | `KafkaMessageListenerContainer.java:2845-2850` |
| DefaultRestClient observation | `DefaultRestClient.java:603-606` |
| ServerHttpObservationFilter | `ServerHttpObservationFilter.java` — servlet filter integration |
| DefaultWebClient observation | `DefaultWebClient.java:442-481` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Integration point** | The method where observation lifecycle wraps the actual operation |
| **Fields pattern** | `ObservationRegistry` (default NOOP) + `@Nullable ObservationConvention` |
| **`setCarrier()`** | Must be called before HTTP execution to enable trace header injection |
| **Error before stop** | `observation.error(ex)` records exception; `observation.stop()` in finally records duration |
| **Backward compatible** | Default NOOP means zero overhead for users who don't opt in |
| **Builder integration** | Registry and convention settable via builder for easy configuration |

**Next: Chapter 7** — Spring Boot Auto-Configuration: how to auto-wire the `ObservationRegistry` bean into Zalobot SDK.
