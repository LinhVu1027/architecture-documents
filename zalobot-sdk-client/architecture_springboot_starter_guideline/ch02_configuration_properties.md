# Chapter 2: Properties Binding with `@ConfigurationProperties`

> How does `zalobot.listener.poll-timeout=60s` in YAML become `Duration.ofSeconds(60)` in Java?

---

## The Annotation

```java
// ConfigurationProperties.java:48-52
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface ConfigurationProperties {

    @AliasFor("prefix")
    String value() default "";         // the property prefix, e.g. "zalobot"

    @AliasFor("value")
    String prefix() default "";

    boolean ignoreInvalidFields() default false;
    boolean ignoreUnknownFields() default true;
}
```

**Source:** `spring-boot/core/spring-boot/.../ConfigurationProperties.java:52`

You annotate a class with `@ConfigurationProperties("prefix")` and Spring Boot's `ConfigurationPropertiesBindingPostProcessor` automatically:
1. Reads all properties matching `prefix.*` from `application.yml` / `application.properties` / environment variables
2. Converts types (`String` → `Duration`, `String` → `int`, etc.)
3. Calls the corresponding setter methods (or constructor parameters)

---

## Registration: `@EnableConfigurationProperties`

For the properties class to be picked up, it must be registered. The standard way in auto-configuration is:

```java
// EnableConfigurationProperties.java:37-41
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EnableConfigurationPropertiesRegistrar.class)    // ← the magic
public @interface EnableConfigurationProperties {
    Class<?>[] value() default {};
}
```

**Source:** `spring-boot/core/spring-boot/.../EnableConfigurationProperties.java:40`

When you write `@EnableConfigurationProperties(ZaloBotProperties.class)` on your auto-configuration class, Spring:
1. Imports the `EnableConfigurationPropertiesRegistrar`
2. The registrar creates a bean definition for `ZaloBotProperties`
3. The `ConfigurationPropertiesBindingPostProcessor` binds properties to it

---

## Real Example: KafkaProperties

Let's look at how Spring Boot's Kafka module does it:

```java
// KafkaProperties.java:64-65
@ConfigurationProperties("spring.kafka")
public class KafkaProperties {

    private List<String> bootstrapServers = new ArrayList<>(
            Collections.singletonList("localhost:9092"));

    private @Nullable String clientId;

    private final Consumer consumer = new Consumer();
    private final Producer producer = new Producer();
    private final Admin admin = new Admin();
    // ...nested classes with their own properties...
}
```

**Source:** `module/spring-boot-kafka/.../KafkaProperties.java:64`

And it's registered in the auto-configuration class:

```java
// KafkaAutoConfiguration.java:84-86
@AutoConfiguration
@ConditionalOnClass(KafkaTemplate.class)
@EnableConfigurationProperties(KafkaProperties.class)    // ← registers KafkaProperties
public final class KafkaAutoConfiguration {
    // ...
}
```

**Source:** `module/spring-boot-kafka/.../KafkaAutoConfiguration.java:86`

This binding means that writing:

```yaml
spring:
  kafka:
    bootstrap-servers: broker1:9092,broker2:9092
    consumer:
      auto-offset-reset: earliest
```

...automatically sets `kafkaProperties.getBootstrapServers()` to `["broker1:9092", "broker2:9092"]`.

---

## Building: ZaloBotProperties

Now let's build our own. We need to map SDK defaults to properties:

### SDK Defaults We Must Mirror

| SDK Class | Field | Default Value | Source |
|---|---|---|---|
| `ZaloBotUrl` | `scheme` | `"https"` | `ZaloBotUrl.java:8` |
| `ZaloBotUrl` | `host` | `"bot-api.zaloplatforms.com"` | `ZaloBotUrl.java:8` |
| `ZaloBotUrl` | `port` | `443` | `ZaloBotUrl.java:8` |
| `ContainerProperties` | `pollTimeout` | `Duration.ofSeconds(30)` | `ContainerProperties.java:8` |
| `ContainerProperties` | `pollInterval` | `Duration.ofSeconds(0)` | `ContainerProperties.java:9` |
| `ContainerProperties` | `shutdownTimeout` | `Duration.ofSeconds(10)` | `ContainerProperties.java:10` |
| `ContainerProperties` | `backOffInterval` | `Duration.ofSeconds(1)` | `ContainerProperties.java:11` |
| `ContainerProperties` | `maxBackOffInterval` | `Duration.ofSeconds(30)` | `ContainerProperties.java:12` |

### The Properties Class

```java
package dev.linhvu.zalobot.autoconfigure;

import java.time.Duration;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("zalobot")
public class ZaloBotProperties {

    /**
     * Bot token for authenticating with the Zalo API. Required.
     */
    private String botToken;

    private final Client client = new Client();
    private final Listener listener = new Listener();

    public String getBotToken() { return this.botToken; }
    public void setBotToken(String botToken) { this.botToken = botToken; }
    public Client getClient() { return this.client; }
    public Listener getListener() { return this.listener; }

    public static class Client {
        private String scheme = "https";
        private String host = "bot-api.zaloplatforms.com";
        private int port = 443;

        // getters + setters
        public String getScheme() { return this.scheme; }
        public void setScheme(String scheme) { this.scheme = scheme; }
        public String getHost() { return this.host; }
        public void setHost(String host) { this.host = host; }
        public int getPort() { return this.port; }
        public void setPort(int port) { this.port = port; }
    }

    public static class Listener {
        /**
         * Whether the listener container auto-starts.
         */
        private boolean enabled = true;
        private Duration pollTimeout = Duration.ofSeconds(30);
        private Duration pollInterval = Duration.ofSeconds(0);
        private Duration shutdownTimeout = Duration.ofSeconds(10);
        private Duration backOffInterval = Duration.ofSeconds(1);
        private Duration maxBackOffInterval = Duration.ofSeconds(30);
        /**
         * Number of concurrent listener containers. 1 = single, >1 = concurrent.
         */
        private int concurrency = 1;

        // getters + setters for all fields
        public boolean isEnabled() { return this.enabled; }
        public void setEnabled(boolean enabled) { this.enabled = enabled; }
        public Duration getPollTimeout() { return this.pollTimeout; }
        public void setPollTimeout(Duration pollTimeout) { this.pollTimeout = pollTimeout; }
        public Duration getPollInterval() { return this.pollInterval; }
        public void setPollInterval(Duration pollInterval) { this.pollInterval = pollInterval; }
        public Duration getShutdownTimeout() { return this.shutdownTimeout; }
        public void setShutdownTimeout(Duration shutdownTimeout) { this.shutdownTimeout = shutdownTimeout; }
        public Duration getBackOffInterval() { return this.backOffInterval; }
        public void setBackOffInterval(Duration backOffInterval) { this.backOffInterval = backOffInterval; }
        public Duration getMaxBackOffInterval() { return this.maxBackOffInterval; }
        public void setMaxBackOffInterval(Duration maxBackOffInterval) { this.maxBackOffInterval = maxBackOffInterval; }
        public int getConcurrency() { return this.concurrency; }
        public void setConcurrency(int concurrency) { this.concurrency = concurrency; }
    }
}
```

### Matching `application.yml`

```yaml
zalobot:
  bot-token: ${ZALO_BOT_TOKEN}       # required — no default
  client:
    scheme: https                      # default: "https"
    host: bot-api.zaloplatforms.com    # default: "bot-api.zaloplatforms.com"
    port: 443                          # default: 443
  listener:
    enabled: true                      # default: true
    poll-timeout: 30s                  # default: 30s
    poll-interval: 0s                  # default: 0s
    shutdown-timeout: 10s              # default: 10s
    back-off-interval: 1s             # default: 1s
    max-back-off-interval: 30s        # default: 30s
    concurrency: 1                    # default: 1
```

---

## How Property Binding Works (Simplified)

```
application.yml                   @ConfigurationProperties("zalobot")
─────────────────                 ───────────────────────────────────
zalobot:
  bot-token: "abc"          →     ZaloBotProperties.setBotToken("abc")
  client:
    host: "custom.api.com"  →     ZaloBotProperties.getClient().setHost("custom.api.com")
  listener:
    poll-timeout: 60s       →     ZaloBotProperties.getListener().setPollTimeout(Duration.ofSeconds(60))
                                  ↑
                                  Spring Boot auto-converts "60s" → Duration.ofSeconds(60)
                                  Also supports: "500ms", "5m", "1h", "P2D" (ISO-8601)
```

**Key insight:** The nested structure of `Client` and `Listener` static inner classes maps directly to the YAML hierarchy. Spring Boot uses relaxed binding, so `poll-timeout` (kebab-case) maps to `setPollTimeout` (camelCase).

---

## Before and After

**Before (Chapter 1):**
```java
props.setPollTimeout(Duration.ofSeconds(30));  // hardcoded in Java
```

**After (Chapter 2):**
```yaml
zalobot.listener.poll-timeout: 60s  # externalized in YAML, with IDE autocomplete
```

---

## What's Next?

We have externalized configuration, but who creates the beans? In Chapter 3, we learn about `@AutoConfiguration` — the annotation that marks a class as an auto-configuration provider.

```
Chapter 1  ──► "Why do we need starters?"
Chapter 2  ←── YOU ARE HERE: "Externalize config"
Chapter 3  ──► "Who creates beans? @AutoConfiguration"
```
