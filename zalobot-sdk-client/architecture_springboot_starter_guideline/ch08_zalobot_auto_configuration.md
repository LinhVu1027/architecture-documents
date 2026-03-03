# Chapter 8: Building the ZaloBot Auto-Configuration

> The heart of the starter: auto-configuration classes that create beans from properties.

---

## Step 0: Prerequisite — Make the SDK Builder Accessible

The SDK's `DefaultZaloBotClientBuilder` is currently `final class` (package-private). The auto-configure module lives in a different package, so it can't instantiate the builder.

**Change in `DefaultZaloBotClientBuilder.java` (line 10):**

```java
// Before:
final class DefaultZaloBotClientBuilder implements ZaloBotClient.Builder {

// After:
public final class DefaultZaloBotClientBuilder implements ZaloBotClient.Builder {
```

**Add factory method in `ZaloBotClient.java` (after line 28):**

```java
static Builder builder() {
    return new DefaultZaloBotClientBuilder();
}
```

This follows the pattern of `RestClient.builder()` in Spring Boot — a static factory that hides the implementation class.

---

## File 1: `ZaloBotClientCustomizer.java`

A functional interface that allows users to customize the client builder without replacing it entirely. This is the same pattern used by `RestClientCustomizer`, `WebClientCustomizer`, etc. in Spring Boot.

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;

/**
 * Callback interface for customizing the {@link ZaloBotClient.Builder}.
 * Beans of this type are applied to the auto-configured builder
 * before it builds the client.
 *
 * <p>Example:
 * <pre>{@code
 * @Bean
 * ZaloBotClientCustomizer customJsonMapper() {
 *     return builder -> builder.jsonMapper(JsonMapper.builder().build());
 * }
 * }</pre>
 */
@FunctionalInterface
public interface ZaloBotClientCustomizer {

    void customize(ZaloBotClient.Builder builder);
}
```

---

## File 2: `ZaloBotClientAutoConfiguration.java`

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.client.ZaloBotUrl;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Scope;

@AutoConfiguration
@ConditionalOnClass(ZaloBotClient.class)
@ConditionalOnProperty(name = "zalobot.bot-token")
@EnableConfigurationProperties(ZaloBotProperties.class)
public class ZaloBotClientAutoConfiguration {

    private final ZaloBotProperties properties;

    ZaloBotClientAutoConfiguration(ZaloBotProperties properties) {
        this.properties = properties;
    }

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    @ConditionalOnMissingBean
    ZaloBotClient.Builder zaloBotClientBuilder(
            ObjectProvider<ZaloBotClientCustomizer> customizerProvider) {
        ZaloBotProperties.Client clientProps = this.properties.getClient();
        ZaloBotUrl url = new ZaloBotUrl(
                clientProps.getScheme(), clientProps.getHost(), clientProps.getPort());

        ZaloBotClient.Builder builder = ZaloBotClient.builder()
                .botToken(this.properties.getBotToken())
                .ZaloBotUrl(url);

        customizerProvider.orderedStream()
                .forEach(customizer -> customizer.customize(builder));
        return builder;
    }

    @Bean
    @ConditionalOnMissingBean
    ZaloBotClient zaloBotClient(ZaloBotClient.Builder zaloBotClientBuilder) {
        return zaloBotClientBuilder.build();
    }
}
```

### Annotation Walkthrough

| Line | Annotation | Why |
|---|---|---|
| `@AutoConfiguration` | Marks as auto-config class. Implies `@Configuration(proxyBeanMethods=false)` | Standard pattern (`AutoConfiguration.java:58`) |
| `@ConditionalOnClass(ZaloBotClient.class)` | Skip entirely if `zalobot-client` JAR not on classpath | Same as Kafka: `KafkaAutoConfiguration.java:85` |
| `@ConditionalOnProperty(name = "zalobot.bot-token")` | Skip if bot token not configured — prevents NPE | Application won't work without a token |
| `@EnableConfigurationProperties(ZaloBotProperties.class)` | Registers and binds `ZaloBotProperties` | Same as Kafka: `KafkaAutoConfiguration.java:86` |
| `@Scope(SCOPE_PROTOTYPE)` on builder | Each injection gets a fresh builder instance | Builders are stateful; sharing one would be a bug |
| `@ConditionalOnMissingBean` on builder | User can define their own `ZaloBotClient.Builder` bean | Granular back-away |
| `@ConditionalOnMissingBean` on client | User can define their own `ZaloBotClient` bean | Granular back-away |

### The Customizer Pattern

```java
customizerProvider.orderedStream()
        .forEach(customizer -> customizer.customize(builder));
```

Users can define multiple `ZaloBotClientCustomizer` beans:

```java
@Bean
ZaloBotClientCustomizer customJsonMapper() {
    return builder -> builder.jsonMapper(myCustomMapper);
}

@Bean
@Order(1)
ZaloBotClientCustomizer customRequestFactory() {
    return builder -> builder.requestFactory(myOkHttpFactory);
}
```

They are applied in `@Order` sequence via `orderedStream()`.

---

## File 3: `ZaloBotListenerAutoConfiguration.java`

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.listener.ConcurrentUpdateListenerContainer;
import dev.linhvu.zalobot.listener.ContainerProperties;
import dev.linhvu.zalobot.listener.ErrorHandler;
import dev.linhvu.zalobot.listener.UpdateListener;
import dev.linhvu.zalobot.listener.UpdateListenerContainer;
import dev.linhvu.zalobot.listener.ZaloUpdateListenerContainer;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;

@AutoConfiguration(after = ZaloBotClientAutoConfiguration.class)
@ConditionalOnClass(UpdateListenerContainer.class)
@ConditionalOnBean(ZaloBotClient.class)
@ConditionalOnProperty(name = "zalobot.listener.enabled", havingValue = "true", matchIfMissing = true)
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
    @ConditionalOnMissingBean(UpdateListenerContainer.class)
    @ConditionalOnBean(UpdateListener.class)
    UpdateListenerContainer zaloBotListenerContainer(
            ZaloBotClient client, ContainerProperties containerProperties) {
        int concurrency = this.properties.getListener().getConcurrency();
        if (concurrency > 1) {
            ConcurrentUpdateListenerContainer container =
                    new ConcurrentUpdateListenerContainer(client, containerProperties);
            container.setConcurrency(concurrency);
            return container;
        }
        return new ZaloUpdateListenerContainer(client, containerProperties);
    }

    @Bean
    @ConditionalOnBean(UpdateListenerContainer.class)
    ZaloBotListenerContainerLifecycle zaloBotListenerContainerLifecycle(
            UpdateListenerContainer container) {
        return new ZaloBotListenerContainerLifecycle(container);
    }
}
```

### Annotation Walkthrough

| Annotation | Why |
|---|---|
| `@AutoConfiguration(after = ZaloBotClientAutoConfiguration.class)` | Client beans must exist before listener can use them. This controls Phase 5 sorting (`AutoConfiguration.java:112-113`) |
| `@ConditionalOnClass(UpdateListenerContainer.class)` | Skip if `zalobot-listener` JAR not on classpath. Users might only want the client |
| `@ConditionalOnBean(ZaloBotClient.class)` | Skip if client auto-config didn't create a client (e.g., token missing) |
| `@ConditionalOnProperty("zalobot.listener.enabled", matchIfMissing=true)` | Enabled by default, but users can set `false` to disable |
| `@ConditionalOnBean(UpdateListener.class)` on container | Only create container if user defined an `UpdateListener` bean |
| `@ConditionalOnBean(UpdateListenerContainer.class)` on lifecycle | Only create lifecycle adapter if container was created |

### Concurrency Routing

```java
int concurrency = this.properties.getListener().getConcurrency();
if (concurrency > 1) {
    ConcurrentUpdateListenerContainer container = ...;
    container.setConcurrency(concurrency);
    return container;
}
return new ZaloUpdateListenerContainer(client, containerProperties);
```

- `concurrency = 1` (default) → single `ZaloUpdateListenerContainer`
- `concurrency > 1` → `ConcurrentUpdateListenerContainer` that manages N child containers

---

## File 4: `ZaloBotListenerContainerLifecycle.java`

The SDK's `UpdateListenerContainer` doesn't implement Spring's `SmartLifecycle`. Without this adapter, the container wouldn't auto-start when the application context starts, or auto-stop when it shuts down.

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.listener.UpdateListenerContainer;
import org.springframework.context.SmartLifecycle;

/**
 * Adapts an {@link UpdateListenerContainer} to Spring's {@link SmartLifecycle},
 * so the container auto-starts with the application context and auto-stops on shutdown.
 */
class ZaloBotListenerContainerLifecycle implements SmartLifecycle {

    private final UpdateListenerContainer container;

    ZaloBotListenerContainerLifecycle(UpdateListenerContainer container) {
        this.container = container;
    }

    @Override
    public void start() {
        this.container.start();
    }

    @Override
    public void stop() {
        this.container.stop();
    }

    @Override
    public boolean isRunning() {
        return this.container.isRunning();
    }

    @Override
    public boolean isAutoStartup() {
        return true;    // auto-start with the context
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE;    // start last, stop first
    }
}
```

This is the **Adapter** pattern: wrapping an existing interface (`UpdateListenerContainer`) to work with another interface (`SmartLifecycle`) that the framework expects.

---

## File 5: The `.imports` Registration File

**File:** `spring-boot-zalobot/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
dev.linhvu.zalobot.boot.autoconfigure.ZaloBotClientAutoConfiguration
dev.linhvu.zalobot.autoconfigure.ZaloBotListenerAutoConfiguration
```

Without this file, Spring Boot cannot discover the auto-configuration classes (Chapter 5).

---

## Bean Dependency Graph

```
@EnableConfigurationProperties
         │
         ▼
┌─────────────────────────────────────┐
│ ZaloBotProperties                    │ ← bound from application.yml
└─────────────┬───────────────────────┘
              │
              ├─────────────────────────────────────────────────┐
              ▼                                                 ▼
┌──────────────────────────┐                 ┌──────────────────────────────────┐
│ ZaloBotClientAutoConfig  │                 │ ZaloBotListenerAutoConfig        │
│                          │                 │   (after = ClientAutoConfig)     │
│  ┌────────────────────┐  │                 │                                  │
│  │ ZaloBotClient      │  │────────────────►│  ┌──────────────────────────┐    │
│  │   .Builder          │  │                 │  │ ContainerProperties      │    │
│  │   (PROTOTYPE)       │  │                 │  │   ├── UpdateListener     │    │
│  └────────┬───────────┘  │                 │  │   └── ErrorHandler       │    │
│           │               │                 │  └──────────┬───────────────┘    │
│           ▼               │                 │             │                    │
│  ┌────────────────────┐  │                 │             ▼                    │
│  │ ZaloBotClient       │──┼─────────────────│  ┌──────────────────────────┐    │
│  └────────────────────┘  │                 │  │ UpdateListenerContainer   │    │
│                          │                 │  │  (Single or Concurrent)   │    │
└──────────────────────────┘                 │  └──────────┬───────────────┘    │
                                             │             │                    │
User provides:                               │             ▼                    │
  ┌──────────────────┐                       │  ┌──────────────────────────┐    │
  │ UpdateListener    │──────────────────────►│  │ ZaloBotListenerContainer │    │
  │ (user @Bean)      │                       │  │   Lifecycle (SmartLife.) │    │
  └──────────────────┘                       │  └──────────────────────────┘    │
  ┌──────────────────┐                       │                                  │
  │ ErrorHandler      │──────────────────────►│                                  │
  │ (optional @Bean)  │                       └──────────────────────────────────┘
  └──────────────────┘
  ┌──────────────────┐
  │ ZaloBotClient    │─── user override ───► replaces auto-configured client
  │   Customizer(s)  │                       (applied via ObjectProvider)
  └──────────────────┘
```

---

## What's Next?

The auto-configuration code is complete. Chapter 9 assembles the Maven module structure — the starter POM that ties everything together.

```
Chapter 7  ──► "Build it: properties"
Chapter 8  ←── YOU ARE HERE: "Build it: auto-config"
Chapter 9  ──► "Build it: POM/dependencies"
```
