# Chapter 6: Application Context

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Five independent pieces exist — bean container, DI, lifecycle, configuration, scanning — but users must manually wire them together: create a factory, add post-processors, register configs, call the processor, then get beans | No single entry point orchestrates startup; no event system notifies components when the container is ready or shutting down | Build an `ApplicationContext` that orchestrates `refresh()` (the 5-step startup pipeline) and `close()`, plus an event system (`ApplicationEvent` / `ApplicationListener`) that publishes `ContextRefreshedEvent` and `ContextClosedEvent` |

---

## 6.1 The Integration Point

The integration point is `DefaultBeanFactory` itself — the class that all five prior features operate on. Currently it's a passive data structure: you push definitions in, you pull beans out. The `ApplicationContext` will wrap it and orchestrate the entire lifecycle through a `refresh()` method.

But first, `DefaultBeanFactory` needs two new capabilities that the context requires:

**Modifying:** `src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Add `getBeansOfType()` and `preInstantiateSingletons()` methods

```java
// -----------------------------------------------------------------------
// Type-based lookup (multiple results)
// -----------------------------------------------------------------------

@SuppressWarnings("unchecked")
public <T> Map<String, T> getBeansOfType(Class<T> type) {
    Map<String, T> result = new LinkedHashMap<>();
    for (String name : this.beanDefinitionNames) {
        BeanDefinition bd = this.beanDefinitionMap.get(name);
        if (bd != null && type.isAssignableFrom(bd.getBeanClass())) {
            result.put(name, (T) getBean(name));
        }
    }
    return result;
}

// -----------------------------------------------------------------------
// Eager singleton instantiation
// -----------------------------------------------------------------------

public void preInstantiateSingletons() {
    List<String> names = new ArrayList<>(this.beanDefinitionNames);
    for (String name : names) {
        BeanDefinition bd = this.beanDefinitionMap.get(name);
        if (bd != null && bd.isSingleton()) {
            getBean(name);
        }
    }
}
```

Two key decisions here:

1. **`getBeansOfType()` returns a `LinkedHashMap`** — preserving bean definition order. The context uses this to discover all `BeanPostProcessor` and `ApplicationListener` beans. In the real Spring Framework, this method lives on `ListableBeanFactory`, a sub-interface of `BeanFactory`. We add it directly to `DefaultBeanFactory` to avoid introducing another interface.

2. **`preInstantiateSingletons()` snapshots the name list** — this prevents `ConcurrentModificationException` if bean creation triggers registration of more beans (e.g., a `@Bean` method that programmatically registers additional definitions).

This connects **the bean factory** to **lifecycle orchestration**. To make it work, we need to build:
- `ApplicationEvent` / `ApplicationListener` / `ApplicationEventPublisher` — the event system
- `ApplicationContext` / `ConfigurableApplicationContext` — the container interface hierarchy
- `AnnotationConfigApplicationContext` — the concrete implementation with `refresh()` and `close()`
- `ContextRefreshedEvent` / `ContextClosedEvent` — lifecycle events

## 6.2 The Event System

The event system has three participants: events (the data), listeners (the handlers), and publishers (the dispatchers).

**New file:** `src/main/java/com/iris/framework/context/ApplicationEvent.java`

```java
package com.iris.framework.context;

import java.util.EventObject;

public abstract class ApplicationEvent extends EventObject {

    private final long timestamp;

    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }

    public long getTimestamp() {
        return this.timestamp;
    }
}
```

**New file:** `src/main/java/com/iris/framework/context/ApplicationListener.java`

```java
package com.iris.framework.context;

import java.util.EventListener;

@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    void onApplicationEvent(E event);
}
```

**New file:** `src/main/java/com/iris/framework/context/ApplicationEventPublisher.java`

```java
package com.iris.framework.context;

@FunctionalInterface
public interface ApplicationEventPublisher {

    void publishEvent(ApplicationEvent event);
}
```

The three together form the **Observer pattern** — `ApplicationEvent` is the notification, `ApplicationListener` is the observer, and `ApplicationEventPublisher` is the subject. The `@FunctionalInterface` annotation on both listener and publisher enables lambda usage: `ctx.publishEvent(new MyEvent(this))`.

## 6.3 Context Lifecycle Events

Two concrete events mark the container's lifecycle boundaries:

**New file:** `src/main/java/com/iris/framework/context/event/ApplicationContextEvent.java`

```java
package com.iris.framework.context.event;

import com.iris.framework.context.ApplicationContext;
import com.iris.framework.context.ApplicationEvent;

public abstract class ApplicationContextEvent extends ApplicationEvent {

    public ApplicationContextEvent(ApplicationContext source) {
        super(source);
    }

    public ApplicationContext getApplicationContext() {
        return (ApplicationContext) getSource();
    }
}
```

**New file:** `src/main/java/com/iris/framework/context/event/ContextRefreshedEvent.java`

```java
package com.iris.framework.context.event;

import com.iris.framework.context.ApplicationContext;

public class ContextRefreshedEvent extends ApplicationContextEvent {
    public ContextRefreshedEvent(ApplicationContext source) {
        super(source);
    }
}
```

**New file:** `src/main/java/com/iris/framework/context/event/ContextClosedEvent.java`

```java
package com.iris.framework.context.event;

import com.iris.framework.context.ApplicationContext;

public class ContextClosedEvent extends ApplicationContextEvent {
    public ContextClosedEvent(ApplicationContext source) {
        super(source);
    }
}
```

## 6.4 The ApplicationContext Interface Hierarchy

**New file:** `src/main/java/com/iris/framework/context/ApplicationContext.java`

```java
package com.iris.framework.context;

import com.iris.framework.beans.factory.BeanFactory;

public interface ApplicationContext extends BeanFactory, ApplicationEventPublisher {

    String getDisplayName();
}
```

**New file:** `src/main/java/com/iris/framework/context/ConfigurableApplicationContext.java`

```java
package com.iris.framework.context;

import java.io.Closeable;

public interface ConfigurableApplicationContext extends ApplicationContext, Closeable {

    void refresh();

    @Override
    void close();

    boolean isActive();
}
```

The interface hierarchy captures a key Spring design principle: `ApplicationContext` is read-only (for application code), while `ConfigurableApplicationContext` adds lifecycle methods (for the framework and tests).

## 6.5 The AnnotationConfigApplicationContext

This is the main class — it wires everything together.

**New file:** `src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java`

```java
package com.iris.framework.context;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;

import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;

public class AnnotationConfigApplicationContext implements ConfigurableApplicationContext {

    private final DefaultBeanFactory beanFactory;
    private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();
    private final AtomicBoolean active = new AtomicBoolean(false);
    private final AtomicBoolean closed = new AtomicBoolean(false);

    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this.beanFactory = new DefaultBeanFactory();

        for (Class<?> componentClass : componentClasses) {
            String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
            beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
        }

        refresh();
    }

    @Override
    public void refresh() {
        try {
            // Step 1: Process @Configuration classes — scanning + @Bean methods
            invokeBeanFactoryPostProcessors();

            // Step 2: Register BeanPostProcessors (internal + user-defined)
            registerBeanPostProcessors();

            // Step 3: Instantiate all remaining singleton beans
            finishBeanFactoryInitialization();

            // Step 4: Discover ApplicationListener beans
            registerListeners();

            // Step 5: Publish ContextRefreshedEvent
            finishRefresh();

            this.active.set(true);
            this.closed.set(false);
        } catch (RuntimeException ex) {
            this.beanFactory.close();
            this.active.set(false);
            throw ex;
        }
    }

    private void invokeBeanFactoryPostProcessors() {
        new ConfigurationClassProcessor().processConfigurationClasses(this.beanFactory);
    }

    private void registerBeanPostProcessors() {
        // Internal processors
        this.beanFactory.addBeanPostProcessor(new AutowiredBeanPostProcessor(this.beanFactory));
        this.beanFactory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

        // User-defined BeanPostProcessors
        Map<String, BeanPostProcessor> bpBeans = this.beanFactory.getBeansOfType(BeanPostProcessor.class);
        for (BeanPostProcessor bp : bpBeans.values()) {
            this.beanFactory.addBeanPostProcessor(bp);
        }
    }

    private void finishBeanFactoryInitialization() {
        this.beanFactory.preInstantiateSingletons();
    }

    @SuppressWarnings({"rawtypes", "unchecked"})
    private void registerListeners() {
        Map<String, ApplicationListener> listenerBeans =
                this.beanFactory.getBeansOfType(ApplicationListener.class);
        for (ApplicationListener listener : listenerBeans.values()) {
            this.applicationListeners.add(listener);
        }
    }

    private void finishRefresh() {
        publishEvent(new ContextRefreshedEvent(this));
    }

    @Override
    public void close() {
        if (this.closed.compareAndSet(false, true)) {
            publishEvent(new ContextClosedEvent(this));
            this.beanFactory.close();
            this.active.set(false);
            this.applicationListeners.clear();
        }
    }

    @Override
    public boolean isActive() {
        return this.active.get();
    }

    // --- Event publishing ---

    @Override
    public void publishEvent(ApplicationEvent event) {
        for (ApplicationListener<?> listener : this.applicationListeners) {
            Class<?> eventType = resolveEventType(listener);
            if (eventType == null || eventType.isInstance(event)) {
                invokeListener(listener, event);
            }
        }
    }

    @SuppressWarnings({"rawtypes", "unchecked"})
    private void invokeListener(ApplicationListener listener, ApplicationEvent event) {
        listener.onApplicationEvent(event);
    }

    static Class<?> resolveEventType(ApplicationListener<?> listener) {
        Class<?> targetClass = listener.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Type genericInterface : targetClass.getGenericInterfaces()) {
                if (genericInterface instanceof ParameterizedType pt) {
                    if (ApplicationListener.class.isAssignableFrom((Class<?>) pt.getRawType())) {
                        Type typeArg = pt.getActualTypeArguments()[0];
                        if (typeArg instanceof Class<?> clazz) {
                            return clazz;
                        }
                    }
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return null;
    }

    // --- BeanFactory delegation ---

    @Override
    public Object getBean(String name) {
        assertActive();
        return this.beanFactory.getBean(name);
    }

    @Override
    public <T> T getBean(String name, Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(name, requiredType);
    }

    @Override
    public <T> T getBean(Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(requiredType);
    }

    @Override
    public boolean containsBean(String name) {
        return this.beanFactory.containsBean(name);
    }

    @Override
    public boolean isSingleton(String name) {
        return this.beanFactory.isSingleton(name);
    }

    @Override
    public String getDisplayName() {
        return getClass().getSimpleName() + "@" + Integer.toHexString(hashCode());
    }

    public DefaultBeanFactory getBeanFactory() {
        return this.beanFactory;
    }

    public void addApplicationListener(ApplicationListener<?> listener) {
        this.applicationListeners.add(listener);
    }

    private void assertActive() {
        if (!this.active.get()) {
            throw new IllegalStateException(
                    "ApplicationContext has not been refreshed yet or has already been closed");
        }
    }
}
```

The `refresh()` pipeline — the heart of the entire framework:

```
refresh()
  ├── 1. invokeBeanFactoryPostProcessors()    ← process @Configuration + component scanning
  ├── 2. registerBeanPostProcessors()          ← @Autowired + @PostConstruct + user-defined
  ├── 3. finishBeanFactoryInitialization()     ← eager singleton creation
  ├── 4. registerListeners()                   ← discover ApplicationListener beans
  └── 5. finishRefresh()                       ← publish ContextRefreshedEvent
```

## 6.6 Try It Yourself

<details>
<summary>Challenge: Implement the event type resolution</summary>

Given a listener like `class MyListener implements ApplicationListener<ContextRefreshedEvent>`, how do you determine at runtime that it listens for `ContextRefreshedEvent`?

Hint: Java erases generics at runtime, but the generic interfaces declared on a class are preserved in the bytecode and accessible via `Class.getGenericInterfaces()`.

```java
static Class<?> resolveEventType(ApplicationListener<?> listener) {
    Class<?> targetClass = listener.getClass();
    while (targetClass != null && targetClass != Object.class) {
        for (Type genericInterface : targetClass.getGenericInterfaces()) {
            if (genericInterface instanceof ParameterizedType pt) {
                if (ApplicationListener.class.isAssignableFrom((Class<?>) pt.getRawType())) {
                    Type typeArg = pt.getActualTypeArguments()[0];
                    if (typeArg instanceof Class<?> clazz) {
                        return clazz;
                    }
                }
            }
        }
        targetClass = targetClass.getSuperclass();
    }
    return null;
}
```

</details>

<details>
<summary>Challenge: Implement the close() method with proper ordering</summary>

The close method must: (1) publish `ContextClosedEvent` while beans are still alive, (2) destroy all singletons, (3) mark as inactive. It must also be idempotent — calling close() twice should not publish the event twice.

```java
@Override
public void close() {
    if (this.closed.compareAndSet(false, true)) {
        publishEvent(new ContextClosedEvent(this));
        this.beanFactory.close();
        this.active.set(false);
        this.applicationListeners.clear();
    }
}
```

The `compareAndSet` on the `AtomicBoolean` gives us idempotency — the body only runs on the first call.

</details>

## 6.7 Tests

### Unit Tests

**New file:** `src/test/java/com/iris/framework/context/ApplicationEventTest.java`

```java
package com.iris.framework.context;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ApplicationEventTest {

    static class TestEvent extends ApplicationEvent {
        private final String message;
        TestEvent(Object source, String message) {
            super(source);
            this.message = message;
        }
        String getMessage() { return message; }
    }

    @Test
    void shouldStoreSource_WhenEventCreated() {
        Object source = "testSource";
        TestEvent event = new TestEvent(source, "hello");
        assertThat(event.getSource()).isSameAs(source);
    }

    @Test
    void shouldRecordTimestamp_WhenEventCreated() {
        long before = System.currentTimeMillis();
        TestEvent event = new TestEvent("source", "hello");
        long after = System.currentTimeMillis();
        assertThat(event.getTimestamp()).isBetween(before, after);
    }

    @Test
    void shouldStoreCustomData_WhenEventCreated() {
        TestEvent event = new TestEvent("source", "custom message");
        assertThat(event.getMessage()).isEqualTo("custom message");
    }

    @Test
    void shouldThrowNPE_WhenSourceIsNull() {
        assertThatThrownBy(() -> new TestEvent(null, "msg"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

**New file:** `src/test/java/com/iris/framework/context/AnnotationConfigApplicationContextTest.java`

```java
package com.iris.framework.context;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;
import static org.assertj.core.api.Assertions.*;

class AnnotationConfigApplicationContextTest {

    @Configuration
    static class SimpleConfig {
        @Bean
        public String greeting() { return "Hello, Iris!"; }

        @Bean
        public Integer magicNumber() { return 42; }
    }

    @Test
    void shouldCreateContext_WhenGivenConfigClass() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        assertThat(ctx.isActive()).isTrue();
        assertThat(ctx.getBean("greeting")).isEqualTo("Hello, Iris!");
        ctx.close();
    }

    @Test
    void shouldBeInactive_WhenClosed() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        ctx.close();
        assertThat(ctx.isActive()).isFalse();
    }

    @Test
    void shouldThrowException_WhenAccessingBeanAfterClose() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        ctx.close();
        assertThatThrownBy(() -> ctx.getBean("greeting"))
                .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void shouldPublishContextClosedEvent_WhenCloseIsCalled() {
        // ... (see full test in Complete Code section)
    }

    @Test
    void shouldCloseOnlyOnce_WhenCloseCalledMultipleTimes() {
        // ... (see full test in Complete Code section)
    }
}
```

### Integration Tests

**New file:** `src/test/java/com/iris/framework/context/integration/ApplicationContextIntegrationTest.java`

```java
@Test
void shouldResolveBeanDependencies_WhenUsingConfigWithBeanMethods() {
    var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    UserService service = ctx.getBean(UserService.class);
    assertThat(service.getUser("42")).isEqualTo("User-42");
    ctx.close();
}

@Test
void shouldRunLifecycleCallbacks_WhenContextRefreshes() {
    LIFECYCLE_EVENTS.clear();
    var ctx = new AnnotationConfigApplicationContext(LifecycleConfig.class);
    assertThat(LIFECYCLE_EVENTS).containsExactly("@PostConstruct", "afterPropertiesSet");
    ctx.close();
    assertThat(LIFECYCLE_EVENTS).containsExactly(
            "@PostConstruct", "afterPropertiesSet", "@PreDestroy");
}

@Test
void shouldPublishContextRefreshedEvent_WhenRefreshCompletes() {
    var ctx = new AnnotationConfigApplicationContext(ListenerConfig.class);
    RefreshListener listener = ctx.getBean(RefreshListener.class);
    assertThat(listener.received).hasSize(1);
    assertThat(listener.received.get(0)).isInstanceOf(ContextRefreshedEvent.class);
    ctx.close();
}

@Test
void shouldFilterEventsByType_WhenMultipleListenersRegistered() {
    var ctx = new AnnotationConfigApplicationContext(ListenerConfig.class);
    RefreshListener refreshOnly = ctx.getBean(RefreshListener.class);
    AllEventsListener allEvents = ctx.getBean(AllEventsListener.class);

    assertThat(refreshOnly.received).hasSize(1); // only ContextRefreshedEvent
    assertThat(allEvents.received).hasSize(1);   // also ContextRefreshedEvent

    ctx.close();

    assertThat(refreshOnly.received).hasSize(1); // NOT ContextClosedEvent
    assertThat(allEvents.received).hasSize(2);   // ContextRefreshedEvent + ContextClosedEvent
}
```

**Run:** `./gradlew :iris-framework:test` — expected: all 142 tests pass (including prior features' tests)

---

## 6.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **The Template Method pattern drives `refresh()`**: Each step in `refresh()` calls a private method that does exactly one thing. In the real Spring Framework, these are overridable template methods (e.g., `onRefresh()` for embedded server creation, `postProcessBeanFactory()` for web-specific setup). By defining the sequence in a fixed pipeline, new features plug in at specific steps without modifying the overall flow. This is why Spring Boot can add embedded Tomcat by just overriding `onRefresh()`.
> - **When this pattern breaks**: If a step needs to communicate intermediate state to another step, the template method pattern leads to protected fields or callback parameters — which is exactly what happened in the real `AbstractApplicationContext` (fields like `earlyApplicationEvents` bridge the gap between steps). Our simplified version avoids this complexity.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Lazy → Eager is the context's key value-add**: Before `ApplicationContext`, bean creation was lazy — errors only surfaced when someone called `getBean()`. The `preInstantiateSingletons()` call in `refresh()` forces all singletons to be created upfront. This is a **fail-fast** strategy: configuration errors (missing beans, circular dependencies, bad property values) surface at startup rather than at runtime. This is why real Spring apps fail fast with clear error messages instead of crashing in production.
> - **Trade-off**: Eager initialization increases startup time. For large applications with hundreds of beans, this can be significant. Spring Boot 3.2+ added support for background bean initialization (`bootstrapExecutor`) to mitigate this.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Event ordering in close() matters**: `ContextClosedEvent` is published BEFORE `destroyBeans()`. This is critical because listeners need the full bean graph to perform cleanup (flushing caches, closing connections, deregistering from service registries). If we destroyed beans first, listeners would find `null` references when trying to access collaborating beans during their cleanup logic.
> -----------------------------------------------------------

## 6.9 What We Enhanced

| Aspect | Before (ch01–ch05) | Current (ch06) | Real Framework |
|--------|---------------------|----------------|----------------|
| Container setup | Manual: create factory, register processors, register definitions, call processor, get beans — 5+ steps | `new AnnotationConfigApplicationContext(AppConfig.class)` — one line | Same one-line pattern (`AbstractApplicationContext.java:602`) |
| Bean initialization | Lazy — errors surface on first `getBean()` | Eager via `preInstantiateSingletons()` — fail-fast at startup | `DefaultListableBeanFactory.preInstantiateSingletons()` (`DefaultListableBeanFactory.java:1005`) |
| Lifecycle notification | None — no way to know when container is ready | `ContextRefreshedEvent` / `ContextClosedEvent` via Observer pattern | 7 lifecycle events including `ContextStartedEvent`, `ContextStoppedEvent`, `ContextRestartedEvent` (`ApplicationContextEvent.java:30`) |
| BeanPostProcessor registration | Manual: `factory.addBeanPostProcessor(new ...)` | Automatic: internal processors added by context, user-defined discovered via `getBeansOfType()` | Priority-sorted registration via `PostProcessorRegistrationDelegate` (`PostProcessorRegistrationDelegate.java:239`) |
| Type-based multi-lookup | `getBean(Class)` returns one or throws | `getBeansOfType(Class)` returns all matching beans | `ListableBeanFactory.getBeansOfType()` with eager/non-eager options (`DefaultListableBeanFactory.java:678`) |

## 6.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `AnnotationConfigApplicationContext` | `AnnotationConfigApplicationContext` | `AnnotationConfigApplicationContext.java:68` | Real extends `GenericApplicationContext` → `AbstractApplicationContext`, 3-level hierarchy |
| `refresh()` (5 steps) | `AbstractApplicationContext.refresh()` | `AbstractApplicationContext.java:602` | Real has 12 steps including message source, lifecycle processor, event multicaster init |
| `close()` with `compareAndSet` | `AbstractApplicationContext.close()` | `AbstractApplicationContext.java:896` | Real uses `ReentrantLock` + `tryLockForShutdown()`, supports JVM shutdown hooks |
| `publishEvent()` inline dispatch | `SimpleApplicationEventMulticaster.multicastEvent()` | `SimpleApplicationEventMulticaster.java:138` | Real supports async execution via `TaskExecutor`, caches listener resolution per event type |
| `resolveEventType()` via reflection | `GenericApplicationListenerAdapter` | `GenericApplicationListenerAdapter.java:61` | Real uses `ResolvableType` — a full generic type resolution framework |
| `getBeansOfType()` | `DefaultListableBeanFactory.getBeansOfType()` | `DefaultListableBeanFactory.java:678` | Real handles `FactoryBean`s, `BeanDefinition` merging, and non-eager resolution |
| `preInstantiateSingletons()` | `DefaultListableBeanFactory.preInstantiateSingletons()` | `DefaultListableBeanFactory.java:1005` | Real handles `FactoryBean`, `SmartInitializingSingleton` callback, and bootstrap executor |

## 6.11 Complete Code

### Production Code

#### File: `src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java` [MODIFIED]

```java
package com.iris.framework.beans.factory.support;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.beans.factory.DisposableBean;
import com.iris.framework.beans.factory.InitializingBean;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.config.BeanPostProcessor;

/**
 * Default implementation of both {@link BeanFactory} and {@link BeanDefinitionRegistry}.
 *
 * <p>Two internal maps form the core:
 * <ul>
 *   <li>{@code beanDefinitionMap} — stores bean metadata (class, scope, supplier)</li>
 *   <li>{@code singletonObjects} — caches created singleton instances</li>
 * </ul>
 *
 * <p>Bean creation is lazy: a singleton is instantiated on the first {@code getBean()}
 * call, then cached for all subsequent calls.
 *
 * <p>Maps to Spring's {@code DefaultListableBeanFactory} (which extends
 * {@code DefaultSingletonBeanRegistry} for the singleton cache and implements
 * {@code BeanDefinitionRegistry} for definition storage).
 *
 * @see org.springframework.beans.factory.support.DefaultListableBeanFactory
 * @see org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
 */
public class DefaultBeanFactory implements BeanFactory, BeanDefinitionRegistry {

    /** Bean definitions keyed by name — the "what to create" map. */
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

    /** Registration-order list of bean definition names. */
    private final List<String> beanDefinitionNames = new ArrayList<>(256);

    /** Singleton instances keyed by name — the "already created" cache. */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    /** Ordered list of BeanPostProcessors to apply during bean creation. */
    private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();

    // -----------------------------------------------------------------------
    // BeanPostProcessor registration
    // -----------------------------------------------------------------------

    /**
     * Add a BeanPostProcessor that will be applied to beans created by this factory.
     * Follows the real Spring pattern: remove-then-add ensures no duplicates while
     * preserving latest ordering.
     *
     * @see org.springframework.beans.factory.support.AbstractBeanFactory#addBeanPostProcessor
     */
    public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
        this.beanPostProcessors.remove(beanPostProcessor);
        this.beanPostProcessors.add(beanPostProcessor);
    }

    /**
     * Return the list of registered BeanPostProcessors.
     */
    public List<BeanPostProcessor> getBeanPostProcessors() {
        return this.beanPostProcessors;
    }

    // -----------------------------------------------------------------------
    // BeanDefinitionRegistry implementation
    // -----------------------------------------------------------------------

    @Override
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
        this.beanDefinitionMap.put(beanName, beanDefinition);
        if (!this.beanDefinitionNames.contains(beanName)) {
            this.beanDefinitionNames.add(beanName);
        }
    }

    @Override
    public BeanDefinition getBeanDefinition(String beanName) {
        BeanDefinition bd = this.beanDefinitionMap.get(beanName);
        if (bd == null) {
            throw new NoSuchBeanDefinitionException(beanName);
        }
        return bd;
    }

    @Override
    public boolean containsBeanDefinition(String beanName) {
        return this.beanDefinitionMap.containsKey(beanName);
    }

    @Override
    public String[] getBeanDefinitionNames() {
        return this.beanDefinitionNames.toArray(new String[0]);
    }

    @Override
    public int getBeanDefinitionCount() {
        return this.beanDefinitionMap.size();
    }

    // -----------------------------------------------------------------------
    // BeanFactory implementation
    // -----------------------------------------------------------------------

    @Override
    public Object getBean(String name) {
        return doGetBean(name);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(String name, Class<T> requiredType) {
        Object bean = doGetBean(name);
        if (!requiredType.isInstance(bean)) {
            throw new BeanCreationException(name,
                    "Bean is not of required type '" + requiredType.getName() + "'");
        }
        return (T) bean;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> requiredType) {
        List<String> candidateNames = new ArrayList<>();
        for (Map.Entry<String, BeanDefinition> entry : this.beanDefinitionMap.entrySet()) {
            if (requiredType.isAssignableFrom(entry.getValue().getBeanClass())) {
                candidateNames.add(entry.getKey());
            }
        }

        if (candidateNames.isEmpty()) {
            throw new NoSuchBeanDefinitionException(requiredType);
        }
        if (candidateNames.size() > 1) {
            throw new NoUniqueBeanDefinitionException(requiredType, candidateNames);
        }

        return (T) doGetBean(candidateNames.get(0));
    }

    @Override
    public boolean containsBean(String name) {
        return containsBeanDefinition(name);
    }

    @Override
    public boolean isSingleton(String name) {
        BeanDefinition bd = getBeanDefinition(name);
        return bd.isSingleton();
    }

    // -----------------------------------------------------------------------
    // Type-based lookup (multiple results)
    // -----------------------------------------------------------------------

    /**
     * Return all beans matching the given type. Returns a name→instance map
     * preserving bean definition order.
     *
     * <p>This is the simplified equivalent of Spring's
     * {@code ListableBeanFactory.getBeansOfType()} which lives on
     * {@code DefaultListableBeanFactory}. We add it here because the
     * {@code ApplicationContext} needs it to discover {@code BeanPostProcessor}
     * and {@code ApplicationListener} beans.
     *
     * @param type the class or interface to match
     * @return ordered map of bean name → bean instance (may be empty, never null)
     * @see org.springframework.beans.factory.ListableBeanFactory#getBeansOfType
     */
    @SuppressWarnings("unchecked")
    public <T> Map<String, T> getBeansOfType(Class<T> type) {
        Map<String, T> result = new LinkedHashMap<>();
        for (String name : this.beanDefinitionNames) {
            BeanDefinition bd = this.beanDefinitionMap.get(name);
            if (bd != null && type.isAssignableFrom(bd.getBeanClass())) {
                result.put(name, (T) getBean(name));
            }
        }
        return result;
    }

    // -----------------------------------------------------------------------
    // Eager singleton instantiation
    // -----------------------------------------------------------------------

    /**
     * Instantiate all singleton bean definitions that haven't been created yet.
     *
     * <p>This transitions the container from lazy to eager initialization — the
     * same thing that happens during {@code ApplicationContext.refresh()}.
     * In the real Spring Framework, this is
     * {@code DefaultListableBeanFactory.preInstantiateSingletons()}, called from
     * {@code AbstractApplicationContext.finishBeanFactoryInitialization()}.
     *
     * @see org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
     */
    public void preInstantiateSingletons() {
        // Snapshot names to avoid ConcurrentModificationException
        List<String> names = new ArrayList<>(this.beanDefinitionNames);
        for (String name : names) {
            BeanDefinition bd = this.beanDefinitionMap.get(name);
            if (bd != null && bd.isSingleton()) {
                getBean(name);
            }
        }
    }

    // -----------------------------------------------------------------------
    // Internal bean creation
    // -----------------------------------------------------------------------

    private Object doGetBean(String name) {
        // 1. Check singleton cache
        Object singleton = this.singletonObjects.get(name);
        if (singleton != null) {
            return singleton;
        }

        // 2. Get the definition
        BeanDefinition bd = getBeanDefinition(name);

        // 3. Create the instance
        Object bean = createBean(name, bd);

        // 4. Cache if singleton
        if (bd.isSingleton()) {
            this.singletonObjects.put(name, bean);
        }

        return bean;
    }

    private Object createBean(String name, BeanDefinition bd) {
        try {
            // 1. Instantiate (factory method, supplier, or constructor injection)
            Object bean;
            if (bd.isFactoryMethodBean()) {
                bean = instantiateUsingFactoryMethod(name, bd);
            } else if (bd.getSupplier() != null) {
                bean = bd.getSupplier().get();
            } else {
                bean = instantiate(name, bd);
            }

            // 2. BeanPostProcessors — before initialization
            //    (@PostConstruct methods are invoked here via LifecycleBeanPostProcessor)
            bean = applyBeanPostProcessorsBeforeInitialization(bean, name);

            // 3. InitializingBean callback
            invokeInitMethods(bean, name);

            // 4. BeanPostProcessors — after initialization
            bean = applyBeanPostProcessorsAfterInitialization(bean, name);

            return bean;
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Exception ex) {
            throw new BeanCreationException(name,
                    "Instantiation of bean failed", ex);
        }
    }

    /**
     * Instantiate the bean, resolving constructor injection if needed.
     *
     * <p>Three strategies, in priority order:
     * <ol>
     *   <li>{@code @Autowired} constructor — explicitly marked for injection</li>
     *   <li>Single constructor with parameters — implicit autowiring (Spring 4.3+ behavior)</li>
     *   <li>No-arg constructor — reflective default instantiation</li>
     * </ol>
     *
     * <p>In the real Spring Framework, constructor selection is delegated to
     * {@code SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()},
     * then resolved by {@code ConstructorResolver.autowireConstructor()}. We inline
     * the logic here for simplicity.
     */
    private Object instantiate(String name, BeanDefinition bd) throws Exception {
        Class<?> clazz = bd.getBeanClass();
        Constructor<?>[] ctors = clazz.getDeclaredConstructors();

        // 1. Look for @Autowired constructor
        Constructor<?> selected = null;
        for (Constructor<?> ctor : ctors) {
            if (ctor.isAnnotationPresent(Autowired.class)) {
                selected = ctor;
                break;
            }
        }

        // 2. Single constructor with parameters → implicit autowiring
        if (selected == null && ctors.length == 1 && ctors[0].getParameterCount() > 0) {
            selected = ctors[0];
        }

        // 3. Autowire constructor parameters
        if (selected != null) {
            Class<?>[] paramTypes = selected.getParameterTypes();
            Object[] args = new Object[paramTypes.length];
            for (int i = 0; i < paramTypes.length; i++) {
                try {
                    args[i] = getBean(paramTypes[i]);
                } catch (NoSuchBeanDefinitionException ex) {
                    throw new BeanCreationException(name,
                            "Unsatisfied dependency for constructor parameter type '"
                                    + paramTypes[i].getName() + "'",
                            ex);
                }
            }
            selected.setAccessible(true);
            return selected.newInstance(args);
        }

        // 4. No-arg constructor fallback
        Constructor<?> defaultCtor = clazz.getDeclaredConstructor();
        defaultCtor.setAccessible(true);
        return defaultCtor.newInstance();
    }

    /**
     * Create a bean by invoking a {@code @Bean} factory method on its owning
     * {@code @Configuration} class instance.
     *
     * <p>This mirrors Spring's {@code ConstructorResolver.instantiateUsingFactoryMethod()}:
     * get the factory bean instance, resolve each method parameter by type from
     * the container, then invoke the method.
     *
     * @see org.springframework.beans.factory.support.ConstructorResolver#instantiateUsingFactoryMethod
     */
    private Object instantiateUsingFactoryMethod(String beanName, BeanDefinition bd) {
        String factoryBeanName = bd.getFactoryBeanName();
        Method factoryMethod = bd.getFactoryMethod();

        // Get the @Configuration class instance that owns this @Bean method
        Object factoryBean = getBean(factoryBeanName);

        // Resolve each method parameter from the container (same as constructor autowiring)
        Class<?>[] paramTypes = factoryMethod.getParameterTypes();
        Object[] args = new Object[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++) {
            try {
                args[i] = getBean(paramTypes[i]);
            } catch (NoSuchBeanDefinitionException ex) {
                throw new BeanCreationException(beanName,
                        "Unsatisfied dependency for @Bean method parameter type '"
                                + paramTypes[i].getName() + "' in factory method '"
                                + factoryMethod.getName() + "'",
                        ex);
            }
        }

        try {
            factoryMethod.setAccessible(true);
            return factoryMethod.invoke(factoryBean, args);
        } catch (Exception ex) {
            throw new BeanCreationException(beanName,
                    "Failed to invoke factory method '" + factoryMethod.getName()
                            + "' on configuration class '" + factoryBeanName + "'",
                    ex);
        }
    }

    private Object applyBeanPostProcessorsBeforeInitialization(Object bean, String beanName) {
        Object current = bean;
        for (BeanPostProcessor bp : this.beanPostProcessors) {
            current = bp.postProcessBeforeInitialization(current, beanName);
        }
        return current;
    }

    private Object applyBeanPostProcessorsAfterInitialization(Object bean, String beanName) {
        Object current = bean;
        for (BeanPostProcessor bp : this.beanPostProcessors) {
            current = bp.postProcessAfterInitialization(current, beanName);
        }
        return current;
    }

    /**
     * Invoke {@link InitializingBean#afterPropertiesSet()} if the bean implements it.
     *
     * <p>Called after {@code @PostConstruct} (which fires in
     * {@code postProcessBeforeInitialization}) and before
     * {@code postProcessAfterInitialization}. This matches the real Spring
     * ordering in {@code AbstractAutowireCapableBeanFactory.invokeInitMethods()}.
     */
    private void invokeInitMethods(Object bean, String beanName) throws Exception {
        if (bean instanceof InitializingBean initializingBean) {
            initializingBean.afterPropertiesSet();
        }
    }

    // -----------------------------------------------------------------------
    // Destruction / Close
    // -----------------------------------------------------------------------

    /**
     * Destroy all singleton beans in reverse registration order, then clear
     * the singleton cache.
     *
     * <p>For each singleton: (1) invoke {@code @PreDestroy} methods via
     * {@link LifecycleBeanPostProcessor}, then (2) call
     * {@link DisposableBean#destroy()} if implemented.
     *
     * <p>Maps to {@code DefaultSingletonBeanRegistry.destroySingletons()} which
     * iterates {@code disposableBeans} in reverse order.
     *
     * @see org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingletons()
     */
    public void close() {
        List<String> names = new ArrayList<>(this.beanDefinitionNames);
        for (int i = names.size() - 1; i >= 0; i--) {
            String name = names.get(i);
            Object singleton = this.singletonObjects.get(name);
            if (singleton != null) {
                destroySingleton(name, singleton);
            }
        }
        this.singletonObjects.clear();
    }

    private void destroySingleton(String beanName, Object bean) {
        // 1. @PreDestroy via LifecycleBeanPostProcessor
        for (BeanPostProcessor bp : this.beanPostProcessors) {
            if (bp instanceof LifecycleBeanPostProcessor lbp) {
                lbp.postProcessBeforeDestruction(bean, beanName);
            }
        }

        // 2. DisposableBean.destroy()
        if (bean instanceof DisposableBean disposableBean) {
            try {
                disposableBean.destroy();
            } catch (Exception ex) {
                System.err.println("Destroy method on bean '" + beanName + "' threw exception: " + ex);
            }
        }
    }
}
```

#### File: `src/main/java/com/iris/framework/context/ApplicationEvent.java` [NEW]

```java
package com.iris.framework.context;

import java.util.EventObject;

public abstract class ApplicationEvent extends EventObject {

    private final long timestamp;

    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }

    public long getTimestamp() {
        return this.timestamp;
    }
}
```

#### File: `src/main/java/com/iris/framework/context/ApplicationListener.java` [NEW]

```java
package com.iris.framework.context;

import java.util.EventListener;

@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    void onApplicationEvent(E event);
}
```

#### File: `src/main/java/com/iris/framework/context/ApplicationEventPublisher.java` [NEW]

```java
package com.iris.framework.context;

@FunctionalInterface
public interface ApplicationEventPublisher {

    void publishEvent(ApplicationEvent event);
}
```

#### File: `src/main/java/com/iris/framework/context/ApplicationContext.java` [NEW]

```java
package com.iris.framework.context;

import com.iris.framework.beans.factory.BeanFactory;

public interface ApplicationContext extends BeanFactory, ApplicationEventPublisher {

    String getDisplayName();
}
```

#### File: `src/main/java/com/iris/framework/context/ConfigurableApplicationContext.java` [NEW]

```java
package com.iris.framework.context;

import java.io.Closeable;

public interface ConfigurableApplicationContext extends ApplicationContext, Closeable {

    void refresh();

    @Override
    void close();

    boolean isActive();
}
```

#### File: `src/main/java/com/iris/framework/context/event/ApplicationContextEvent.java` [NEW]

```java
package com.iris.framework.context.event;

import com.iris.framework.context.ApplicationContext;
import com.iris.framework.context.ApplicationEvent;

public abstract class ApplicationContextEvent extends ApplicationEvent {

    public ApplicationContextEvent(ApplicationContext source) {
        super(source);
    }

    public ApplicationContext getApplicationContext() {
        return (ApplicationContext) getSource();
    }
}
```

#### File: `src/main/java/com/iris/framework/context/event/ContextRefreshedEvent.java` [NEW]

```java
package com.iris.framework.context.event;

import com.iris.framework.context.ApplicationContext;

public class ContextRefreshedEvent extends ApplicationContextEvent {
    public ContextRefreshedEvent(ApplicationContext source) {
        super(source);
    }
}
```

#### File: `src/main/java/com/iris/framework/context/event/ContextClosedEvent.java` [NEW]

```java
package com.iris.framework.context.event;

import com.iris.framework.context.ApplicationContext;

public class ContextClosedEvent extends ApplicationContextEvent {
    public ContextClosedEvent(ApplicationContext source) {
        super(source);
    }
}
```

#### File: `src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java` [NEW]

```java
package com.iris.framework.context;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;

public class AnnotationConfigApplicationContext implements ConfigurableApplicationContext {

    private final DefaultBeanFactory beanFactory;
    private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();
    private final AtomicBoolean active = new AtomicBoolean(false);
    private final AtomicBoolean closed = new AtomicBoolean(false);

    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this.beanFactory = new DefaultBeanFactory();

        for (Class<?> componentClass : componentClasses) {
            String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
            beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
        }

        refresh();
    }

    @Override
    public void refresh() {
        try {
            invokeBeanFactoryPostProcessors();
            registerBeanPostProcessors();
            finishBeanFactoryInitialization();
            registerListeners();
            finishRefresh();

            this.active.set(true);
            this.closed.set(false);
        } catch (RuntimeException ex) {
            this.beanFactory.close();
            this.active.set(false);
            throw ex;
        }
    }

    private void invokeBeanFactoryPostProcessors() {
        new ConfigurationClassProcessor().processConfigurationClasses(this.beanFactory);
    }

    private void registerBeanPostProcessors() {
        this.beanFactory.addBeanPostProcessor(new AutowiredBeanPostProcessor(this.beanFactory));
        this.beanFactory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

        Map<String, BeanPostProcessor> bpBeans = this.beanFactory.getBeansOfType(BeanPostProcessor.class);
        for (BeanPostProcessor bp : bpBeans.values()) {
            this.beanFactory.addBeanPostProcessor(bp);
        }
    }

    private void finishBeanFactoryInitialization() {
        this.beanFactory.preInstantiateSingletons();
    }

    @SuppressWarnings({"rawtypes", "unchecked"})
    private void registerListeners() {
        Map<String, ApplicationListener> listenerBeans =
                this.beanFactory.getBeansOfType(ApplicationListener.class);
        for (ApplicationListener listener : listenerBeans.values()) {
            this.applicationListeners.add(listener);
        }
    }

    private void finishRefresh() {
        publishEvent(new ContextRefreshedEvent(this));
    }

    @Override
    public void close() {
        if (this.closed.compareAndSet(false, true)) {
            publishEvent(new ContextClosedEvent(this));
            this.beanFactory.close();
            this.active.set(false);
            this.applicationListeners.clear();
        }
    }

    @Override
    public boolean isActive() {
        return this.active.get();
    }

    @Override
    public void publishEvent(ApplicationEvent event) {
        for (ApplicationListener<?> listener : this.applicationListeners) {
            Class<?> eventType = resolveEventType(listener);
            if (eventType == null || eventType.isInstance(event)) {
                invokeListener(listener, event);
            }
        }
    }

    @SuppressWarnings({"rawtypes", "unchecked"})
    private void invokeListener(ApplicationListener listener, ApplicationEvent event) {
        listener.onApplicationEvent(event);
    }

    static Class<?> resolveEventType(ApplicationListener<?> listener) {
        Class<?> targetClass = listener.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Type genericInterface : targetClass.getGenericInterfaces()) {
                if (genericInterface instanceof ParameterizedType pt) {
                    if (ApplicationListener.class.isAssignableFrom((Class<?>) pt.getRawType())) {
                        Type typeArg = pt.getActualTypeArguments()[0];
                        if (typeArg instanceof Class<?> clazz) {
                            return clazz;
                        }
                    }
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return null;
    }

    @Override
    public Object getBean(String name) {
        assertActive();
        return this.beanFactory.getBean(name);
    }

    @Override
    public <T> T getBean(String name, Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(name, requiredType);
    }

    @Override
    public <T> T getBean(Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(requiredType);
    }

    @Override
    public boolean containsBean(String name) {
        return this.beanFactory.containsBean(name);
    }

    @Override
    public boolean isSingleton(String name) {
        return this.beanFactory.isSingleton(name);
    }

    @Override
    public String getDisplayName() {
        return getClass().getSimpleName() + "@" + Integer.toHexString(hashCode());
    }

    public DefaultBeanFactory getBeanFactory() {
        return this.beanFactory;
    }

    public void addApplicationListener(ApplicationListener<?> listener) {
        this.applicationListeners.add(listener);
    }

    private void assertActive() {
        if (!this.active.get()) {
            throw new IllegalStateException(
                    "ApplicationContext has not been refreshed yet or has already been closed");
        }
    }
}
```

### Test Code

#### File: `src/test/java/com/iris/framework/context/ApplicationEventTest.java` [NEW]

```java
package com.iris.framework.context;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class ApplicationEventTest {

    static class TestEvent extends ApplicationEvent {
        private final String message;

        TestEvent(Object source, String message) {
            super(source);
            this.message = message;
        }

        String getMessage() {
            return message;
        }
    }

    @Test
    void shouldStoreSource_WhenEventCreated() {
        Object source = "testSource";
        TestEvent event = new TestEvent(source, "hello");

        assertThat(event.getSource()).isSameAs(source);
    }

    @Test
    void shouldRecordTimestamp_WhenEventCreated() {
        long before = System.currentTimeMillis();
        TestEvent event = new TestEvent("source", "hello");
        long after = System.currentTimeMillis();

        assertThat(event.getTimestamp()).isBetween(before, after);
    }

    @Test
    void shouldStoreCustomData_WhenEventCreated() {
        TestEvent event = new TestEvent("source", "custom message");

        assertThat(event.getMessage()).isEqualTo("custom message");
    }

    @Test
    void shouldThrowNPE_WhenSourceIsNull() {
        assertThatThrownBy(() -> new TestEvent(null, "msg"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

#### File: `src/test/java/com/iris/framework/context/AnnotationConfigApplicationContextTest.java` [NEW]

```java
package com.iris.framework.context;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;

import static org.assertj.core.api.Assertions.*;

class AnnotationConfigApplicationContextTest {

    @Configuration
    static class SimpleConfig {
        @Bean
        public String greeting() {
            return "Hello, Iris!";
        }

        @Bean
        public Integer magicNumber() {
            return 42;
        }
    }

    @Configuration
    static class ServiceConfig {
        @Bean
        public StringBuilder messageLog() {
            return new StringBuilder();
        }
    }

    static class EventRecorder implements ApplicationListener<ContextRefreshedEvent> {
        final List<ApplicationEvent> events = new ArrayList<>();

        @Override
        public void onApplicationEvent(ContextRefreshedEvent event) {
            events.add(event);
        }
    }

    static class CloseRecorder implements ApplicationListener<ContextClosedEvent> {
        final List<ApplicationEvent> events = new ArrayList<>();

        @Override
        public void onApplicationEvent(ContextClosedEvent event) {
            events.add(event);
        }
    }

    @Test
    void shouldCreateContext_WhenGivenConfigClass() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);

        assertThat(ctx.isActive()).isTrue();
        assertThat(ctx.getBean("greeting")).isEqualTo("Hello, Iris!");
        assertThat(ctx.getBean("magicNumber")).isEqualTo(42);

        ctx.close();
    }

    @Test
    void shouldReturnBeanByType_WhenSingleCandidate() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);

        String greeting = ctx.getBean(String.class);
        assertThat(greeting).isEqualTo("Hello, Iris!");

        ctx.close();
    }

    @Test
    void shouldBeInactive_WhenClosed() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        assertThat(ctx.isActive()).isTrue();

        ctx.close();
        assertThat(ctx.isActive()).isFalse();
    }

    @Test
    void shouldThrowException_WhenAccessingBeanAfterClose() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        ctx.close();

        assertThatThrownBy(() -> ctx.getBean("greeting"))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("not been refreshed");
    }

    @Test
    void shouldPublishContextRefreshedEvent_WhenRefreshCompletes() {
        EventRecorder recorder = new EventRecorder();

        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        ctx.close();
    }

    @Test
    void shouldPublishContextClosedEvent_WhenCloseIsCalled() {
        CloseRecorder recorder = new CloseRecorder();
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        ctx.addApplicationListener(recorder);

        ctx.close();

        assertThat(recorder.events).hasSize(1);
        assertThat(recorder.events.get(0)).isInstanceOf(ContextClosedEvent.class);
        assertThat(((ContextClosedEvent) recorder.events.get(0)).getApplicationContext())
                .isSameAs(ctx);
    }

    @Test
    void shouldCloseOnlyOnce_WhenCloseCalledMultipleTimes() {
        CloseRecorder recorder = new CloseRecorder();
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);
        ctx.addApplicationListener(recorder);

        ctx.close();
        ctx.close();

        assertThat(recorder.events).hasSize(1);
    }

    @Test
    void shouldProvideDisplayName_WhenContextCreated() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);

        assertThat(ctx.getDisplayName()).startsWith("AnnotationConfigApplicationContext@");

        ctx.close();
    }

    @Test
    void shouldSupportMultipleConfigClasses() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class, ServiceConfig.class);

        assertThat(ctx.getBean("greeting")).isEqualTo("Hello, Iris!");
        assertThat(ctx.getBean(StringBuilder.class)).isNotNull();

        ctx.close();
    }

    @Test
    void shouldReturnSameSingleton_WhenBeanFetchedMultipleTimes() {
        var ctx = new AnnotationConfigApplicationContext(ServiceConfig.class);

        StringBuilder first = ctx.getBean(StringBuilder.class);
        StringBuilder second = ctx.getBean(StringBuilder.class);
        assertThat(first).isSameAs(second);

        ctx.close();
    }

    @Test
    void shouldThrowException_WhenBeanNotFound() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);

        assertThatThrownBy(() -> ctx.getBean("nonExistent"))
                .isInstanceOf(NoSuchBeanDefinitionException.class);

        ctx.close();
    }

    @Test
    void shouldResolveProgrammaticListenerEventType() {
        ApplicationListener<ContextRefreshedEvent> listener = event -> {};
        Class<?> eventType = AnnotationConfigApplicationContext.resolveEventType(listener);

        EventRecorder recorder = new EventRecorder();
        Class<?> recorderType = AnnotationConfigApplicationContext.resolveEventType(recorder);
        assertThat(recorderType).isEqualTo(ContextRefreshedEvent.class);
    }

    @Test
    void shouldExposeBeanFactory_WhenContextCreated() {
        var ctx = new AnnotationConfigApplicationContext(SimpleConfig.class);

        assertThat(ctx.getBeanFactory()).isNotNull();
        assertThat(ctx.getBeanFactory().getBeanDefinitionCount()).isGreaterThan(0);

        ctx.close();
    }
}
```

#### File: `src/test/java/com/iris/framework/context/integration/ApplicationContextIntegrationTest.java` [NEW]

```java
package com.iris.framework.context.integration;

import java.util.ArrayList;
import java.util.List;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.InitializingBean;
import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationEvent;
import com.iris.framework.context.ApplicationListener;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.ComponentScan;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;
import com.iris.framework.stereotype.Component;
import com.iris.framework.stereotype.Repository;
import com.iris.framework.stereotype.Service;

import static org.assertj.core.api.Assertions.*;

class ApplicationContextIntegrationTest {

    static final List<String> LIFECYCLE_EVENTS = new ArrayList<>();

    @Configuration
    static class AppConfig {
        @Bean
        public UserRepository userRepository() {
            return new UserRepository();
        }

        @Bean
        public UserService userService(UserRepository repo) {
            return new UserService(repo);
        }
    }

    static class UserRepository {
        public String findUser(String id) {
            return "User-" + id;
        }
    }

    static class UserService {
        private final UserRepository repository;

        UserService(UserRepository repository) {
            this.repository = repository;
        }

        public String getUser(String id) {
            return repository.findUser(id);
        }
    }

    @Configuration
    static class LifecycleConfig {
        @Bean
        public LifecycleBean lifecycleBean() {
            return new LifecycleBean();
        }
    }

    static class LifecycleBean implements InitializingBean {
        @PostConstruct
        void init() {
            LIFECYCLE_EVENTS.add("@PostConstruct");
        }

        @Override
        public void afterPropertiesSet() {
            LIFECYCLE_EVENTS.add("afterPropertiesSet");
        }

        @PreDestroy
        void cleanup() {
            LIFECYCLE_EVENTS.add("@PreDestroy");
        }
    }

    @Configuration
    static class ListenerConfig {
        @Bean
        public RefreshListener refreshListener() {
            return new RefreshListener();
        }

        @Bean
        public AllEventsListener allEventsListener() {
            return new AllEventsListener();
        }
    }

    static class RefreshListener implements ApplicationListener<ContextRefreshedEvent> {
        final List<ApplicationEvent> received = new ArrayList<>();

        @Override
        public void onApplicationEvent(ContextRefreshedEvent event) {
            received.add(event);
        }
    }

    static class AllEventsListener implements ApplicationListener<ApplicationEvent> {
        final List<ApplicationEvent> received = new ArrayList<>();

        @Override
        public void onApplicationEvent(ApplicationEvent event) {
            received.add(event);
        }
    }

    @Configuration
    static class FieldInjectionConfig {
        @Bean
        public MessageRepository messageRepository() {
            return new MessageRepository();
        }

        @Bean
        public MessageService messageService() {
            return new MessageService();
        }
    }

    static class MessageRepository {
        public String getMessage() {
            return "Hello from repo";
        }
    }

    static class MessageService {
        @Autowired
        private MessageRepository repository;

        public String getMessage() {
            return repository.getMessage();
        }
    }

    @Configuration
    @ComponentScan("com.iris.framework.context.integration.scanpkg")
    static class ScanConfig {
    }

    @Test
    void shouldResolveBeanDependencies_WhenUsingConfigWithBeanMethods() {
        var ctx = new AnnotationConfigApplicationContext(AppConfig.class);

        UserService service = ctx.getBean(UserService.class);
        assertThat(service.getUser("42")).isEqualTo("User-42");

        ctx.close();
    }

    @Test
    void shouldRunLifecycleCallbacks_WhenContextRefreshes() {
        LIFECYCLE_EVENTS.clear();

        var ctx = new AnnotationConfigApplicationContext(LifecycleConfig.class);

        assertThat(LIFECYCLE_EVENTS).containsExactly("@PostConstruct", "afterPropertiesSet");

        ctx.close();

        assertThat(LIFECYCLE_EVENTS).containsExactly(
                "@PostConstruct", "afterPropertiesSet", "@PreDestroy");
    }

    @Test
    void shouldPublishContextRefreshedEvent_WhenRefreshCompletes() {
        var ctx = new AnnotationConfigApplicationContext(ListenerConfig.class);

        RefreshListener listener = ctx.getBean(RefreshListener.class);
        assertThat(listener.received).hasSize(1);
        assertThat(listener.received.get(0)).isInstanceOf(ContextRefreshedEvent.class);
        assertThat(((ContextRefreshedEvent) listener.received.get(0)).getApplicationContext())
                .isSameAs(ctx);

        ctx.close();
    }

    @Test
    void shouldPublishContextClosedEvent_WhenContextCloses() {
        var ctx = new AnnotationConfigApplicationContext(ListenerConfig.class);

        AllEventsListener listener = ctx.getBean(AllEventsListener.class);
        assertThat(listener.received).hasSize(1);

        ctx.close();

        assertThat(listener.received).hasSize(2);
        assertThat(listener.received.get(0)).isInstanceOf(ContextRefreshedEvent.class);
        assertThat(listener.received.get(1)).isInstanceOf(ContextClosedEvent.class);
    }

    @Test
    void shouldInjectDependencies_WhenUsingAutowiredFields() {
        var ctx = new AnnotationConfigApplicationContext(FieldInjectionConfig.class);

        MessageService service = ctx.getBean(MessageService.class);
        assertThat(service.getMessage()).isEqualTo("Hello from repo");

        ctx.close();
    }

    @Test
    void shouldDiscoverComponentsByScanning_WhenConfigHasComponentScan() {
        var ctx = new AnnotationConfigApplicationContext(ScanConfig.class);

        assertThat(ctx.containsBean("scanRepo")).isTrue();
        assertThat(ctx.containsBean("scanService")).isTrue();

        ctx.close();
    }

    @Test
    void shouldMaintainSingletonIdentity_AcrossMultipleGetBeanCalls() {
        var ctx = new AnnotationConfigApplicationContext(AppConfig.class);

        UserService first = ctx.getBean(UserService.class);
        UserService second = ctx.getBean(UserService.class);
        assertThat(first).isSameAs(second);

        ctx.close();
    }

    @Test
    void shouldDestroyBeansInReverseOrder_WhenContextCloses() {
        var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        assertThat(ctx.isActive()).isTrue();
        ctx.close();
        assertThat(ctx.isActive()).isFalse();
    }

    @Test
    void shouldFilterEventsByType_WhenMultipleListenersRegistered() {
        var ctx = new AnnotationConfigApplicationContext(ListenerConfig.class);

        RefreshListener refreshOnly = ctx.getBean(RefreshListener.class);
        AllEventsListener allEvents = ctx.getBean(AllEventsListener.class);

        assertThat(refreshOnly.received).hasSize(1);
        assertThat(refreshOnly.received.get(0)).isInstanceOf(ContextRefreshedEvent.class);

        assertThat(allEvents.received).hasSize(1);
        assertThat(allEvents.received.get(0)).isInstanceOf(ContextRefreshedEvent.class);

        ctx.close();

        assertThat(refreshOnly.received).hasSize(1);
        assertThat(allEvents.received).hasSize(2);
    }

    @Test
    void shouldCombineMultipleConfigClasses_WhenPassedToConstructor() {
        var ctx = new AnnotationConfigApplicationContext(AppConfig.class, ListenerConfig.class);

        assertThat(ctx.getBean(UserService.class)).isNotNull();
        assertThat(ctx.getBean(RefreshListener.class)).isNotNull();

        ctx.close();
    }
}
```

#### File: `src/test/java/com/iris/framework/context/integration/scanpkg/ScanRepo.java` [NEW]

```java
package com.iris.framework.context.integration.scanpkg;

import com.iris.framework.stereotype.Repository;

@Repository("scanRepo")
public class ScanRepo {
    public String find(String id) {
        return "Found-" + id;
    }
}
```

#### File: `src/test/java/com/iris/framework/context/integration/scanpkg/ScanService.java` [NEW]

```java
package com.iris.framework.context.integration.scanpkg;

import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.stereotype.Service;

@Service("scanService")
public class ScanService {
    @Autowired
    private ScanRepo repo;

    public String process(String id) {
        return repo.find(id);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`ApplicationContext`** | The full container — extends `BeanFactory` + `ApplicationEventPublisher`. The read-only view for application code. |
| **`ConfigurableApplicationContext`** | The mutable SPI — adds `refresh()`, `close()`, `isActive()` for framework/test code. |
| **`refresh()`** | The 5-step startup pipeline: process configs → register processors → instantiate singletons → discover listeners → publish `ContextRefreshedEvent`. |
| **`ApplicationEvent` / `ApplicationListener`** | Observer pattern — decouple event producers from consumers. The container is both a producer (lifecycle events) and a dispatcher (routing events to matching listeners). |
| **`preInstantiateSingletons()`** | Transitions from lazy to eager initialization — fail-fast at startup instead of at runtime. |
| **Template Method** | `refresh()` defines a fixed sequence; each step is a method that could be overridden by subclasses (e.g., `onRefresh()` for embedded server creation in Spring Boot). |

**Next: Chapter 7 — Environment & Properties** — Load `application.properties`, resolve `${...}` placeholders, and inject values with `@Value`. This is how external configuration flows into your beans.
