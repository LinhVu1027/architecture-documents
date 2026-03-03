# Chapter 7: Spring Boot Auto-Configuration — Wiring It All Together

## What You'll Learn
- How Spring Boot auto-configures `ObservationRegistry` and registers handlers
- How to auto-wire the registry into Zalobot's client and listener container
- The `ObservationRegistryCustomizer` pattern for global customization
- How to add observation support to `ZaloBotClientAutoConfiguration`
- The dependency management strategy for optional Micrometer support

---

## 7.1 How Spring Boot Sets Up Observation Infrastructure

Before we wire Zalobot, we need to understand what Spring Boot already provides when `spring-boot-starter-actuator` is on the classpath:

```
Spring Boot starts
    │
    ▼
ObservationAutoConfiguration
    ├── Creates ObservationRegistry.create() bean
    ├── ObservationRegistryPostProcessor:
    │   ├── Registers ObservationPredicates (enable/disable by name)
    │   ├── Registers GlobalObservationConventions (beans)
    │   ├── Registers ObservationHandlerGroups:
    │   │   ├── metricsObservationHandlerGroup (from MetricsAutoConfiguration)
    │   │   └── tracingObservationHandlerGroup (from MicrometerTracingAutoConfiguration)
    │   ├── Registers ObservationFilters
    │   └── Applies ObservationRegistryCustomizers
    │
    ▼
MetricsAutoConfiguration
    ├── Creates MeterRegistry (Prometheus, OTLP, etc.)
    ├── Creates DefaultMeterObservationHandler
    └── Groups as metricsObservationHandlerGroup
    │
    ▼
MicrometerTracingAutoConfiguration (if tracing starter present)
    ├── Creates Tracer (Brave or OpenTelemetry)
    ├── Creates DefaultTracingObservationHandler
    ├── Creates PropagatingSenderTracingObservationHandler
    ├── Creates PropagatingReceiverTracingObservationHandler
    └── Groups as tracingObservationHandlerGroup
```

**Result**: A fully configured `ObservationRegistry` bean is available in the application context with both metrics and tracing handlers registered.

## 7.2 Auto-Wiring into Zalobot Client

The goal: When a user has both `zalobot-spring-boot-starter` and `spring-boot-starter-actuator` on their classpath, the `ObservationRegistry` should be automatically wired into the Zalobot client — **no manual configuration required**.

### Modifying `ZaloBotClientAutoConfiguration`

```java
@AutoConfiguration
@ConditionalOnClass(ZaloBotClient.class)
@ConditionalOnProperty(name = "zalobot.bot-token")
@EnableConfigurationProperties(ZaloBotProperties.class)
public class ZaloBotClientAutoConfiguration {

    @Bean
    @Scope("prototype")
    ZaloBotClient.Builder zaloBotClientBuilder(
            ZaloBotProperties properties,
            ObjectProvider<ZaloBotClientCustomizer> customizers,
            ObjectProvider<ObservationRegistry> observationRegistry,         // NEW
            ObjectProvider<ZaloBotClientObservationConvention> convention) { // NEW

        ZaloBotClient.Builder builder = ZaloBotClient.builder()
                .botToken(properties.getBotToken());

        // Apply observation if registry is available
        observationRegistry.ifAvailable(builder::observationRegistry);      // NEW
        convention.ifAvailable(builder::observationConvention);             // NEW

        // Apply user customizers
        customizers.orderedStream().forEach(c -> c.customize(builder));

        return builder;
    }

    @Bean
    @ConditionalOnMissingBean
    ZaloBotClient zaloBotClient(ZaloBotClient.Builder builder) {
        return builder.build();
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - **`ObjectProvider<ObservationRegistry>` instead of `@Autowired`**: This is the Spring Boot pattern for optional dependencies. If `spring-boot-starter-actuator` is NOT on the classpath, `ObservationRegistry` doesn't exist — `ObjectProvider.ifAvailable()` gracefully does nothing. If we used `@Autowired` directly, the application would fail to start without actuator.
> - **Why `convention` is also an `ObjectProvider`**: This allows users to register a custom convention as a Spring bean. Spring Boot's `WebMvcObservationAutoConfiguration` does the same: `@Bean ServerRequestObservationConvention` is optional and auto-detected.
> ─────────────────────────────────────────────────────

## 7.3 Auto-Wiring into Zalobot Listener Container

### Modifying `ZaloBotListenerAutoConfiguration`

```java
@AutoConfiguration(after = ZaloBotClientAutoConfiguration.class)
@ConditionalOnClass(UpdateListenerContainer.class)
@ConditionalOnBean(ZaloBotClient.class)
@ConditionalOnProperty(name = "zalobot.listener.enabled", matchIfMissing = true)
public class ZaloBotListenerAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    ContainerProperties containerProperties(
            ZaloBotProperties properties,
            ObjectProvider<ObservationRegistry> observationRegistry,             // NEW
            ObjectProvider<ZaloBotListenerObservationConvention> convention) {   // NEW

        ContainerProperties containerProperties = new ContainerProperties();
        ZaloBotProperties.Listener listener = properties.getListener();
        containerProperties.setPollTimeout(listener.getPollTimeout());
        containerProperties.setShutdownTimeout(listener.getShutdownTimeout());
        containerProperties.setBackOffInterval(listener.getBackOffInterval());
        containerProperties.setMaxBackOffInterval(listener.getMaxBackOffInterval());
        containerProperties.setQueueCapacity(listener.getQueueCapacity());
        containerProperties.setProcessingConcurrency(listener.getProcessingConcurrency());

        // Wire observation support
        observationRegistry.ifAvailable(containerProperties::setObservationRegistry);  // NEW
        convention.ifAvailable(containerProperties::setObservationConvention);         // NEW

        return containerProperties;
    }

    // ... rest unchanged ...
}
```

## 7.4 How Spring Kafka Does It (The `afterSingletonsInstantiated` Pattern)

Spring Kafka uses an alternative approach — it auto-discovers the registry from the application context:

```java
// KafkaTemplate.java:499-516
@Override
public void afterSingletonsInstantiated() {
    if (this.observationEnabled && this.applicationContext != null) {
        if (this.observationRegistry.isNoop()) {
            this.observationRegistry = this.applicationContext
                .getBeanProvider(ObservationRegistry.class)
                .getIfUnique(() -> this.observationRegistry);
        }
    }
}
```

This pattern:
1. Implements `SmartInitializingSingleton`
2. After all beans are created, checks if `ObservationRegistry` bean exists
3. If found, replaces the NOOP registry

**For Zalobot SDK**, the auto-configuration approach (Section 7.2) is simpler and more idiomatic for Spring Boot libraries. The `afterSingletonsInstantiated` pattern is used by Spring Kafka because `KafkaTemplate` can be used both with and without Spring Boot.

## 7.5 The Dependency Management Strategy

### Maven Dependencies

The key is making `micrometer-observation` an **optional** dependency:

**In `zalobot-client/pom.xml`:**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-observation</artifactId>
    <optional>true</optional>
</dependency>
```

**In `zalobot-spring-boot/pom.xml`:**
```xml
<!-- No need to declare micrometer — Spring Boot Actuator brings it -->
<!-- It's already available when user adds spring-boot-starter-actuator -->
```

**In `zalobot-spring-boot-starter/pom.xml`:**
```xml
<!-- Does NOT pull in spring-boot-starter-actuator — user opts in -->
<dependency>
    <groupId>dev.linhvu</groupId>
    <artifactId>zalobot-spring-boot</artifactId>
</dependency>
```

**User's application `pom.xml`:**
```xml
<!-- Base SDK — no observation overhead -->
<dependency>
    <groupId>dev.linhvu</groupId>
    <artifactId>zalobot-spring-boot-starter</artifactId>
</dependency>

<!-- Opt-in to metrics + tracing -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Optional: distributed tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
```

> ★ **Insight** ─────────────────────────────────────
> - **`<optional>true</optional>` is the key**: This means `micrometer-observation` is available at compile time (for the SDK's observation code) but NOT transitively pulled into user applications. Users only get it when they add `spring-boot-starter-actuator` which brings `micrometer-observation` as a transitive dependency.
> - **What if the user doesn't have micrometer on the classpath?**: Since `ObservationRegistry.NOOP` is the default, and the observation code only runs if a non-NOOP registry is set, the code paths are safe. However, the observation classes must be available at compile time — so the `<optional>true</optional>` dependency ensures they're on the classpath during compilation.
> ─────────────────────────────────────────────────────

## 7.6 Configuration Properties

Add optional observation properties:

```java
@ConfigurationProperties(prefix = "zalobot")
public class ZaloBotProperties {
    // ... existing properties ...

    public static class Client {
        // ... existing properties ...
    }

    public static class Listener {
        // ... existing properties ...

        /**
         * Whether to enable observation for the listener container.
         */
        private boolean observationEnabled = true;

        public boolean isObservationEnabled() {
            return this.observationEnabled;
        }

        public void setObservationEnabled(boolean observationEnabled) {
            this.observationEnabled = observationEnabled;
        }
    }
}
```

```yaml
# application.yml
zalobot:
  bot-token: ${ZALO_BOT_TOKEN}
  listener:
    observation-enabled: true  # default; set to false to disable

# Spring Boot's built-in observation control also works:
management:
  observations:
    enable:
      zalobot.client.requests: true    # per-observation toggle
      zalobot.listener: true
```

## 7.7 The Complete Auto-Configuration Flow

```
User adds to classpath:
  ├── zalobot-spring-boot-starter    (SDK)
  └── spring-boot-starter-actuator   (Observability, optional)
      │
      ▼
Spring Boot Auto-Configuration:
  │
  ├── ObservationAutoConfiguration
  │   └── Creates ObservationRegistry with metric + tracing handlers
  │
  ├── ZaloBotClientAutoConfiguration
  │   ├── Detects ObservationRegistry bean (via ObjectProvider)
  │   ├── Detects ZaloBotClientObservationConvention bean (optional)
  │   └── Wires both into ZaloBotClient.Builder
  │
  └── ZaloBotListenerAutoConfiguration
      ├── Detects ObservationRegistry bean (via ObjectProvider)
      ├── Detects ZaloBotListenerObservationConvention bean (optional)
      └── Wires both into ContainerProperties
          │
          ▼
Runtime behavior:
  │
  ├── Every ZaloBotClient API call:
  │   └── Produces observation → metrics timer + tracing span
  │
  └── Every listener update processing:
      └── Produces observation → metrics timer + tracing span

Result in Grafana:
  ├── Dashboard: "Zalobot API Request Latency by Method"
  ├── Dashboard: "Zalobot Listener Processing Time"
  ├── Alert: "Error rate > 5% on sendMessage"
  └── Traces: Full request flow visible in Zipkin/Jaeger
```

## 7.8 Connection to Real Source

| Concept | Source |
|---------|--------|
| ObservationAutoConfiguration | `spring-boot/module/spring-boot-micrometer-observation/.../ObservationAutoConfiguration.java` |
| ObservationRegistryPostProcessor | `spring-boot/module/spring-boot-micrometer-observation/.../ObservationRegistryPostProcessor.java` |
| ObservationRegistryConfigurer | `spring-boot/module/spring-boot-micrometer-observation/.../ObservationRegistryConfigurer.java` |
| MetricsAutoConfiguration | `spring-boot/module/spring-boot-micrometer-metrics/.../MetricsAutoConfiguration.java` |
| MicrometerTracingAutoConfiguration | `spring-boot/module/spring-boot-micrometer-tracing/.../MicrometerTracingAutoConfiguration.java` |
| WebMvcObservationAutoConfiguration | `spring-boot/module/spring-boot-webmvc/.../WebMvcObservationAutoConfiguration.java` |
| RestClientObservationAutoConfiguration | `spring-boot/module/spring-boot-restclient/.../RestClientObservationAutoConfiguration.java` |
| KafkaTemplate.afterSingletonsInstantiated() | `spring-kafka/.../core/KafkaTemplate.java:499-516` |
| ContainerProperties observation fields | `spring-kafka/.../listener/ContainerProperties.java:285-309` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ObjectProvider pattern** | Optional dependency injection — gracefully handles missing `ObservationRegistry` |
| **Auto-configuration** | If actuator is on classpath, observation is automatic; otherwise NOOP |
| **Optional Maven dependency** | `<optional>true</optional>` on `micrometer-observation` — not forced on users |
| **Convention as bean** | Users can register a custom convention as a Spring bean; auto-detected |
| **afterSingletonsInstantiated** | Alternative pattern (used by Spring Kafka) for non-Spring Boot contexts |
| **management.observations.enable** | Spring Boot's built-in per-observation toggle works automatically |

**Next: Chapter 8** — Putting It All Together: the complete file listing, module structure, and end-to-end walkthrough.
