# Chapter 8: Putting It All Together — Complete Implementation Map

## What You'll Learn
- The complete list of files to create/modify for Micrometer integration
- Module placement for each artifact
- End-to-end walkthrough: what happens when a user sends a message
- How all 5 artifacts connect at runtime
- Testing strategy for observations

---

## 8.1 Complete File Listing

Here's every file that needs to be created or modified, organized by module:

### New Files to Create

```
zalobot-client/src/main/java/dev/linhvu/zalobot/client/observation/
├── ZaloBotClientContext.java                    ← Context (extends RequestReplySenderContext)
├── ZaloBotClientObservationConvention.java      ← Convention interface
└── ZaloBotClientObservation.java                ← Documentation enum + DefaultConvention inner class

zalobot-listener/src/main/java/dev/linhvu/zalobot/listener/observation/
├── ZaloBotListenerContext.java                  ← Context (extends Observation.Context)
├── ZaloBotListenerObservationConvention.java    ← Convention interface
└── ZaloBotListenerObservation.java              ← Documentation enum + DefaultConvention inner class
```

### Existing Files to Modify

```
zalobot-client/
├── pom.xml                                      ← Add micrometer-observation (optional)
└── src/main/java/dev/linhvu/zalobot/client/
    ├── ZaloBotClient.java                       ← Add observationRegistry() + observationConvention() to Builder
    ├── DefaultZaloBotClient.java                ← Add observation fields + instrument exchangeInternal()
    └── DefaultZaloBotClientBuilder.java         ← Add observation fields + wire into build()

zalobot-listener/
├── pom.xml                                      ← Add micrometer-observation (optional)
└── src/main/java/dev/linhvu/zalobot/listener/
    ├── ContainerProperties.java                 ← Add observationRegistry + observationConvention fields
    └── ZaloUpdateListenerContainer.java         ← Instrument ProcessingLoop with observation

zalobot-spring-boot/
└── src/main/java/dev/linhvu/zalobot/boot/autoconfigure/
    ├── ZaloBotClientAutoConfiguration.java      ← Wire ObservationRegistry via ObjectProvider
    └── ZaloBotListenerAutoConfiguration.java    ← Wire ObservationRegistry via ObjectProvider
```

## 8.2 Module Placement Rationale

```
┌──────────────────────────────────────────────────────────────────┐
│ zalobot-core                                                     │
│   No observation code here.                                      │
│   Core models are observation-agnostic.                          │
├──────────────────────────────────────────────────────────────────┤
│ zalobot-client                                                   │
│   observation/ package:                                          │
│     ZaloBotClientContext.java                                    │
│     ZaloBotClientObservationConvention.java                      │
│     ZaloBotClientObservation.java                                │
│   Why here: These are specific to the HTTP client operations.    │
│   Same pattern as Spring Kafka putting observation classes       │
│   in spring-kafka/support/micrometer/                            │
├──────────────────────────────────────────────────────────────────┤
│ zalobot-listener                                                 │
│   observation/ package:                                          │
│     ZaloBotListenerContext.java                                  │
│     ZaloBotListenerObservationConvention.java                    │
│     ZaloBotListenerObservation.java                              │
│   Why here: These are specific to the listener processing.       │
│   Same pattern as Spring Kafka's KafkaListenerObservation.       │
├──────────────────────────────────────────────────────────────────┤
│ zalobot-spring-boot                                              │
│   Auto-configuration wiring only.                                │
│   No new observation classes — just ObjectProvider injection.     │
│   Why here: Auto-config is Spring Boot's concern, not the SDK's. │
├──────────────────────────────────────────────────────────────────┤
│ zalobot-spring-boot-starter                                      │
│   No changes needed.                                             │
│   Does NOT pull in spring-boot-starter-actuator.                 │
└──────────────────────────────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - **Why observation classes live in the module that performs the operation**: Spring Kafka's `KafkaRecordSenderContext` is in `spring-kafka` (not a separate metrics module) because the context needs access to `ProducerRecord`. Similarly, `ZaloBotClientContext` needs access to `ClientHttpRequest` — so it belongs in `zalobot-client`.
> - **Anti-pattern: Centralized observation module**: You might think about creating a `zalobot-observation` module. Don't. It would create circular dependencies (observation needs client types, client needs observation types). The per-module approach mirrors Spring's architecture.
> ─────────────────────────────────────────────────────

## 8.3 End-to-End Walkthrough: Sending a Message

Let's trace what happens when a Spring Boot application sends a Zalo message with full observability enabled:

```
1. Application calls:
   client.sendMessage()
     .body(new SendMessage(chatId, "Hello"))
     .retrieve()
     .call(SendMessageResult.class)
       │
       ▼
2. DefaultZaloBotClient.exchangeInternal("sendMessage", POST, ...)
       │
       ▼
3. Create ZaloBotClientContext:
   context = new ZaloBotClientContext("sendMessage", POST)
   → Propagator.Setter = (request, key, value) -> request.getHeaders().put(key, value)
   → remoteServiceName = "Zalo Bot API"
       │
       ▼
4. Create Observation from enum:
   ZaloBotClientObservation.API_REQUEST.observation(
       null,                              // no custom convention
       DefaultConvention.INSTANCE,        // fallback
       () -> context,                     // context supplier
       observationRegistry                // from Spring Boot
   )
       │
       ▼
5. observation.start()
   → DefaultMeterObservationHandler.onStart(context):
       Timer.Sample sample = Timer.start(clock);
       context.put(Timer.Sample.class, sample);
   → PropagatingSenderTracingObservationHandler.onStart(context):
       Span span = tracer.nextSpan().name("sendMessage").start();
       context.put(TracingContext.class, new TracingContext(span));
       // Inject trace headers via Propagator.Setter:
       request.getHeaders().put("traceparent", "00-abc123-def456-01");
       │
       ▼
6. observation.openScope()
   → Span placed in ThreadLocal (CurrentTraceContext)
   → Observation placed in ThreadLocal (ObservationRegistry)
       │
       ▼
7. HTTP Request executes:
   POST https://bot-api.zaloplatforms.com/bot{token}/sendMessage
   Headers: Content-Type: application/json
            traceparent: 00-abc123-def456-01     ← injected by tracing handler
   Body: {"chat_id": "12345", "text": "Hello"}
       │
       ▼
8. Response received: 200 OK
   context.setResponse(response)
       │
       ▼
9. observation.stop()
   → Convention resolves tags:
       Low-cardinality:
         zalobot.method = "sendMessage"
         outcome = "SUCCESS"
         exception = "none"
       High-cardinality:
         zalobot.request.url = "https://bot-api.zaloplatforms.com/bot.../sendMessage"
   → DefaultMeterObservationHandler.onStop(context):
       Timer.Sample sample = context.get(Timer.Sample.class);
       sample.stop(Timer.builder("zalobot.client.requests")
           .tags("zalobot.method", "sendMessage", "outcome", "SUCCESS", "exception", "none")
           .register(meterRegistry));
       // Metric recorded: zalobot.client.requests{method=sendMessage,outcome=SUCCESS} = 45ms
   → TracingHandler.onStop(context):
       span.tag("zalobot.method", "sendMessage");
       span.tag("zalobot.request.url", "https://...");
       span.end();
       // Span recorded: name=sendMessage, duration=45ms, traceId=abc123
       │
       ▼
10. Result in monitoring:
    Prometheus: zalobot_client_requests_seconds_count{method="sendMessage",outcome="SUCCESS"} 1
    Prometheus: zalobot_client_requests_seconds_sum{method="sendMessage",outcome="SUCCESS"} 0.045
    Zipkin: Span(name=sendMessage, service=my-app, remoteService=Zalo Bot API, duration=45ms)
```

## 8.4 End-to-End Walkthrough: Receiving and Processing an Update

```
1. PollingLoop calls:
   client.getUpdates().body(new GetUpdates(30)).retrieve().call(...)
   → Observation for the HTTP call (same as above)
       │
       ▼
2. Response enqueued:
   queue.put(update)
       │
       ▼
3. ProcessingLoop dequeues:
   update = queue.poll(1, TimeUnit.SECONDS)
       │
       ▼
4. Create ZaloBotListenerContext:
   context = new ZaloBotListenerContext("default", update)
   → eventName = update.eventName()  // e.g., "text_message"
       │
       ▼
5. Create Observation:
   ZaloBotListenerObservation.LISTENER_OBSERVATION.observation(...)
       │
       ▼
6. observation.start() → Timer.Sample + Span
       │
       ▼
7. observation.openScope() → ThreadLocal binding
       │
       ▼
8. listener.onUpdate(update) → user's business logic
       │
       ▼
9. observation.stop()
   → Metric: zalobot.listener{listener.id=default, event.name=text_message} = 12ms
   → Span: name=text_message process, duration=12ms
```

## 8.5 Artifact Summary Table

| # | Artifact | File | Module | Pattern From |
|---|----------|------|--------|--------------|
| 1 | Client Context | `ZaloBotClientContext.java` | zalobot-client | `KafkaRecordSenderContext` |
| 2 | Client Convention Interface | `ZaloBotClientObservationConvention.java` | zalobot-client | `KafkaTemplateObservationConvention` |
| 3 | Client Documentation Enum | `ZaloBotClientObservation.java` | zalobot-client | `KafkaTemplateObservation` |
| 4 | Client Integration | `DefaultZaloBotClient.exchangeInternal()` | zalobot-client | `KafkaTemplate.observeSend()` |
| 5 | Listener Context | `ZaloBotListenerContext.java` | zalobot-listener | `KafkaRecordReceiverContext` (simplified) |
| 6 | Listener Convention Interface | `ZaloBotListenerObservationConvention.java` | zalobot-listener | `KafkaListenerObservationConvention` |
| 7 | Listener Documentation Enum | `ZaloBotListenerObservation.java` | zalobot-listener | `KafkaListenerObservation` |
| 8 | Listener Integration | `ZaloUpdateListenerContainer.ProcessingLoop` | zalobot-listener | `KafkaMessageListenerContainer:2845` |
| 9 | Client Auto-Config | `ZaloBotClientAutoConfiguration.java` | zalobot-spring-boot | `RestClientObservationAutoConfiguration` |
| 10 | Listener Auto-Config | `ZaloBotListenerAutoConfiguration.java` | zalobot-spring-boot | `WebMvcObservationAutoConfiguration` |

## 8.6 Testing Strategy

### Unit Testing Observations

Use `TestObservationRegistry` from `micrometer-observation-test`:

```java
@Test
void sendMessageRecordsObservation() {
    TestObservationRegistry registry = TestObservationRegistry.create();
    ZaloBotClient client = ZaloBotClient.builder()
        .botToken("test-token")
        .observationRegistry(registry)
        .requestFactory(mockFactory)
        .build();

    client.sendMessage()
        .body(new SendMessage("chat1", "hello"))
        .retrieve()
        .call(SendMessageResult.class);

    // Verify observation was recorded
    TestObservationRegistryAssert.assertThat(registry)
        .hasObservationWithNameEqualTo("zalobot.client.requests")
        .that()
        .hasLowCardinalityKeyValue("zalobot.method", "sendMessage")
        .hasLowCardinalityKeyValue("outcome", "SUCCESS")
        .hasLowCardinalityKeyValue("exception", "none")
        .hasBeenStarted()
        .hasBeenStopped();
}

@Test
void apiErrorRecordsExceptionTag() {
    TestObservationRegistry registry = TestObservationRegistry.create();
    // ... setup mock to return error response ...

    assertThrows(ZaloBotApiException.class, () ->
        client.sendMessage()
            .body(new SendMessage("chat1", "hello"))
            .retrieve()
            .call(SendMessageResult.class)
    );

    TestObservationRegistryAssert.assertThat(registry)
        .hasObservationWithNameEqualTo("zalobot.client.requests")
        .that()
        .hasLowCardinalityKeyValue("outcome", "ERROR")
        .hasLowCardinalityKeyValue("exception", "ZaloBotApiException")
        .hasError();
}
```

### Test Dependencies

```xml
<!-- In zalobot-client/pom.xml, test scope -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-observation-test</artifactId>
    <scope>test</scope>
</dependency>
```

> ★ **Insight** ─────────────────────────────────────
> - **`TestObservationRegistry` is purpose-built for testing**: It records all observations and provides fluent assertions. You don't need to set up a real MeterRegistry or Tracer — the test registry captures everything. Spring Kafka's observation tests use this exact approach (see `ObservationIntegrationTests.java`).
> - **Test the convention, not the handler**: Your tests should verify that the correct tags are produced (convention behavior), not that Prometheus received a metric (handler behavior). Handler behavior is Micrometer's responsibility, not yours.
> ─────────────────────────────────────────────────────

## 8.7 Maven Dependency Summary

```xml
<!-- zalobot-client/pom.xml -->
<dependencies>
    <!-- Existing -->
    <dependency>
        <groupId>dev.linhvu</groupId>
        <artifactId>zalobot-core</artifactId>
    </dependency>

    <!-- NEW: Observation API (optional — not forced on users) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-observation</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- NEW: Test support -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-observation-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<!-- zalobot-listener/pom.xml -->
<dependencies>
    <!-- Existing -->
    <dependency>
        <groupId>dev.linhvu</groupId>
        <artifactId>zalobot-client</artifactId>
    </dependency>

    <!-- NEW: Observation API (optional) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-observation</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## 8.8 What We Didn't Cover (Future Enhancements)

| Feature | Why Deferred | When to Add |
|---------|-------------|-------------|
| **Polling loop observation** | Long-polling has complex lifecycle (timeout is normal, not an error) | After basic client/listener observations are proven |
| **Queue depth metrics** | Requires `Gauge` (not Observation) — different Micrometer API | When operational monitoring is needed |
| **Connection pool metrics** | OkHttp3 and JDK HttpClient have their own metrics | When users report HTTP-level issues |
| **Custom MeterBinder** | For SDK-wide metrics (active listeners, total messages) | When dashboard requirements are clear |
| **@Observed annotation** | AOP-based; useful for user's business methods, not SDK internals | Document as user-facing feature |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **6 new files** | 3 per domain (client + listener): Context, Convention interface, Documentation enum |
| **6 modified files** | Client (3), Listener (2), Spring Boot auto-config (2) minus overlap = 6 |
| **Module placement** | Observation classes in the module that performs the operation |
| **No new modules needed** | Unlike Spring Boot's separate `spring-boot-micrometer-*` modules, SDK is small enough to colocate |
| **Testing** | `TestObservationRegistry` for unit tests; verify tags, not handler behavior |
| **Zero impact on existing users** | NOOP default + optional Maven dependency = no changes unless opted in |

This concludes the guide. The pattern is identical across Spring Framework, Spring Kafka, and Micrometer core — learn it once, apply it everywhere.
