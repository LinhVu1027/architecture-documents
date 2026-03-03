# Chapter 3: The Convention Pattern — Naming and Tagging

## What You'll Learn
- Why naming and tagging are separated from observation creation
- The 3-level convention fallback: custom → global → default
- The difference between low-cardinality and high-cardinality tags
- How to define a `ObservationConvention` interface and its default implementation
- How conventions enable user customization without changing library code

---

## 3.1 The Problem: Who Decides the Metric Name?

When you instrument `exchangeInternal()`, someone needs to decide:
- What's the observation name? (`zalobot.client.requests`? `zalo.api.calls`?)
- What tags to attach? (`method=sendMessage`? `operation=send_message`?)
- Which tags are low-cardinality (safe for Prometheus) vs high-cardinality (traces only)?

If the **library** hardcodes these, users can't customize. If the **user** must provide them, the library can't ship useful defaults.

```java
// ❌ Hardcoded in library — user can't change naming
observation.lowCardinalityKeyValue("method", methodPath);  // What if user wants "operation"?

// ❌ User must configure everything — tedious
ZaloBotClient.builder()
    .observationName("my.custom.name")
    .tagKeyForMethod("operation")
    .tagKeyForStatus("http.status")
    ...  // explosion of configuration
```

## 3.2 The Solution: Conventions

A **Convention** is a strategy that encapsulates all naming and tagging decisions:

```java
public interface ObservationConvention<T extends Observation.Context> {

    // Technical name (used as metric name, e.g., "zalobot.client.requests")
    String getName();

    // Contextual name (used as span name, e.g., "sendMessage")
    default String getContextualName(T context) {
        return getName();
    }

    // Tags safe for metrics (bounded cardinality)
    default KeyValues getLowCardinalityKeyValues(T context) {
        return KeyValues.empty();
    }

    // Tags only for traces (unbounded cardinality)
    default KeyValues getHighCardinalityKeyValues(T context) {
        return KeyValues.empty();
    }

    // Type check: does this convention apply to this context?
    boolean supportsContext(Observation.Context context);
}
```

## 3.3 Low-Cardinality vs High-Cardinality

This distinction is **critical** for production systems:

```
┌─────────────────────────────────────────────────────────────────┐
│ LOW CARDINALITY (finite set of values)                          │
│ → Safe for metrics (Prometheus labels)                          │
│ → Used in dashboards and alerts                                 │
│                                                                 │
│ Examples:                                                       │
│   method = sendMessage | getUpdates | getMe | sendPhoto         │
│   status = success | error                                      │
│   exception = none | ZaloBotApiException | IOException          │
│                                                                 │
│ Why: Prometheus creates a time series per unique label combo.   │
│ 6 methods × 2 statuses × 3 exceptions = 36 time series ✓       │
├─────────────────────────────────────────────────────────────────┤
│ HIGH CARDINALITY (unbounded set of values)                      │
│ → Only for traces (NOT metrics)                                 │
│ → Used for drill-down investigation                             │
│                                                                 │
│ Examples:                                                       │
│   chat.id = "user_12345" | "user_67890" | ...                   │
│   request.url = "https://bot-api.zaloplatforms.com/bot.../..."  │
│   message.id = "msg_abc123"                                     │
│                                                                 │
│ Why: 1M unique chat IDs = 1M time series in Prometheus = OOM 💥 │
│ But in traces, each span has its own tags — no explosion.       │
└─────────────────────────────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - **Cardinality explosion is the #1 production metrics pitfall.** Spring Framework famously uses `uri=/users/{id}` (the route template, not `/users/12345`) as a low-cardinality tag. If your convention accidentally uses the full URL as a metric label, you'll create millions of time series and crash Prometheus.
> - **Rule of thumb**: If the number of distinct values can grow with traffic/users, it's high-cardinality. If it's bounded by your API surface, it's low-cardinality.
> ─────────────────────────────────────────────────────

## 3.4 The Two-Interface Pattern

Every Spring project defines conventions as a **pair**: an interface and a default implementation.

### Step 1: Convention Interface (Marker)

```java
/**
 * Convention for Zalo Bot API client observations.
 * Users can implement this to customize naming/tagging.
 */
public interface ZaloBotClientObservationConvention
        extends ObservationConvention<ZaloBotClientContext> {

    @Override
    default boolean supportsContext(Observation.Context context) {
        return context instanceof ZaloBotClientContext;
    }

    @Override
    default String getName() {
        return "zalobot.client.requests";
    }
}
```

### Step 2: Default Implementation

```java
/**
 * Default convention shipped with the SDK.
 * Provides sensible defaults; users can override by implementing
 * {@link ZaloBotClientObservationConvention}.
 */
public class DefaultZaloBotClientObservationConvention
        implements ZaloBotClientObservationConvention {

    public static final DefaultZaloBotClientObservationConvention INSTANCE =
            new DefaultZaloBotClientObservationConvention();

    @Override
    public String getContextualName(ZaloBotClientContext context) {
        return context.getMethodPath();  // "sendMessage", "getUpdates", etc.
    }

    @Override
    public KeyValues getLowCardinalityKeyValues(ZaloBotClientContext context) {
        return KeyValues.of(
            KeyValue.of("method", context.getMethodPath()),
            KeyValue.of("outcome", context.isSuccess() ? "SUCCESS" : "ERROR"),
            KeyValue.of("exception", context.getExceptionName())
        );
    }

    @Override
    public KeyValues getHighCardinalityKeyValues(ZaloBotClientContext context) {
        return KeyValues.of(
            KeyValue.of("zalobot.request.url", context.getRequestUrl())
        );
    }
}
```

## 3.5 The 3-Level Convention Fallback

When creating an observation, Micrometer uses a 3-level fallback:

```java
Observation observation = ZaloBotClientObservation.API_REQUEST
    .observation(
        customConvention,      // Level 1: per-observation custom (nullable)
        DEFAULT_CONVENTION,    // Level 3: library default (always provided)
        () -> context,
        registry               // Level 2: global conventions registered here
    );
```

Resolution order:

```
Level 1: customConvention (passed directly)
  │
  ├── NOT null → Use it
  │
  └── null → Check Level 2
              │
              ├── registry has GlobalObservationConvention
              │   that supportsContext(myContext) → Use it
              │
              └── No matching global → Level 3
                                        │
                                        └── DEFAULT_CONVENTION (library default)
```

This gives users three customization points:

```java
// Level 1: Per-observation (most specific)
ZaloBotClient.builder()
    .observationConvention(new MyCustomConvention())  // Only this client
    .build();

// Level 2: Global (application-wide, via Spring Boot)
@Bean
ZaloBotClientObservationConvention globalConvention() {
    return new MyGlobalConvention();  // All Zalobot observations
}

// Level 3: Library default (no configuration needed)
// DefaultZaloBotClientObservationConvention.INSTANCE is always used as fallback
```

> ★ **Insight** ─────────────────────────────────────
> - **Why the singleton INSTANCE pattern**: The default convention is stateless — it only reads from the Context. A singleton avoids creating a new object per observation. Spring Kafka and Spring Framework both use this pattern: `DefaultKafkaTemplateObservationConvention.INSTANCE`, `DefaultClientRequestObservationConvention.INSTANCE`.
> - **The interface exists for `supportsContext()` only**: You might wonder why there's a separate interface with just `supportsContext()` and `getName()`. It exists so that `GlobalObservationConvention` beans can be type-checked at registration time. Without it, the registry wouldn't know which conventions apply to which contexts.
> ─────────────────────────────────────────────────────

## 3.6 How Spring Kafka Does It

Let's see the real pattern in Spring Kafka to verify our understanding:

**Convention Interface** (`KafkaTemplateObservationConvention.java`):
```java
public interface KafkaTemplateObservationConvention
        extends ObservationConvention<KafkaRecordSenderContext> {

    @Override
    default boolean supportsContext(Observation.Context context) {
        return context instanceof KafkaRecordSenderContext;
    }
}
```

**Default Convention** (inner class in `KafkaTemplateObservation.java`):
```java
public static class DefaultKafkaTemplateObservationConvention
        implements KafkaTemplateObservationConvention {

    public static final DefaultKafkaTemplateObservationConvention INSTANCE =
            new DefaultKafkaTemplateObservationConvention();

    @Override
    public KeyValues getLowCardinalityKeyValues(KafkaRecordSenderContext context) {
        return KeyValues.of(
            TemplateLowCardinalityTags.BEAN_NAME.withValue(context.getBeanName()),
            TemplateLowCardinalityTags.MESSAGING_SYSTEM.withValue("kafka"),
            TemplateLowCardinalityTags.MESSAGING_OPERATION.withValue("publish"),
            TemplateLowCardinalityTags.MESSAGING_DESTINATION_NAME.withValue(context.getDestination())
        );
    }

    @Override
    public String getName() {
        return "spring.kafka.template";
    }

    @Override
    public String getContextualName(KafkaRecordSenderContext context) {
        return context.getDestination() + " send";  // e.g., "my-topic send"
    }
}
```

**Usage in KafkaTemplate** (`KafkaTemplate.java:819-822`):
```java
Observation observation = KafkaTemplateObservation.TEMPLATE_OBSERVATION.observation(
    this.observationConvention,                            // custom (nullable)
    DefaultKafkaTemplateObservationConvention.INSTANCE,   // default
    () -> new KafkaRecordSenderContext(producerRecord, this.beanName, this::clusterId),
    this.observationRegistry
);
```

Identical pattern. Every Spring project follows this exactly.

## 3.7 Connection to Real Source

| Concept | Spring Kafka | Spring Framework |
|---------|-------------|-----------------|
| Convention interface | `KafkaTemplateObservationConvention.java` | `ServerRequestObservationConvention.java` |
| Default convention | `DefaultKafkaTemplateObservationConvention` (inner class) | `DefaultServerRequestObservationConvention.java` |
| Convention in constructor | `KafkaTemplate.setObservationConvention()` | `ServerHttpObservationFilter(registry, convention)` |
| Fallback to default | `KafkaTemplate.observeSend():819` | `DefaultRestClient:603-606` |
| Singleton INSTANCE | `DefaultKafkaTemplateObservationConvention.INSTANCE` | — (separate class, not inner) |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Convention** | Strategy that encapsulates naming + tagging decisions, separate from observation creation |
| **Low-cardinality** | Bounded tag values safe for metrics (method, status, exception type) |
| **High-cardinality** | Unbounded tag values for traces only (user ID, full URL, message ID) |
| **3-level fallback** | custom → global → default; gives users multiple override points |
| **Convention interface** | Marker with `supportsContext()` + default `getName()`; enables type-safe registration |
| **Default convention** | Singleton INSTANCE with sensible defaults; shipped by the library |

**Next: Chapter 4** — The ObservationContext: how to create a domain-specific context that carries your API-specific data.
