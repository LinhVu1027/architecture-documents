# Chapter 5: ObservationDocumentation — Self-Documenting Enums

## What You'll Learn
- Why observations are defined as enum constants
- How `ObservationDocumentation` serves as both documentation AND factory
- The `KeyName` inner enums for compile-time tag documentation
- How the `observation()` factory method works
- How to create the Zalobot observation documentation enum

---

## 5.1 The Problem: What Observations Does This Library Produce?

As a user of a library, you need to know:
1. What observations exist? (names)
2. What tags does each observation produce? (metric labels)
3. What's the default convention? (can I override it?)

Without a structured way to document this, you'd need to dig through source code or hope for good Javadoc. Micrometer solves this with **enum-based documentation** that is machine-readable and self-documenting.

## 5.2 The ObservationDocumentation Interface

```java
public interface ObservationDocumentation {

    // What's the default convention for this observation?
    Class<? extends ObservationConvention<?>> getDefaultConvention();

    // What low-cardinality tags are expected?
    KeyName[] getLowCardinalityKeyNames();

    // What high-cardinality tags are expected?
    default KeyName[] getHighCardinalityKeyNames() {
        return new KeyName[0];
    }

    // Factory method: create an Observation from this documentation
    default Observation observation(
            @Nullable ObservationConvention<? extends Context> customConvention,
            ObservationConvention<? extends Context> defaultConvention,
            Supplier<? extends Context> contextSupplier,
            ObservationRegistry registry) {
        // ... convention resolution + observation creation
    }
}
```

The `observation()` factory method is what makes enums double as factories.

## 5.3 The KeyName Interface

Each tag is documented as an inner enum implementing `KeyName`:

```java
public interface KeyName {
    // The actual tag key string (e.g., "messaging.system", "http.method")
    String asString();
}
```

This provides **compile-time documentation** of all possible tags.

## 5.4 Building the Zalobot Client Observation Enum

Following the pattern from Spring Kafka (`KafkaTemplateObservation`) and Spring Framework (`ServerHttpObservationDocumentation`):

```java
/**
 * {@link ObservationDocumentation} for Zalo Bot client operations.
 *
 * @author Linh Vu
 * @since 0.1.0
 */
public enum ZaloBotClientObservation implements ObservationDocumentation {

    /**
     * Observation for Zalo Bot API HTTP requests.
     */
    API_REQUEST {

        @Override
        public Class<? extends ObservationConvention<? extends Observation.Context>>
                getDefaultConvention() {
            return DefaultZaloBotClientObservationConvention.class;
        }

        @Override
        public KeyName[] getLowCardinalityKeyNames() {
            return LowCardinalityKeyNames.values();
        }

        @Override
        public KeyName[] getHighCardinalityKeyNames() {
            return HighCardinalityKeyNames.values();
        }
    };

    // ──────────────────────────────────────────────────────
    // Tag documentation
    // ──────────────────────────────────────────────────────

    /**
     * Low-cardinality tag names (safe for metrics).
     */
    public enum LowCardinalityKeyNames implements KeyName {

        /**
         * Zalo Bot API method path (e.g., "sendMessage", "getUpdates").
         */
        METHOD {
            @Override
            public String asString() {
                return "zalobot.method";
            }
        },

        /**
         * Request outcome: "SUCCESS" or "ERROR".
         */
        OUTCOME {
            @Override
            public String asString() {
                return "outcome";
            }
        },

        /**
         * Exception class name or "none".
         */
        EXCEPTION {
            @Override
            public String asString() {
                return "exception";
            }
        }
    }

    /**
     * High-cardinality tag names (traces only).
     */
    public enum HighCardinalityKeyNames implements KeyName {

        /**
         * Full request URL.
         */
        REQUEST_URL {
            @Override
            public String asString() {
                return "zalobot.request.url";
            }
        }
    }

    // ──────────────────────────────────────────────────────
    // Default convention (singleton inner class)
    // ──────────────────────────────────────────────────────

    /**
     * Default {@link ZaloBotClientObservationConvention}.
     */
    public static class DefaultZaloBotClientObservationConvention
            implements ZaloBotClientObservationConvention {

        public static final DefaultZaloBotClientObservationConvention INSTANCE =
                new DefaultZaloBotClientObservationConvention();

        private static final KeyValue EXCEPTION_NONE =
                KeyValue.of(LowCardinalityKeyNames.EXCEPTION, KeyValue.NONE_VALUE);
        private static final KeyValue OUTCOME_SUCCESS =
                KeyValue.of(LowCardinalityKeyNames.OUTCOME, "SUCCESS");
        private static final KeyValue OUTCOME_ERROR =
                KeyValue.of(LowCardinalityKeyNames.OUTCOME, "ERROR");

        @Override
        public String getContextualName(ZaloBotClientContext context) {
            return context.getMethodPath();
        }

        @Override
        public KeyValues getLowCardinalityKeyValues(ZaloBotClientContext context) {
            return KeyValues.of(
                method(context),
                outcome(context),
                exception(context)
            );
        }

        @Override
        public KeyValues getHighCardinalityKeyValues(ZaloBotClientContext context) {
            return KeyValues.of(
                KeyValue.of(HighCardinalityKeyNames.REQUEST_URL, context.getRequestUrl())
            );
        }

        private KeyValue method(ZaloBotClientContext context) {
            return KeyValue.of(LowCardinalityKeyNames.METHOD, context.getMethodPath());
        }

        private KeyValue outcome(ZaloBotClientContext context) {
            return context.isSuccess() ? OUTCOME_SUCCESS : OUTCOME_ERROR;
        }

        private KeyValue exception(ZaloBotClientContext context) {
            Throwable error = context.getError();
            if (error != null) {
                String simpleName = error.getClass().getSimpleName();
                return KeyValue.of(LowCardinalityKeyNames.EXCEPTION,
                        simpleName.isEmpty() ? error.getClass().getName() : simpleName);
            }
            return EXCEPTION_NONE;
        }
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Why the default convention is an inner class of the enum**: This is the Spring Kafka pattern (see `KafkaTemplateObservation.DefaultKafkaTemplateObservationConvention`). It keeps the entire observation definition — enum, tags, and default convention — in a single file. Spring Framework sometimes uses a separate file (`DefaultServerRequestObservationConvention.java`), but the inner class approach is more cohesive for simpler observations.
> - **Pre-computed `KeyValue` constants** (like `EXCEPTION_NONE`, `OUTCOME_SUCCESS`): These avoid allocating new `KeyValue` objects on every observation. Since they're used on every API call, this micro-optimization adds up. Spring Kafka's `DefaultKafkaTemplateObservationConvention` does the same thing.
> ─────────────────────────────────────────────────────

## 5.5 The Listener Observation Enum

Similarly, for the listener container:

```java
/**
 * {@link ObservationDocumentation} for Zalo Bot listener operations.
 */
public enum ZaloBotListenerObservation implements ObservationDocumentation {

    /**
     * Observation for processing a received update.
     */
    LISTENER_OBSERVATION {

        @Override
        public Class<? extends ObservationConvention<? extends Observation.Context>>
                getDefaultConvention() {
            return DefaultZaloBotListenerObservationConvention.class;
        }

        @Override
        public KeyName[] getLowCardinalityKeyNames() {
            return LowCardinalityKeyNames.values();
        }
    };

    public enum LowCardinalityKeyNames implements KeyName {

        LISTENER_ID {
            @Override
            public String asString() {
                return "zalobot.listener.id";
            }
        },

        EVENT_NAME {
            @Override
            public String asString() {
                return "zalobot.event.name";
            }
        },

        EXCEPTION {
            @Override
            public String asString() {
                return "exception";
            }
        }
    }

    public static class DefaultZaloBotListenerObservationConvention
            implements ZaloBotListenerObservationConvention {

        public static final DefaultZaloBotListenerObservationConvention INSTANCE =
                new DefaultZaloBotListenerObservationConvention();

        @Override
        public String getContextualName(ZaloBotListenerContext context) {
            return context.getEventName() + " process";
        }

        @Override
        public KeyValues getLowCardinalityKeyValues(ZaloBotListenerContext context) {
            return KeyValues.of(
                KeyValue.of(LowCardinalityKeyNames.LISTENER_ID, context.getListenerId()),
                KeyValue.of(LowCardinalityKeyNames.EVENT_NAME, context.getEventName()),
                exception(context)
            );
        }

        private KeyValue exception(ZaloBotListenerContext context) {
            Throwable error = context.getError();
            if (error != null) {
                return KeyValue.of(LowCardinalityKeyNames.EXCEPTION,
                        error.getClass().getSimpleName());
            }
            return KeyValue.of(LowCardinalityKeyNames.EXCEPTION, KeyValue.NONE_VALUE);
        }
    }
}
```

## 5.6 How the observation() Factory Method Works

The `ObservationDocumentation.observation()` method is the key — it creates observations with proper convention resolution:

```java
// Pseudocode of the factory method:
default Observation observation(
        @Nullable ObservationConvention<?> customConvention,
        ObservationConvention<?> defaultConvention,
        Supplier<? extends Context> contextSupplier,
        ObservationRegistry registry) {

    // 1. If registry is NOOP, return NOOP observation immediately
    if (registry.isNoop()) {
        return Observation.NOOP;
    }

    // 2. Create the context (lazy — only if observation is enabled)
    Context context = contextSupplier.get();
    context.setName(getName());  // from the enum

    // 3. Resolve convention: custom → global → default
    ObservationConvention<?> convention = resolveConvention(
        customConvention, defaultConvention, registry);

    // 4. Create (but don't start) the observation
    return Observation.createNotStarted(convention, context, registry);
}
```

**Usage at the integration point:**

```java
// One line to create the observation — everything is wired
Observation observation = ZaloBotClientObservation.API_REQUEST
    .observation(
        this.observationConvention,                              // custom (nullable)
        DefaultZaloBotClientObservationConvention.INSTANCE,     // fallback
        () -> new ZaloBotClientContext(methodPath, httpMethod), // context supplier
        this.observationRegistry                                 // registry
    );
```

> ★ **Insight** ─────────────────────────────────────
> - **Lazy context creation via `Supplier`**: The context is only created if the observation is enabled. If `ObservationPredicate` disables it, the supplier is never called — avoiding object allocation for disabled observations.
> - **Enum as factory is a Micrometer innovation**: Unlike traditional enums that just hold constants, `ObservationDocumentation` enums actively create observations. This pattern ensures that every observation creation goes through the documented definition, preventing "undocumented" observations from sneaking in.
> ─────────────────────────────────────────────────────

## 5.7 Comparison Across Spring Projects

| Aspect | Spring Kafka | Spring Framework | Zalobot SDK |
|--------|-------------|-----------------|-------------|
| Enum class | `KafkaTemplateObservation` | `ServerHttpObservationDocumentation` | `ZaloBotClientObservation` |
| Enum constant | `TEMPLATE_OBSERVATION` | `HTTP_SERVLET_SERVER_REQUESTS` | `API_REQUEST` |
| Low-cardinality tags | `BEAN_NAME`, `MESSAGING_SYSTEM`, `MESSAGING_OPERATION`, `DESTINATION_NAME` | `METHOD`, `STATUS`, `URI`, `EXCEPTION`, `OUTCOME` | `METHOD`, `OUTCOME`, `EXCEPTION` |
| High-cardinality tags | — | `HTTP_URL` | `REQUEST_URL` |
| Default convention location | Inner class | Separate file | Inner class |
| Convention name | `spring.kafka.template` | `http.server.requests` | `zalobot.client.requests` |

## 5.8 Connection to Real Source

| Concept | Source File |
|---------|------------|
| `ObservationDocumentation` | `micrometer/micrometer-observation/.../docs/ObservationDocumentation.java` |
| `KeyName` | `micrometer/micrometer-observation/.../docs/ObservationDocumentation.java` (inner interface) |
| `KafkaTemplateObservation` | `spring-kafka/.../support/micrometer/KafkaTemplateObservation.java` |
| `KafkaListenerObservation` | `spring-kafka/.../support/micrometer/KafkaListenerObservation.java` |
| `ServerHttpObservationDocumentation` | `spring-framework/spring-web/.../server/observation/ServerHttpObservationDocumentation.java` |
| `ClientHttpObservationDocumentation` | `spring-framework/spring-web/.../client/observation/ClientHttpObservationDocumentation.java` |
| `JmsObservationDocumentation` | `micrometer/micrometer-jakarta9/.../jms/JmsObservationDocumentation.java` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ObservationDocumentation enum** | Self-documenting enum that serves as BOTH documentation AND observation factory |
| **KeyName inner enums** | Compile-time documentation of tag keys, with `asString()` returning the actual key |
| **getDefaultConvention()** | Points to the default convention class; used for documentation generation |
| **observation() factory** | Creates observation with proper convention resolution: custom → global → default |
| **Default convention as inner class** | Keeps the entire observation definition (enum + tags + convention) in one file |
| **Lazy context via Supplier** | Context only allocated if observation is enabled — zero cost when disabled |

**Next: Chapter 6** — The Integration Point: where and how to wire observations into `DefaultZaloBotClient` and `ZaloUpdateListenerContainer`.
