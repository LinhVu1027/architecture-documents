# Chapter 7: The Bean Post-Processor — Discovering @ZaloListener

## What You'll Learn
- How `BeanPostProcessor` and `SmartInitializingSingleton` work together
- How the BPP scans methods for `@ZaloListener` and creates endpoints
- How property placeholder resolution turns `"${concurrency}"` into `3`
- Why proxy-awareness matters in annotation scanning

---

## 7.1 The BPP's Two Responsibilities

The Bean Post-Processor has exactly two jobs:

1. **During bean creation** (`postProcessAfterInitialization`): Scan each bean for `@ZaloListener` methods and register endpoints with the registrar
2. **After all beans created** (`afterSingletonsInstantiated`): Tell the registrar to flush all endpoints to the registry

```
Spring creates Bean A ──▶ BPP scans A ──▶ found @ZaloListener → register endpoint
Spring creates Bean B ──▶ BPP scans B ──▶ no @ZaloListener → skip
Spring creates Bean C ──▶ BPP scans C ──▶ found @ZaloListener → register endpoint
...
All beans created ──▶ BPP.afterSingletonsInstantiated()
                        └── registrar.afterPropertiesSet()
                              └── registerAllEndpoints() → containers created
```

---

## 7.2 The Full BPP Implementation

```java
package dev.linhvu.zalobot.boot.annotation;

import java.lang.reflect.Method;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

import dev.linhvu.zalobot.boot.config.MethodZaloListenerEndpoint;
import dev.linhvu.zalobot.boot.config.ZaloListenerEndpointRegistrar;
import dev.linhvu.zalobot.boot.config.ZaloListenerEndpointRegistry;

import org.springframework.aop.framework.AopProxyUtils;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.SmartInitializingSingleton;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.core.MethodIntrospector;
import org.springframework.core.annotation.AnnotatedElementUtils;
import org.springframework.core.env.Environment;
import org.springframework.util.ReflectionUtils;

/**
 * {@link BeanPostProcessor} that scans beans for {@link ZaloListener @ZaloListener}
 * annotated methods and registers them as listener endpoints.
 *
 * <p>Registered via {@link ZaloBootstrapConfiguration} which is imported
 * by {@link EnableZaloBot @EnableZaloBot}.
 *
 * @author Linh Vu
 * @since 0.1.0
 * @see ZaloListener
 * @see EnableZaloBot
 * @see ZaloListenerEndpointRegistrar
 */
public class ZaloListenerAnnotationBeanPostProcessor
        implements BeanPostProcessor, SmartInitializingSingleton, BeanFactoryAware {

    /**
     * Default container factory bean name.
     */
    public static final String DEFAULT_ZALO_LISTENER_CONTAINER_FACTORY_BEAN_NAME =
            "zaloListenerContainerFactory";

    private BeanFactory beanFactory;
    private Environment environment;
    private ZaloListenerEndpointRegistry endpointRegistry;

    private final ZaloListenerEndpointRegistrar registrar =
            new ZaloListenerEndpointRegistrar();

    private final Set<Class<?>> processedBeanClasses =
            ConcurrentHashMap.newKeySet();

    private final AtomicInteger counter = new AtomicInteger();

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    public void setEndpointRegistry(ZaloListenerEndpointRegistry endpointRegistry) {
        this.endpointRegistry = endpointRegistry;
    }

    // ── BeanPostProcessor ─────────────────────────────────────

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {

        Class<?> targetClass = AopUtils.getTargetClass(bean);

        // Skip beans we've already processed (proxy case)
        if (!this.processedBeanClasses.add(targetClass)) {
            return bean;
        }

        // Find all methods annotated with @ZaloListener
        Map<Method, ZaloListener> annotatedMethods;
        try {
            annotatedMethods = MethodIntrospector.selectMethods(targetClass,
                    (MethodIntrospector.MetadataLookup<ZaloListener>) method ->
                            AnnotatedElementUtils.findMergedAnnotation(
                                    method, ZaloListener.class));
        }
        catch (Throwable ex) {
            // Non-fatal — some classes can't be introspected
            return bean;
        }

        if (annotatedMethods.isEmpty()) {
            return bean;
        }

        // Process each annotated method
        for (Map.Entry<Method, ZaloListener> entry : annotatedMethods.entrySet()) {
            Method method = entry.getKey();
            ZaloListener annotation = entry.getValue();
            processZaloListener(annotation, method, bean, beanName);
        }

        return bean;
    }

    /**
     * Process a single @ZaloListener annotation on a method.
     */
    private void processZaloListener(ZaloListener annotation, Method method,
            Object bean, String beanName) {

        // If bean is a proxy, get the actual target method
        Method targetMethod = checkProxy(method, bean);

        // Create endpoint
        MethodZaloListenerEndpoint endpoint = new MethodZaloListenerEndpoint();
        endpoint.setBean(bean);
        endpoint.setMethod(targetMethod);

        // Resolve annotation attributes
        String id = resolveAnnotationAttribute(annotation.id());
        if (id != null && !id.isBlank()) {
            endpoint.setId(id);
        }

        String concurrency = resolveAnnotationAttribute(annotation.concurrency());
        if (concurrency != null && !concurrency.isBlank()) {
            endpoint.setConcurrency(Integer.parseInt(concurrency));
        }

        String autoStartup = resolveAnnotationAttribute(annotation.autoStartup());
        if (autoStartup != null && !autoStartup.isBlank()) {
            endpoint.setAutoStartup(Boolean.parseBoolean(autoStartup));
        }

        // Resolve container factory
        String factory = resolveAnnotationAttribute(annotation.containerFactory());

        // Register with registrar (deferred)
        this.registrar.registerEndpoint(endpoint, factory);
    }

    // ── SmartInitializingSingleton ──────────────────────────────

    @Override
    public void afterSingletonsInstantiated() {
        // Configure the registrar
        this.registrar.setBeanFactory(this.beanFactory);
        this.registrar.setEndpointRegistry(this.endpointRegistry);
        this.registrar.setDefaultContainerFactoryBeanName(
                DEFAULT_ZALO_LISTENER_CONTAINER_FACTORY_BEAN_NAME);

        // Flush all endpoints to the registry
        this.registrar.afterPropertiesSet();
    }

    // ── Private helpers ────────────────────────────────────────

    /**
     * Resolve property placeholders in an annotation attribute value.
     * Converts "${my.prop}" to the resolved value.
     */
    private String resolveAnnotationAttribute(String value) {
        if (value == null || value.isEmpty()) {
            return value;
        }
        if (this.environment != null) {
            return this.environment.resolvePlaceholders(value);
        }
        return value;
    }

    /**
     * If the bean is a proxy, find the actual method on the target class.
     */
    private Method checkProxy(Method method, Object bean) {
        if (AopUtils.isAopProxy(bean)) {
            Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
            try {
                return targetClass.getMethod(
                        method.getName(), method.getParameterTypes());
            }
            catch (NoSuchMethodException ex) {
                throw new IllegalStateException(
                        "Cannot find target method for proxy: " + method, ex);
            }
        }
        return method;
    }
}
```

---

## 7.3 Method Discovery: How MethodIntrospector Works

The key line is:

```java
Map<Method, ZaloListener> annotatedMethods =
    MethodIntrospector.selectMethods(targetClass,
        (MethodIntrospector.MetadataLookup<ZaloListener>) method ->
            AnnotatedElementUtils.findMergedAnnotation(method, ZaloListener.class));
```

This is a Spring utility that:
1. Walks ALL methods in the class (including inherited and interface-default methods)
2. For each method, calls the `MetadataLookup` lambda
3. If the lambda returns non-null, the method is included in the result map

`AnnotatedElementUtils.findMergedAnnotation()` is preferred over `method.getAnnotation()` because it:
- Handles meta-annotations (annotations on annotations)
- Handles `@AliasFor` attribute merging
- Works across proxy boundaries

Spring Kafka's `KafkaListenerAnnotationBeanPostProcessor` (line ~383) uses the same approach.

> ★ **Insight** ─────────────────────────────────────
> - **Why `AopUtils.getTargetClass(bean)` before scanning?** In Spring, beans can be wrapped in proxies (JDK dynamic proxies or CGLIB). If you scan the proxy class, you might miss annotations on the original class. `AopUtils.getTargetClass()` pierces through proxies to get the real class. This is critical — without it, `@ZaloListener` methods on `@Transactional` or `@Async` beans would be invisible.
> - **Why `processedBeanClasses` set?** A single class can produce multiple bean instances (prototype scope) or be processed multiple times if both the proxy and the target are visible. The set prevents processing the same class twice, which would create duplicate containers.
> ─────────────────────────────────────────────────────

---

## 7.4 Property Placeholder Resolution

When a user writes:

```java
@ZaloListener(concurrency = "${zalobot.text-handler.concurrency}")
public void handleText(GetUpdatesResult update) { ... }
```

And `application.properties` contains:
```properties
zalobot.text-handler.concurrency=5
```

The BPP resolves it:

```
annotation.concurrency() → "${zalobot.text-handler.concurrency}"
    │
    ▼
environment.resolvePlaceholders("${zalobot.text-handler.concurrency}")
    │
    ▼
"5" (String)
    │
    ▼
Integer.parseInt("5") → 5
    │
    ▼
endpoint.setConcurrency(5)
```

**Why not full SpEL (`#{...}`)?**

Spring Kafka supports both `${...}` (property placeholders) and `#{...}` (SpEL expressions). SpEL requires a `BeanExpressionResolver` and adds significant complexity. For ZaloBot, property placeholders cover 99% of use cases. You can always add SpEL support later by using `StandardBeanExpressionResolver`.

> ★ **Insight** ─────────────────────────────────────
> - **`Environment.resolvePlaceholders()` is null-safe for non-placeholder strings.** If the value is just `"3"` (no `${}`), it passes through unchanged. If the value is `"${missing}"` and the property doesn't exist, it returns `"${missing}"` literally (no exception). For strict resolution, use `resolveRequiredPlaceholders()` instead. Spring Kafka uses the stricter `BeanFactory.resolveEmbeddedValue()` which throws on missing placeholders.
> ─────────────────────────────────────────────────────

---

## 7.5 The Complete Processing Flow

```
postProcessAfterInitialization(bean="myHandler", beanName="myHandler")
    │
    ├── targetClass = AopUtils.getTargetClass(bean)
    │   → MyZaloBotHandlers.class (pierces proxy if any)
    │
    ├── processedBeanClasses.add(targetClass)
    │   → true (first time seeing this class)
    │
    ├── MethodIntrospector.selectMethods(targetClass, lookup)
    │   ├── inspects handleText() → found @ZaloListener → included
    │   ├── inspects handleImage() → found @ZaloListener → included
    │   └── inspects toString() → no @ZaloListener → excluded
    │   → {handleText: @ZaloListener(...), handleImage: @ZaloListener(...)}
    │
    ├── processZaloListener(@ZaloListener, handleText, bean, "myHandler")
    │   ├── endpoint = new MethodZaloListenerEndpoint()
    │   ├── endpoint.setBean(bean)
    │   ├── endpoint.setMethod(handleText)
    │   ├── endpoint.setId("text-handler")
    │   ├── endpoint.setConcurrency(3)
    │   └── registrar.registerEndpoint(endpoint, null)
    │
    └── processZaloListener(@ZaloListener, handleImage, bean, "myHandler")
        ├── endpoint = new MethodZaloListenerEndpoint()
        ├── endpoint.setBean(bean)
        ├── endpoint.setMethod(handleImage)
        ├── endpoint.setId("image-handler")
        └── registrar.registerEndpoint(endpoint, null)
```

---

## 7.6 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `ZaloListenerAnnotationBeanPostProcessor` | `KafkaListenerAnnotationBeanPostProcessor` | `annotation/KafkaListenerAnnotationBeanPostProcessor.java` |
| `postProcessAfterInitialization()` | `postProcessAfterInitialization()` | `KafkaListenerAnnotationBeanPostProcessor.java:383` |
| `processZaloListener()` | `processKafkaListener()` + `processListener()` | `KafkaListenerAnnotationBeanPostProcessor.java:494,644` |
| `MethodIntrospector.selectMethods()` | `MethodIntrospector.selectMethods()` | Same Spring utility |
| `AnnotatedElementUtils.findMergedAnnotation()` | `AnnotatedElementUtils.findMergedAnnotation()` | Same Spring utility |
| `AopUtils.getTargetClass()` | `AopUtils.getTargetClass()` | Same Spring utility |
| `SmartInitializingSingleton` | `SmartInitializingSingleton` | Same Spring interface |
| `environment.resolvePlaceholders()` | `beanFactory.resolveEmbeddedValue()` + SpEL resolver | More powerful in Kafka |
| `DEFAULT_ZALO_LISTENER_CONTAINER_FACTORY_BEAN_NAME` | `DEFAULT_KAFKA_LISTENER_CONTAINER_FACTORY_BEAN_NAME` | Constant in BPP |

**Key simplifications:**
- Spring Kafka's BPP handles `@KafkaListener` on classes (multi-method dispatch via `@KafkaHandler`) — we only handle method-level
- Spring Kafka's BPP supports `@KafkaListeners` (repeatable container) — we don't need repeatable
- Spring Kafka's BPP handles retry topic configuration — we skip this
- Spring Kafka's BPP has `AnnotationEnhancer` support — we skip this

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **BeanPostProcessor** | Spring hook called for every bean after initialization — our annotation scanner |
| **SmartInitializingSingleton** | Callback after ALL singletons created — triggers endpoint registration |
| **MethodIntrospector** | Spring utility to find annotated methods across class hierarchies |
| **AopUtils.getTargetClass()** | Pierces proxies to get the real class for annotation scanning |
| **Property placeholder resolution** | `"${prop}"` → resolved via `Environment` at BPP time |
| **processedBeanClasses** | Prevents duplicate processing of the same class |
| **Two-phase flow** | Phase 1: discover & collect; Phase 2: materialize |

**Next: Chapter 8** — We build `@EnableZaloBot` and `ZaloBootstrapConfiguration` that wire the BPP and registry into the Spring context.
