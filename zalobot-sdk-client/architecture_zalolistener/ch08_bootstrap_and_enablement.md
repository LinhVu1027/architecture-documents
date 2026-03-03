# Chapter 8: Bootstrap & Enablement — @EnableZaloBot and Wiring the Infrastructure

## What You'll Learn
- How `@EnableZaloBot` triggers the entire annotation infrastructure
- How `ZaloBootstrapConfiguration` registers the BPP and registry as bean definitions
- Why bean definitions (not `@Bean` methods) are used for infrastructure beans
- The chicken-and-egg problem with BeanPostProcessors

---

## 8.1 The Bootstrap Problem

We have all the pieces:
- **BPP** (Ch07) — scans for `@ZaloListener`
- **Registrar** (Ch06) — collects endpoints
- **Registry** (Ch06) — manages containers
- **Factory** (Ch05) — creates containers
- **Endpoint** (Ch04) — bundles method + config
- **Adapter** (Ch03) — bridges method to `UpdateListener`

But who creates the BPP and registry beans? They can't be created by a BPP (that's circular). They need to be registered very early in the Spring context lifecycle — before any regular `@Bean` methods run.

The answer: an `ImportBeanDefinitionRegistrar` imported by `@EnableZaloBot`.

```
@EnableZaloBot
    │
    └── @Import(ZaloBootstrapConfiguration.class)
            │
            └── ZaloBootstrapConfiguration implements ImportBeanDefinitionRegistrar
                    │
                    ├── registers BPP bean definition (before any beans are created)
                    └── registers Registry bean definition (before any beans are created)
```

---

## 8.2 @EnableZaloBot Annotation

```java
package dev.linhvu.zalobot.boot.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.context.annotation.Import;

/**
 * Enable ZaloBot listener annotation processing.
 *
 * <p>When added to a {@code @Configuration} class, this annotation triggers
 * the registration of a {@link ZaloListenerAnnotationBeanPostProcessor}
 * that scans for {@link ZaloListener @ZaloListener} on beans.
 *
 * <p>Usage:
 * <pre>{@code
 * @Configuration
 * @EnableZaloBot
 * public class MyConfig {
 *     // @ZaloListener methods will be discovered automatically
 * }
 * }</pre>
 *
 * <p>When using Spring Boot auto-configuration, this annotation may be
 * applied automatically. See
 * {@link dev.linhvu.zalobot.boot.autoconfigure.ZaloBotListenerAutoConfiguration}.
 *
 * @author Linh Vu
 * @since 0.1.0
 * @see ZaloListener
 * @see ZaloBootstrapConfiguration
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ZaloBootstrapConfiguration.class)
public @interface EnableZaloBot {
}
```

**That's it.** The annotation itself does nothing except `@Import` the bootstrap configuration. This is the standard Spring "Enable" pattern:
- `@EnableKafka` → `@Import(KafkaBootstrapConfiguration.class)`
- `@EnableScheduling` → `@Import(SchedulingConfiguration.class)`
- `@EnableAsync` → `@Import(AsyncConfigurationSelector.class)`

---

## 8.3 ZaloBootstrapConfiguration

```java
package dev.linhvu.zalobot.boot.annotation;

import dev.linhvu.zalobot.boot.config.ZaloListenerEndpointRegistry;

import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.context.annotation.ImportBeanDefinitionRegistrar;
import org.springframework.core.type.AnnotationMetadata;

/**
 * Registers the infrastructure beans needed for {@code @ZaloListener}
 * annotation processing.
 *
 * <p>This registrar is imported by {@link EnableZaloBot @EnableZaloBot}
 * and registers:
 * <ul>
 *   <li>{@link ZaloListenerAnnotationBeanPostProcessor} — scans beans for @ZaloListener</li>
 *   <li>{@link ZaloListenerEndpointRegistry} — manages listener container lifecycles</li>
 * </ul>
 *
 * @author Linh Vu
 * @since 0.1.0
 */
public class ZaloBootstrapConfiguration implements ImportBeanDefinitionRegistrar {

    /**
     * Well-known bean name for the BPP.
     */
    static final String ZALO_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME =
            "dev.linhvu.zalobot.boot.annotation"
                    + ".internalZaloListenerAnnotationProcessor";

    /**
     * Well-known bean name for the endpoint registry.
     */
    static final String ZALO_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME =
            "dev.linhvu.zalobot.boot.config"
                    + ".internalZaloListenerEndpointRegistry";

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {

        // Register the BPP
        if (!registry.containsBeanDefinition(
                ZALO_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)) {

            RootBeanDefinition bppDef = new RootBeanDefinition(
                    ZaloListenerAnnotationBeanPostProcessor.class);
            bppDef.setRole(RootBeanDefinition.ROLE_INFRASTRUCTURE);
            registry.registerBeanDefinition(
                    ZALO_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME, bppDef);
        }

        // Register the endpoint registry
        if (!registry.containsBeanDefinition(
                ZALO_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME)) {

            RootBeanDefinition registryDef = new RootBeanDefinition(
                    ZaloListenerEndpointRegistry.class);
            registryDef.setRole(RootBeanDefinition.ROLE_INFRASTRUCTURE);
            registry.registerBeanDefinition(
                    ZALO_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME, registryDef);
        }
    }
}
```

---

## 8.4 Why Bean Definitions, Not @Bean Methods?

You might wonder: why not just use `@Bean` methods?

```java
// ❌ This would NOT work correctly
@Configuration
public class ZaloBootstrapConfiguration {

    @Bean
    ZaloListenerAnnotationBeanPostProcessor zaloListenerBpp() {
        return new ZaloListenerAnnotationBeanPostProcessor();
    }
}
```

**Problem:** `BeanPostProcessor` beans have a special lifecycle in Spring. They MUST be registered as bean definitions before any regular beans are created. If you use a `@Bean` method, the configuration class itself needs to be processed first, which means other `@Bean` methods in it (and its dependencies) are created BEFORE the BPP exists — missing those beans for annotation scanning.

Using `ImportBeanDefinitionRegistrar.registerBeanDefinitions()` registers the BPP **at the bean definition level**, before any beans are instantiated. This guarantees the BPP processes ALL beans.

> ★ **Insight** ─────────────────────────────────────
> - **`ROLE_INFRASTRUCTURE`** marks these beans as internal framework beans, not user-facing. Spring uses this to:
>   1. Exclude them from component scanning summaries
>   2. Allow them to be overridden by user beans without warning
>   3. Identify them in debugging output as infrastructure
>   Spring Kafka's `KafkaBootstrapConfiguration` also sets `ROLE_INFRASTRUCTURE` on both the BPP and registry.
> - **The `containsBeanDefinition` guard** prevents double registration. If `@EnableZaloBot` is applied to multiple `@Configuration` classes (or if auto-configuration also registers the BPP), the guard ensures only one instance exists. This is idempotent — calling it twice is safe.
> ─────────────────────────────────────────────────────

---

## 8.5 How the BPP Gets Its Dependencies

The BPP needs:
- `BeanFactory` → for the registrar to look up factory beans
- `Environment` → for property placeholder resolution
- `ZaloListenerEndpointRegistry` → for the registrar to register containers

How does it get them if it's created via a bean definition (not `@Bean`)?

```
BPP implements BeanFactoryAware     → Spring auto-injects BeanFactory
BPP implements EnvironmentAware     → Spring auto-injects Environment (add this)
BPP.setEndpointRegistry(registry)   → Spring injects via setter or post-construct

OR:

BPP gets registry from BeanFactory in afterSingletonsInstantiated():
    this.endpointRegistry = this.beanFactory.getBean(
        ZALO_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,
        ZaloListenerEndpointRegistry.class);
```

The second approach (lazy lookup from BeanFactory) is simpler and what Spring Kafka does. The BPP doesn't need the registry until `afterSingletonsInstantiated()`, so looking it up at that point avoids circular dependency issues.

Updated `afterSingletonsInstantiated()` to show this:

```java
@Override
public void afterSingletonsInstantiated() {
    // Lazy-load the registry
    ZaloListenerEndpointRegistry registry = this.beanFactory.getBean(
            ZaloBootstrapConfiguration.ZALO_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,
            ZaloListenerEndpointRegistry.class);

    // Configure the registrar
    this.registrar.setBeanFactory(this.beanFactory);
    this.registrar.setEndpointRegistry(registry);
    this.registrar.setDefaultContainerFactoryBeanName(
            DEFAULT_ZALO_LISTENER_CONTAINER_FACTORY_BEAN_NAME);

    // Flush all endpoints to the registry
    this.registrar.afterPropertiesSet();
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Lazy-load dependencies from BeanFactory** is a best practice for BeanPostProcessors. Since BPPs are created very early (before most beans), injecting dependencies via constructor/setter can trigger premature bean creation. Instead, look up what you need from the `BeanFactory` at the latest possible moment — `afterSingletonsInstantiated()`. Spring Kafka's `KafkaListenerAnnotationBeanPostProcessor.afterSingletonsInstantiated()` (line ~340) uses the same pattern.
> ─────────────────────────────────────────────────────

---

## 8.6 The Complete Startup Timeline

```
1. Spring Boot starts
   │
2. @EnableZaloBot detected on @Configuration class
   │  (or via auto-configuration)
   │
3. ZaloBootstrapConfiguration.registerBeanDefinitions()
   │  ├── registers BPP bean definition
   │  └── registers Registry bean definition
   │
4. Spring creates BPP (early, because it's a BeanPostProcessor)
   │  └── BPP receives BeanFactory via BeanFactoryAware
   │
5. Spring creates all other beans...
   │  ├── For each bean:
   │  │   └── BPP.postProcessAfterInitialization(bean)
   │  │       ├── scan for @ZaloListener
   │  │       └── register endpoints with registrar
   │  │
   │  ├── Creates Registry (when needed)
   │  ├── Creates ContainerFactory (user/auto-configured)
   │  ├── Creates ZaloBotClient (auto-configured)
   │  └── Creates ContainerProperties (auto-configured)
   │
6. All singletons created
   │  └── BPP.afterSingletonsInstantiated()
   │      ├── look up Registry from BeanFactory
   │      ├── configure Registrar
   │      └── Registrar.afterPropertiesSet()
   │          └── for each endpoint:
   │              ├── look up Factory from BeanFactory
   │              ├── Factory creates container
   │              └── Registry stores container
   │
7. Application context refresh completes
   │  └── SmartLifecycle beans started
   │      └── Registry.start()
   │          └── all containers.start()
   │              └── ListenerConsumer threads begin polling
   │
8. Application is running ✓
```

---

## 8.7 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `@EnableZaloBot` | `@EnableKafka` | `annotation/EnableKafka.java` |
| `ZaloBootstrapConfiguration` | `KafkaBootstrapConfiguration` | `annotation/KafkaBootstrapConfiguration.java` |
| `ImportBeanDefinitionRegistrar` | `ImportBeanDefinitionRegistrar` | Same Spring interface |
| `ROLE_INFRASTRUCTURE` | `ROLE_INFRASTRUCTURE` | Same constant |
| Bean definition registration | Bean definition registration | Same pattern |
| Lazy registry lookup in `afterSingletonsInstantiated` | Lazy registry lookup | `KafkaListenerAnnotationBeanPostProcessor.java:340` |

**Key simplification:** Spring Kafka's `@EnableKafka` uses a `KafkaListenerConfigurationSelector` (an `ImportSelector`) instead of directly importing the bootstrap config. This adds flexibility for conditional imports. We use direct `@Import` for simplicity.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **@EnableZaloBot** | Trigger annotation that imports the bootstrap configuration |
| **ZaloBootstrapConfiguration** | `ImportBeanDefinitionRegistrar` that registers BPP + Registry |
| **Bean definitions** | Infrastructure beans registered before any regular beans exist |
| **ROLE_INFRASTRUCTURE** | Marks beans as internal framework plumbing |
| **Lazy dependency lookup** | BPP gets Registry from BeanFactory at `afterSingletonsInstantiated()` time |
| **Idempotent registration** | `containsBeanDefinition()` guard prevents duplicates |

**Next: Chapter 9** — We update the Spring Boot auto-configuration to automatically enable `@ZaloListener` support and create the default container factory bean.
