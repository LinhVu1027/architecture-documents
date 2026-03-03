# Chapter 3: Auto-Configuration Classes and `@AutoConfiguration`

> Who creates the beans? An `@AutoConfiguration` class is a `@Configuration` with superpowers.

---

## The Annotation

```java
// AutoConfiguration.java:55-61
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration(proxyBeanMethods = false)    // ← KEY: always false for performance
@AutoConfigureBefore                         // ← ordering support
@AutoConfigureAfter                          // ← ordering support
public @interface AutoConfiguration {

    @AliasFor(annotation = Configuration.class)
    String value() default "";

    @AliasFor(annotation = AutoConfigureBefore.class, attribute = "value")
    Class<?>[] before() default {};

    @AliasFor(annotation = AutoConfigureAfter.class, attribute = "value")
    Class<?>[] after() default {};

    // ... beforeName, afterName for string-based references
}
```

**Source:** `core/spring-boot-autoconfigure/.../AutoConfiguration.java:55-125`

### Key Insight: `proxyBeanMethods = false`

At line 58, `@AutoConfiguration` is meta-annotated with `@Configuration(proxyBeanMethods = false)`. This is always `false` for auto-configuration classes. Here's why:

With `proxyBeanMethods = true` (the default for regular `@Configuration`):
```java
@Configuration
public class MyConfig {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());  // ← calls dataSource() again
        // With proxy: returns SAME singleton instance (intercepted by CGLIB)
        // Without proxy: creates SECOND DataSource (bug!)
    }
}
```

With `proxyBeanMethods = false`:
- No CGLIB subclass is created → faster startup, less memory
- You must NOT call one `@Bean` method from another
- Instead, inject via method parameters:

```java
@AutoConfiguration  // implies proxyBeanMethods = false
public class MyAutoConfig {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {  // ← injected
        return new JdbcTemplate(dataSource);  // ← same singleton, no proxy needed
    }
}
```

Auto-configuration classes follow this pattern consistently. The `@AutoConfiguration` annotation enforces it.

---

## Real Example: KafkaAutoConfiguration

```java
// KafkaAutoConfiguration.java:84-95
@AutoConfiguration
@ConditionalOnClass(KafkaTemplate.class)
@EnableConfigurationProperties(KafkaProperties.class)
@Import({ KafkaAnnotationDrivenConfiguration.class,
          KafkaStreamsAnnotationDrivenConfiguration.class })
@ImportRuntimeHints(KafkaAutoConfiguration.KafkaRuntimeHints.class)
public final class KafkaAutoConfiguration {

    private final KafkaProperties properties;

    KafkaAutoConfiguration(KafkaProperties properties) {    // ← constructor injection
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean(KafkaTemplate.class)
    KafkaTemplate<?, ?> kafkaTemplate(
            ProducerFactory<Object, Object> kafkaProducerFactory, ...) {
        // uses this.properties + injected beans
    }

    @Bean
    @ConditionalOnMissingBean(ConsumerFactory.class)
    DefaultKafkaConsumerFactory<?, ?> kafkaConsumerFactory(...) { ... }

    @Bean
    @ConditionalOnMissingBean(ProducerFactory.class)
    DefaultKafkaProducerFactory<?, ?> kafkaProducerFactory(...) { ... }
    // ... more @Bean methods
}
```

**Source:** `module/spring-boot-kafka/.../KafkaAutoConfiguration.java:84-95`

Notice the patterns:
- Constructor injection of `KafkaProperties` (not field injection)
- Each `@Bean` method receives its dependencies as parameters (not calling other `@Bean` methods)
- Class is `final` (no subclassing expected)
- Package-private constructor and `@Bean` methods (Spring can access them, but they're not part of the public API)

---

## Building: Basic ZaloBotClientAutoConfiguration (No Conditionals Yet)

Let's write our first auto-configuration class. In this chapter, we skip conditional annotations — we'll add them in Chapter 4.

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.client.ZaloBotUrl;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration                                         // ← marks this as auto-config
@EnableConfigurationProperties(ZaloBotProperties.class)    // ← registers our properties
public class ZaloBotClientAutoConfiguration {

    private final ZaloBotProperties properties;

    ZaloBotClientAutoConfiguration(ZaloBotProperties properties) {
        this.properties = properties;
    }

    @Bean
    ZaloBotClient.Builder zaloBotClientBuilder() {
        ZaloBotProperties.Client clientProps = this.properties.getClient();
        ZaloBotUrl url = new ZaloBotUrl(
                clientProps.getScheme(), clientProps.getHost(), clientProps.getPort());

        return ZaloBotClient.builder()
                .botToken(this.properties.getBotToken())
                .ZaloBotUrl(url);
    }

    @Bean
    ZaloBotClient zaloBotClient(ZaloBotClient.Builder builder) {   // ← parameter injection
        return builder.build();
    }
}
```

### Before and After

**Before (Chapter 1 — manual):**
```java
@Configuration
public class ZaloBotConfig {
    @Bean
    public ZaloBotClient zaloBotClient() {
        return new DefaultZaloBotClientBuilder()       // package-private!
                .botToken("my-secret-token")           // hardcoded!
                .ZaloBotUrl(new ZaloBotUrl("https", "bot-api.zaloplatforms.com", 443))
                .build();
    }
}
```

**After (Chapter 3 — auto-config, no conditionals yet):**
```java
@AutoConfiguration
@EnableConfigurationProperties(ZaloBotProperties.class)
public class ZaloBotClientAutoConfiguration {
    // Properties from YAML, not hardcoded
    // Builder pattern, not direct constructor call
    // Dependency injection, not method chaining across beans
}
```

---

## What This Doesn't Handle Yet

Our Chapter 3 version has problems:
1. **Always runs** — even when `ZaloBotClient.class` isn't on the classpath
2. **Always runs** — even when `zalobot.bot-token` isn't set (NPE at runtime)
3. **No back-away** — if the user defines their own `ZaloBotClient`, both exist (conflict)

We solve all three in Chapter 4 with conditional annotations.

---

## What's Next?

```
Chapter 1  ──► "Why do we need starters?"
Chapter 2  ──► "Externalize config"
Chapter 3  ←── YOU ARE HERE: "Who creates beans? @AutoConfiguration"
Chapter 4  ──► "When to create? Conditional annotations"
```
