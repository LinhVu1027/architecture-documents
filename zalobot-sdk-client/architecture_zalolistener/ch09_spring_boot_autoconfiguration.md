# Chapter 9: Spring Boot Auto-Configuration — Making It All Work with Zero Config

## What You'll Learn
- How to auto-configure `@EnableZaloBot` and the default container factory
- How to support BOTH the annotation path and the legacy `UpdateListener` bean path
- How the auto-configuration classes are updated to work with the new infrastructure
- The complete end-to-end developer experience

---

## 9.1 The Goal: Zero-Configuration @ZaloListener

After all the infrastructure from Chapters 2-8, we want this to "just work":

```java
@SpringBootApplication
public class MyZaloBotApp {
    public static void main(String[] args) {
        SpringApplication.run(MyZaloBotApp.class, args);
    }
}

@Component
public class MyHandlers {

    @ZaloListener(id = "text-handler")
    public void handleText(GetUpdatesResult update) {
        System.out.println("Got: " + update.message().text());
    }
}
```

With just:
```yaml
zalobot:
  bot-token: "my-bot-token"
```

For this to work, the auto-configuration must:
1. Apply `@EnableZaloBot` automatically
2. Create the default `zaloListenerContainerFactory` bean
3. Continue supporting the legacy `UpdateListener` bean path for backward compatibility

---

## 9.2 New Auto-Configuration: ZaloBotAnnotationAutoConfiguration

A new auto-configuration class that handles the `@ZaloListener` annotation infrastructure:

```java
package dev.linhvu.zalobot.boot.autoconfigure;

import dev.linhvu.zalobot.boot.annotation.EnableZaloBot;
import dev.linhvu.zalobot.boot.config.ConcurrentZaloListenerContainerFactory;
import dev.linhvu.zalobot.boot.config.ZaloListenerContainerFactory;
import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.listener.ContainerProperties;
import dev.linhvu.zalobot.listener.ErrorHandler;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;

/**
 * Auto-configuration for {@code @ZaloListener} annotation support.
 *
 * <p>This configuration:
 * <ul>
 *   <li>Enables {@code @EnableZaloBot} annotation processing</li>
 *   <li>Creates the default {@code zaloListenerContainerFactory} bean</li>
 * </ul>
 *
 * <p>This configuration runs AFTER the client auto-configuration to ensure
 * {@link ZaloBotClient} is available.
 *
 * @author Linh Vu
 * @since 0.1.0
 */
@AutoConfiguration(after = ZaloBotClientAutoConfiguration.class)
@ConditionalOnClass(ZaloBotClient.class)
@ConditionalOnBean(ZaloBotClient.class)
@ConditionalOnProperty(name = "zalobot.listener.enabled", havingValue = "true",
        matchIfMissing = true)
@EnableZaloBot
public class ZaloBotAnnotationAutoConfiguration {

    private final ZaloBotProperties properties;

    ZaloBotAnnotationAutoConfiguration(ZaloBotProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean(name = "zaloListenerContainerFactory")
    ConcurrentZaloListenerContainerFactory zaloListenerContainerFactory(
            ZaloBotClient client,
            ObjectProvider<ErrorHandler> errorHandlerProvider) {

        ZaloBotProperties.Listener listenerProps = this.properties.getListener();

        ContainerProperties containerProperties = new ContainerProperties();
        containerProperties.setPollTimeout(listenerProps.getPollTimeout());
        containerProperties.setPollInterval(listenerProps.getPollInterval());
        containerProperties.setShutdownTimeout(listenerProps.getShutdownTimeout());
        containerProperties.setBackOffInterval(listenerProps.getBackOffInterval());
        containerProperties.setMaxBackOffInterval(listenerProps.getMaxBackOffInterval());
        errorHandlerProvider.ifAvailable(containerProperties::setErrorHandler);

        ConcurrentZaloListenerContainerFactory factory =
                new ConcurrentZaloListenerContainerFactory();
        factory.setClient(client);
        factory.setContainerProperties(containerProperties);
        factory.setConcurrency(listenerProps.getConcurrency());

        return factory;
    }
}
```

---

## 9.3 Key Auto-Configuration Decisions

### @EnableZaloBot on the Auto-Configuration Class

By annotating the auto-configuration class itself with `@EnableZaloBot`, we ensure the BPP and registry are registered automatically. The user doesn't need to add `@EnableZaloBot` to their own config — it's done for them.

This mirrors Spring Kafka's approach in Spring Boot:

```java
// Spring Boot's KafkaAnnotationDrivenConfiguration (simplified)
@Configuration
@ConditionalOnClass(EnableKafka.class)
@EnableKafka
class KafkaAnnotationDrivenConfiguration {
    // ...
}
```

### @ConditionalOnMissingBean(name = "zaloListenerContainerFactory")

Users can define their own factory bean with the same name to override the default:

```java
@Bean
ConcurrentZaloListenerContainerFactory zaloListenerContainerFactory(
        ZaloBotClient client) {
    // Custom factory with different settings
    ConcurrentZaloListenerContainerFactory factory = new ConcurrentZaloListenerContainerFactory();
    factory.setClient(client);
    // ...custom config...
    return factory;
}
```

---

## 9.4 Updated ZaloBotListenerAutoConfiguration

The existing `ZaloBotListenerAutoConfiguration` continues to support the legacy path (manual `UpdateListener` bean). It now coexists with the new annotation-driven path:

```java
package dev.linhvu.zalobot.boot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.listener.ConcurrentUpdateListenerContainer;
import dev.linhvu.zalobot.listener.ContainerProperties;
import dev.linhvu.zalobot.listener.ErrorHandler;
import dev.linhvu.zalobot.listener.UpdateListener;
import dev.linhvu.zalobot.listener.UpdateListenerContainer;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;

/**
 * Auto-configuration for the legacy (non-annotation) listener path.
 *
 * <p>This configuration creates a single listener container when the user
 * defines an {@link UpdateListener} bean directly (without using @ZaloListener).
 *
 * <p>This configuration is NOT needed when using @ZaloListener, because the
 * annotation infrastructure creates containers automatically.
 *
 * @author Linh Vu
 * @since 0.0.1
 */
@AutoConfiguration(after = ZaloBotClientAutoConfiguration.class)
@ConditionalOnClass(UpdateListenerContainer.class)
@ConditionalOnBean(ZaloBotClient.class)
@ConditionalOnProperty(name = "zalobot.listener.enabled", havingValue = "true",
        matchIfMissing = true)
public class ZaloBotListenerAutoConfiguration {

    private final ZaloBotProperties properties;

    ZaloBotListenerAutoConfiguration(ZaloBotProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean
    ContainerProperties zaloBotContainerProperties(
            ObjectProvider<UpdateListener> updateListenerProvider,
            ObjectProvider<ErrorHandler> errorHandlerProvider) {

        ZaloBotProperties.Listener listenerProps = this.properties.getListener();

        ContainerProperties cp = new ContainerProperties();
        cp.setPollTimeout(listenerProps.getPollTimeout());
        cp.setPollInterval(listenerProps.getPollInterval());
        cp.setShutdownTimeout(listenerProps.getShutdownTimeout());
        cp.setBackOffInterval(listenerProps.getBackOffInterval());
        cp.setMaxBackOffInterval(listenerProps.getMaxBackOffInterval());
        updateListenerProvider.ifAvailable(cp::setUpdateListener);
        errorHandlerProvider.ifAvailable(cp::setErrorHandler);
        return cp;
    }

    @Bean
    @ConditionalOnBean(UpdateListener.class)
    @ConditionalOnMissingBean(UpdateListenerContainer.class)
    UpdateListenerContainer zaloBotListenerContainer(
            ZaloBotClient client, ContainerProperties containerProperties) {

        int concurrency = this.properties.getListener().getConcurrency();
        ConcurrentUpdateListenerContainer container =
                new ConcurrentUpdateListenerContainer(client, containerProperties);
        container.setConcurrency(concurrency);
        return container;
    }

    @Bean
    @ConditionalOnBean(UpdateListenerContainer.class)
    ZaloBotListenerContainerLifecycle zaloBotListenerContainerLifecycle(
            UpdateListenerContainer container) {
        return new ZaloBotListenerContainerLifecycle(container);
    }
}
```

### How the Two Paths Coexist

```
Path A: @ZaloListener annotation (NEW)
─────────────────────────────────────
@ZaloListener method found
    → ZaloBotAnnotationAutoConfiguration provides:
      - @EnableZaloBot (BPP + Registry)
      - zaloListenerContainerFactory bean
    → BPP creates containers via factory
    → Registry manages lifecycle via SmartLifecycle

Path B: Manual UpdateListener bean (EXISTING)
──────────────────────────────────────────────
@Bean UpdateListener defined by user
    → ZaloBotListenerAutoConfiguration provides:
      - ContainerProperties bean
      - UpdateListenerContainer bean (@ConditionalOnBean(UpdateListener.class))
      - ZaloBotListenerContainerLifecycle bean
    → Single container managed by lifecycle bean

Path A + B: Both coexist
────────────────────────
If user defines BOTH @ZaloListener methods AND an UpdateListener bean:
    → Annotation path creates containers for @ZaloListener methods
    → Legacy path creates one container for the UpdateListener bean
    → Both run independently
```

> ★ **Insight** ─────────────────────────────────────
> - **Backward compatibility without compromise.** The existing `ZaloBotListenerAutoConfiguration` is unchanged. Users who don't use `@ZaloListener` see zero difference. Users who adopt `@ZaloListener` get the new infrastructure automatically. This is the "strangler fig" migration pattern — the new system grows around the old one without breaking it.
> - **Why a SEPARATE auto-configuration class?** We could modify `ZaloBotListenerAutoConfiguration` to add the factory bean. But keeping them separate follows the Single Responsibility Principle: one class handles the annotation infrastructure, another handles the legacy path. It also makes it easy to remove the legacy path in a future major version.
> ─────────────────────────────────────────────────────

---

## 9.5 Registering the New Auto-Configuration

Spring Boot auto-configuration classes are registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
dev.linhvu.zalobot.boot.autoconfigure.ZaloBotClientAutoConfiguration
dev.linhvu.zalobot.boot.autoconfigure.ZaloBotAnnotationAutoConfiguration
dev.linhvu.zalobot.boot.autoconfigure.ZaloBotListenerAutoConfiguration
```

The ordering (`after = ZaloBotClientAutoConfiguration.class` on the annotation) ensures the client is configured first.

---

## 9.6 The Complete End-to-End Example

### application.yaml
```yaml
zalobot:
  bot-token: "${ZALO_BOT_TOKEN}"
  listener:
    concurrency: 2
    poll-timeout: 30s
```

### Application class
```java
@SpringBootApplication
public class MyZaloBotApp {
    public static void main(String[] args) {
        SpringApplication.run(MyZaloBotApp.class, args);
    }
}
```

### Handler class
```java
@Component
public class MyZaloBotHandlers {

    @ZaloListener(id = "text-handler", concurrency = "3")
    public void handleTextMessage(GetUpdatesResult update) {
        if (update.isTextMessage()) {
            System.out.println("Text from "
                + update.message().from().displayName()
                + ": " + update.message().text());
        }
    }

    @ZaloListener(id = "image-handler")
    public void handleImageMessage(GetUpdatesResult.Message message) {
        System.out.println("Image message from: " + message.from().displayName());
    }
}
```

### What Happens at Startup

```
1. Spring Boot starts, reads application.yaml
   zalobot.bot-token = "abc123"
   zalobot.listener.concurrency = 2

2. ZaloBotClientAutoConfiguration creates ZaloBotClient

3. ZaloBotAnnotationAutoConfiguration:
   a. @EnableZaloBot → ZaloBootstrapConfiguration
      → registers BPP and Registry bean definitions
   b. Creates zaloListenerContainerFactory bean:
      - client = auto-configured ZaloBotClient
      - containerProperties = from zalobot.listener.* props
      - concurrency = 2 (default from properties)

4. BPP processes MyZaloBotHandlers bean:
   a. Finds handleTextMessage() with @ZaloListener(id="text-handler", concurrency="3")
      → endpoint1: id="text-handler", concurrency=3, bean=myHandlers, method=handleTextMessage
      → registrar.registerEndpoint(endpoint1, null)

   b. Finds handleImageMessage() with @ZaloListener(id="image-handler")
      → endpoint2: id="image-handler", concurrency=null, bean=myHandlers, method=handleImageMessage
      → registrar.registerEndpoint(endpoint2, null)

5. All singletons created → BPP.afterSingletonsInstantiated()
   → registrar.afterPropertiesSet()
   → registrar.registerAllEndpoints()

   a. For endpoint1:
      factory = getBean("zaloListenerContainerFactory")
      container1 = factory.createListenerContainer(endpoint1)
        → ConcurrentUpdateListenerContainer(client, properties)
        → container1.setConcurrency(3)     ← endpoint override
        → endpoint1.setupListenerContainer(container1)
          → adapter1 = new UpdateListenerAdapter(myHandlers, handleTextMessage)
          → container1.setUpdateListener(adapter1)
      registry stores container1 as "text-handler"

   b. For endpoint2:
      container2 = factory.createListenerContainer(endpoint2)
        → ConcurrentUpdateListenerContainer(client, properties)
        → container2.setConcurrency(2)     ← factory default (no endpoint override)
        → endpoint2.setupListenerContainer(container2)
          → adapter2 = new UpdateListenerAdapter(myHandlers, handleImageMessage)
          → container2.setUpdateListener(adapter2)
      registry stores container2 as "image-handler"

6. Context refresh → SmartLifecycle → Registry.start()
   → container1.start() → 3 ListenerConsumer threads polling Zalo API
   → container2.start() → 2 ListenerConsumer threads polling Zalo API

7. When a text message arrives on container1:
   → adapter1.onUpdate(update)
   → resolves argument: GetUpdatesResult → pass directly
   → myHandlers.handleTextMessage(update)
   → prints "Text from John: Hello!"

8. When an image message arrives on container2:
   → adapter2.onUpdate(update)
   → resolves argument: GetUpdatesResult.Message → update.message()
   → myHandlers.handleImageMessage(message)
   → prints "Image message from: John"
```

---

## 9.7 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `ZaloBotAnnotationAutoConfiguration` | `KafkaAnnotationDrivenConfiguration` | Spring Boot's `spring-boot-autoconfigure` module |
| `@EnableZaloBot` on auto-config class | `@EnableKafka` on annotation-driven config | Same pattern |
| `zaloListenerContainerFactory` bean | `kafkaListenerContainerFactory` bean | Spring Boot auto-config |
| `ZaloBotListenerAutoConfiguration` (legacy) | N/A (Kafka never had a legacy path) | Unique to ZaloBot |
| `AutoConfiguration.imports` registration | `AutoConfiguration.imports` registration | Same mechanism |
| `@ConditionalOnMissingBean` factory | `@ConditionalOnMissingBean` factory | Same conditional |

> ★ **Insight** ─────────────────────────────────────
> - **Auto-configuration ordering is critical.** `ZaloBotAnnotationAutoConfiguration` must run after `ZaloBotClientAutoConfiguration` (to have the client available) but the annotation `@AutoConfiguration(after = ...)` only suggests ordering — Spring Boot's auto-configuration ordering is best-effort. In practice, `@ConditionalOnBean(ZaloBotClient.class)` acts as the real gate: if the client isn't ready, the condition fails and the auto-configuration is skipped. Spring Kafka handles this the same way — conditions are the real mechanism, ordering is a hint.
> ─────────────────────────────────────────────────────

---

## 9.8 Full File List: All New Classes

```
zalobot-spring-boot/src/main/java/dev/linhvu/zalobot/boot/
├── annotation/
│   ├── ZaloListener.java                                    [Ch02]
│   ├── EnableZaloBot.java                                   [Ch08]
│   ├── ZaloBootstrapConfiguration.java                      [Ch08]
│   └── ZaloListenerAnnotationBeanPostProcessor.java         [Ch07]
├── config/
│   ├── ZaloListenerEndpoint.java                            [Ch04]
│   ├── AbstractZaloListenerEndpoint.java                    [Ch04]
│   ├── MethodZaloListenerEndpoint.java                      [Ch04]
│   ├── ZaloListenerContainerFactory.java                    [Ch05]
│   ├── ConcurrentZaloListenerContainerFactory.java          [Ch05]
│   ├── ZaloListenerEndpointRegistrar.java                   [Ch06]
│   └── ZaloListenerEndpointRegistry.java                    [Ch06]
├── listener/
│   └── adapter/
│       ├── UpdateListenerAdapter.java                       [Ch03]
│       └── UpdateListenerExecutionException.java            [Ch03]
└── autoconfigure/
    ├── ZaloBotClientAutoConfiguration.java                  [existing]
    ├── ZaloBotAnnotationAutoConfiguration.java              [Ch09 - NEW]
    ├── ZaloBotListenerAutoConfiguration.java                [existing - UNCHANGED]
    ├── ZaloBotListenerContainerLifecycle.java               [existing - UNCHANGED]
    └── ZaloBotProperties.java                               [existing - UNCHANGED]

Also update:
├── META-INF/spring/
│   └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
│       → add ZaloBotAnnotationAutoConfiguration
```

**Total new files: 12**
**Modified files: 1** (AutoConfiguration.imports)
**Unchanged files: 4** (existing auto-configuration)

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ZaloBotAnnotationAutoConfiguration** | New auto-config: enables @EnableZaloBot + creates default factory |
| **zaloListenerContainerFactory** | Default factory bean, configured from `zalobot.listener.*` properties |
| **Backward compatibility** | Legacy `UpdateListener` bean path still works alongside @ZaloListener |
| **Zero-config for users** | Just add `@ZaloListener` to methods — auto-config handles everything |
| **@ConditionalOnMissingBean** | Users can override the default factory by defining their own |
| **AutoConfiguration.imports** | Registration file for Spring Boot to discover the new auto-config |

---

## What's Next?

This completes the `@ZaloListener` annotation infrastructure. The complete system:

```
@ZaloListener ──▶ BPP discovers ──▶ Endpoint created ──▶ Registrar collects
                                                               │
                                                               ▼
Container running ◀── Registry starts ◀── Factory creates ◀── Registrar flushes
      │
      ▼
Update arrives ──▶ Adapter invokes ──▶ User's method handles it
```

**Possible future enhancements:**
- Event-type filtering (`@ZaloListener(eventType = "message.text.received")`)
- `@ZaloHandler` for class-level multi-method dispatch
- SpEL expression support in annotation attributes
- Programmatic endpoint registration via `ZaloListenerConfigurer`
- `@SendTo` reply integration
