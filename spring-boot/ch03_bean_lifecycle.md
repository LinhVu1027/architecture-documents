# Chapter 3: Bean Lifecycle

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Beans are constructed and injected with dependencies, but they have no way to perform setup after wiring or release resources on shutdown | A `DataSource` bean can't open a connection pool after injection, and there's no way to close it when the application stops | Build `@PostConstruct` / `@PreDestroy` annotation callbacks, `InitializingBean` / `DisposableBean` interface callbacks, and a `close()` method that tears down singletons in reverse order |

---

## 3.1 The Integration Point

The integration point is the same `DefaultBeanFactory.createBean()` method we expanded in Chapter 2 — but now the placeholder comment becomes real code. The lifecycle sits between the two BeanPostProcessor phases.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Replace the `// (Ch03: ...)` placeholder with a call to `invokeInitMethods()`, and add a `close()` method for singleton destruction.

```java
private Object createBean(String name, BeanDefinition bd) {
    try {
        // 1. Instantiate (supplier or constructor injection)
        Object bean;
        if (bd.getSupplier() != null) {
            bean = bd.getSupplier().get();
        } else {
            bean = instantiate(name, bd);
        }

        // 2. BeanPostProcessors — before initialization
        //    (@PostConstruct methods are invoked here via LifecycleBeanPostProcessor)
        bean = applyBeanPostProcessorsBeforeInitialization(bean, name);

        // 3. InitializingBean callback                          ← NEW
        invokeInitMethods(bean, name);

        // 4. BeanPostProcessors — after initialization
        bean = applyBeanPostProcessorsAfterInitialization(bean, name);

        return bean;
    } catch (BeanCreationException ex) {
        throw ex;
    } catch (Exception ex) {
        throw new BeanCreationException(name, "Instantiation of bean failed", ex);
    }
}
```

Two key decisions here:

1. **`@PostConstruct` fires inside `postProcessBeforeInitialization`, not in `invokeInitMethods`.** This is because `@PostConstruct` is handled by a `BeanPostProcessor` — the `LifecycleBeanPostProcessor`. The factory doesn't know about annotations; it only knows about the `InitializingBean` interface. This separation is exactly how real Spring works: `CommonAnnotationBeanPostProcessor` handles the annotations while `AbstractAutowireCapableBeanFactory.invokeInitMethods()` handles the interface callback.

2. **`invokeInitMethods` sits between the two BPP phases.** This means `@PostConstruct` runs first (in the "before" phase), then `InitializingBean.afterPropertiesSet()`, then AOP proxying would happen (in the "after" phase). This ordering guarantee is a core Spring contract.

This connects **initialization callbacks** to the **bean creation pipeline** and **destruction** to a new `close()` method. To make it work, we need to build:
- `InitializingBean` interface — the interface-based init callback
- `DisposableBean` interface — the interface-based destroy callback
- `LifecycleBeanPostProcessor` — scans for and invokes `@PostConstruct` / `@PreDestroy` annotations
- `close()` / `destroySingleton()` — tears down singletons in reverse order

The complete initialization and destruction ordering:

```
                    INITIALIZATION                              DESTRUCTION

  1. Constructor (instantiate)                     1. @PreDestroy (LifecycleBeanPostProcessor)
  2. @Autowired field injection (BPP before)       2. DisposableBean.destroy()
  3. @PostConstruct (BPP before, via LBPP)
  4. InitializingBean.afterPropertiesSet()           Singletons destroyed in reverse
  5. BPP after initialization                        registration order (LIFO)
  6. Cache in singletonObjects
```

## 3.2 InitializingBean and DisposableBean Interfaces

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/InitializingBean.java`

```java
package com.iris.framework.beans.factory;

public interface InitializingBean {

    void afterPropertiesSet() throws Exception;
}
```

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/DisposableBean.java`

```java
package com.iris.framework.beans.factory;

public interface DisposableBean {

    void destroy() throws Exception;
}
```

These are single-method interfaces — one for init, one for destroy. The `throws Exception` signature is intentional: Spring logs and wraps these exceptions rather than requiring checked exception handling at the call site.

## 3.3 LifecycleBeanPostProcessor

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/annotation/LifecycleBeanPostProcessor.java`

This is the annotation-based counterpart to the interface callbacks. It implements `BeanPostProcessor` to handle `@PostConstruct` during initialization, and provides a separate `postProcessBeforeDestruction()` method for `@PreDestroy` during shutdown.

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanPostProcessor;

public class LifecycleBeanPostProcessor implements BeanPostProcessor {

    private final Map<Class<?>, LifecycleMetadata> metadataCache = new ConcurrentHashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        for (Method method : metadata.initMethods()) {
            try {
                method.setAccessible(true);
                method.invoke(bean);
            } catch (InvocationTargetException ex) {
                throw new BeanCreationException(beanName,
                        "Invocation of @PostConstruct method '" + method.getName() + "' failed",
                        ex.getTargetException());
            } catch (IllegalAccessException ex) {
                throw new BeanCreationException(beanName,
                        "Failed to invoke @PostConstruct method '" + method.getName() + "'", ex);
            }
        }
        return bean;
    }

    public void postProcessBeforeDestruction(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        for (Method method : metadata.destroyMethods()) {
            try {
                method.setAccessible(true);
                method.invoke(bean);
            } catch (Exception ex) {
                // Log but don't rethrow — allow other beans to be destroyed
                System.err.println("Invocation of @PreDestroy method '"
                        + method.getName() + "' on bean '" + beanName + "' failed: " + ex);
            }
        }
    }

    private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
        return metadataCache.computeIfAbsent(clazz, this::buildLifecycleMetadata);
    }

    private LifecycleMetadata buildLifecycleMetadata(Class<?> clazz) {
        List<Method> initMethods = new ArrayList<>();
        List<Method> destroyMethods = new ArrayList<>();

        Class<?> targetClass = clazz;
        while (targetClass != null && targetClass != Object.class) {
            List<Method> currentInit = new ArrayList<>();
            List<Method> currentDestroy = new ArrayList<>();

            for (Method method : targetClass.getDeclaredMethods()) {
                if (method.isAnnotationPresent(PostConstruct.class)) {
                    if (method.getParameterCount() != 0) {
                        throw new IllegalStateException(
                                "@PostConstruct method must have no arguments: " + method);
                    }
                    currentInit.add(method);
                }
                if (method.isAnnotationPresent(PreDestroy.class)) {
                    if (method.getParameterCount() != 0) {
                        throw new IllegalStateException(
                                "@PreDestroy method must have no arguments: " + method);
                    }
                    currentDestroy.add(method);
                }
            }

            // Prepend: parent class methods end up at the beginning of the list
            initMethods.addAll(0, currentInit);
            destroyMethods.addAll(0, currentDestroy);

            targetClass = targetClass.getSuperclass();
        }

        return new LifecycleMetadata(
                List.copyOf(initMethods), List.copyOf(destroyMethods));
    }

    private record LifecycleMetadata(List<Method> initMethods, List<Method> destroyMethods) {
    }
}
```

Three design details to note:

1. **Metadata caching.** `buildLifecycleMetadata()` uses reflection to scan the class hierarchy — this is expensive. The `metadataCache` ensures we do it only once per class. The real Spring does the same in `InitDestroyAnnotationBeanPostProcessor.lifecycleMetadataCache`.

2. **Hierarchy walking with prepend.** The `while` loop walks from concrete class up to `Object`. Each level's methods are inserted at index 0 (`addAll(0, ...)`), so superclass `@PostConstruct` methods run first. This matches the real Spring's behavior.

3. **Error handling asymmetry.** `@PostConstruct` failures throw `BeanCreationException` (the app can't start with a broken bean). `@PreDestroy` failures are logged and swallowed (one bean's cleanup failure shouldn't prevent other beans from shutting down).

## 3.4 The close() Method — Singleton Destruction

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Add `invokeInitMethods()`, `close()`, and `destroySingleton()` methods.

```java
private void invokeInitMethods(Object bean, String beanName) throws Exception {
    if (bean instanceof InitializingBean initializingBean) {
        initializingBean.afterPropertiesSet();
    }
}

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
            System.err.println("Destroy method on bean '" + beanName
                    + "' threw exception: " + ex);
        }
    }
}
```

Key design decisions:

- **Reverse iteration** (`i = names.size() - 1; i >= 0; i--`) — beans registered last are destroyed first. If bean A was registered before bean B, B is likely a higher-level service that depends on A. Destroying B first ensures it can still use A during cleanup. The real Spring uses the same LIFO ordering in `DefaultSingletonBeanRegistry.destroySingletons()`.

- **Only created singletons are destroyed.** The loop checks `singletonObjects.get(name) != null` — beans that were registered but never requested (lazy init beans that nobody called `getBean()` on) are skipped.

- **`@PreDestroy` before `DisposableBean.destroy()`** — the same ordering that Spring guarantees. The annotation-based callback is considered more specific.

## 3.5 Build Configuration Update

**Modifying:** `iris-framework/build.gradle`
**Change:** Add Jakarta Annotation API dependency for `@PostConstruct` and `@PreDestroy`.

```gradle
dependencies {
    // Jakarta Annotation API — needed for @PostConstruct / @PreDestroy (Feature 3)
    implementation 'jakarta.annotation:jakarta.annotation-api:2.1.1'

    // Jakarta Servlet API — needed for DispatcherServlet (Feature 10)
    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    testImplementation 'jakarta.servlet:jakarta.servlet-api:6.1.0'
}
```

Note: this is `implementation` (not `compileOnly`) because `@PostConstruct` and `@PreDestroy` must be on the runtime classpath for reflection-based scanning to work.

## 3.6 Try It Yourself

<details>
<summary>Challenge: Implement the LifecycleBeanPostProcessor that invokes @PostConstruct methods</summary>

Given the `BeanPostProcessor` interface from Chapter 2, implement a processor that:
1. Scans the bean's class hierarchy for `@PostConstruct` methods
2. Invokes them in parent-first order
3. Caches the discovered methods so reflection only happens once per class

Think about: How do you handle private methods? What about methods that throw exceptions?

```java
public class LifecycleBeanPostProcessor implements BeanPostProcessor {

    private final Map<Class<?>, LifecycleMetadata> metadataCache = new ConcurrentHashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        for (Method method : metadata.initMethods()) {
            try {
                method.setAccessible(true);  // handles private methods
                method.invoke(bean);
            } catch (InvocationTargetException ex) {
                throw new BeanCreationException(beanName,
                        "Invocation of @PostConstruct method '" + method.getName() + "' failed",
                        ex.getTargetException());  // unwrap the real exception
            } catch (IllegalAccessException ex) {
                throw new BeanCreationException(beanName,
                        "Failed to invoke @PostConstruct method '" + method.getName() + "'", ex);
            }
        }
        return bean;
    }
    // ...
}
```

The key insight: `InvocationTargetException` wraps the actual exception thrown by the `@PostConstruct` method. Unwrapping it with `ex.getTargetException()` gives the user a clear stack trace.

</details>

<details>
<summary>Challenge: Implement close() with reverse-order destruction</summary>

The `close()` method must destroy singletons in reverse registration order and call both `@PreDestroy` and `DisposableBean.destroy()`. Think about: What happens to beans that were registered but never created? Should one bean's destruction failure prevent others from being destroyed?

```java
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
```

Uncreated singletons are skipped (they have no resources to release). Each bean's destruction is isolated — exceptions are caught and logged, never rethrown. This matches `DefaultSingletonBeanRegistry.destroyBean()` where "exceptions will get logged but not rethrown."

</details>

## 3.7 Tests

### Unit Tests — LifecycleBeanPostProcessor

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/annotation/LifecycleBeanPostProcessorTest.java`

```java
@Test
void shouldInvokePostConstructMethod_WhenAnnotated()            // basic @PostConstruct
@Test
void shouldNotInvokeAnyMethod_WhenNoAnnotations()               // passthrough for plain beans
@Test
void shouldInvokeParentPostConstructFirst_WhenInherited()        // hierarchy ordering
@Test
void shouldInvokePrivatePostConstruct_WhenAnnotated()            // private methods work
@Test
void shouldThrowBeanCreationException_WhenPostConstructFails()   // error handling
@Test
void shouldInvokePreDestroyMethod_WhenAnnotated()                // basic @PreDestroy
@Test
void shouldInvokeParentPreDestroyFirst_WhenInherited()           // hierarchy ordering
@Test
void shouldCacheMetadata_WhenSameClassProcessedTwice()           // metadata caching
```

### Integration Tests — Bean Lifecycle

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/integration/BeanLifecycleIntegrationTest.java`

```java
@Test
void shouldCallPostConstructAfterInjection_WhenBeanHasDependencies()   // can use injected deps
@Test
void shouldCallInitializingBeanAfterPostConstruct_WhenBothPresent()    // ordering guarantee
@Test
void shouldCallAfterPropertiesSet_WhenOnlyInitializingBean()           // interface-only
@Test
void shouldCallPostConstruct_WhenOnlyAnnotation()                      // annotation-only
@Test
void shouldCallPreDestroyBeforeDisposableBean_WhenBothPresent()        // destruction ordering
@Test
void shouldCallDisposableBeanDestroy_WhenOnlyInterface()               // interface-only destruction
@Test
void shouldCallPreDestroy_WhenOnlyAnnotation()                         // annotation-only destruction
@Test
void shouldDestroyInReverseOrder_WhenCloseCalled()                     // LIFO destruction
@Test
void shouldCallFullLifecycle_WhenBeanImplementsAllCallbacks()          // complete lifecycle
@Test
void shouldNotDestroyUncreatedSingletons_WhenCloseCalled()             // lazy beans skipped
```

**Run:** `./gradlew :iris-framework:test` — expected: all 59 tests pass (41 from Ch01–02 + 18 new)

---

## 3.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why two callback mechanisms (annotations vs interfaces)?** `InitializingBean`/`DisposableBean` came first (Spring 1.0, 2004). `@PostConstruct`/`@PreDestroy` came later (JSR-250, adopted in Spring 2.5, 2007). The interfaces couple your code to Spring's API; the annotations are standard Jakarta EE and work with any compliant container. Spring kept both for backward compatibility, but the community recommends annotations for application code and reserves the interfaces for framework-internal components where the compile-time contract matters.
> - **When to use which:** If you're writing a library that must work outside Spring (e.g., in a CDI container), use `@PostConstruct`/`@PreDestroy`. If you're writing infrastructure code that IS part of the framework, `InitializingBean`/`DisposableBean` give you type-safe contracts without annotation scanning overhead.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why destroy in reverse order?** If bean A is registered before bean B, B might depend on A (it was created later, likely using A as a dependency). Destroying B first ensures it can still reference A during cleanup (e.g., closing a connection that was obtained from a pool). This is the same principle as stack unwinding — last in, first out. The real Spring's `DefaultSingletonBeanRegistry.destroySingletons()` iterates `disposableBeanNames.length - 1` down to `0`.
> - **Caveat:** This LIFO heuristic works for simple dependency chains but doesn't handle complex graphs. The real Spring also tracks `dependentBeanMap` (which beans depend on which) and recursively destroys dependents first, which handles cases where registration order doesn't match dependency order.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why is `@PostConstruct` in `postProcessBeforeInitialization` and not a separate step?** Spring's architecture principle: the factory only knows about interfaces (`InitializingBean`, `DisposableBean`). Everything annotation-based is handled by `BeanPostProcessor` implementations. This means new annotation types can be added without modifying the core factory — you just register a new post-processor. `CommonAnnotationBeanPostProcessor` handles `@PostConstruct`/`@PreDestroy`, `AutowiredAnnotationBeanPostProcessor` handles `@Autowired`, and future processors could handle entirely new annotations. The factory remains closed for modification but open for extension (OCP).
> -----------------------------------------------------------

## 3.9 What We Enhanced

| Aspect | Before (ch02) | Current (ch03) | Real Framework |
|--------|---------------|-----------------|----------------|
| **Init callbacks** | None — beans had no way to perform setup after injection | `@PostConstruct` + `InitializingBean.afterPropertiesSet()` with guaranteed ordering | Same + custom init methods via `@Bean(initMethod="...")`, `Aware` callbacks (`BeanNameAware`, `BeanFactoryAware`), and `SmartInitializingSingleton` for post-all-singletons callback |
| **Destroy callbacks** | None — singletons lived forever, resources leaked | `@PreDestroy` + `DisposableBean.destroy()` with guaranteed ordering | Same + `AutoCloseable.close()` auto-detection, custom destroy methods via `@Bean(destroyMethod="...")`, dependent bean destruction ordering via `dependentBeanMap` |
| **Shutdown** | No shutdown mechanism — calling `getBean()` worked until the JVM exited | `close()` method destroys all created singletons in reverse registration order | `AbstractApplicationContext.close()` → `destroyBeans()` → `DefaultSingletonBeanRegistry.destroySingletons()` with `singletonsCurrentlyInDestruction` flag, `DisposableBeanAdapter` sequencing, and shutdown hook registration |
| **createBean pipeline** | 3 steps: instantiate → BPP before → BPP after (with placeholder comment) | 4 steps: instantiate → BPP before (@PostConstruct) → invokeInitMethods (InitializingBean) → BPP after | `initializeBean()` has 4 sub-steps: `invokeAwareMethods()` → `applyBPPBeforeInit` → `invokeInitMethods` → `applyBPPAfterInit` (`AbstractAutowireCapableBeanFactory.java:1799-1824`) |

## 3.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `InitializingBean` | `InitializingBean` | `InitializingBean.java:1` | Identical single-method interface |
| `DisposableBean` | `DisposableBean` | `DisposableBean.java:1` | Identical single-method interface |
| `LifecycleBeanPostProcessor` | `CommonAnnotationBeanPostProcessor` → extends `InitDestroyAnnotationBeanPostProcessor` | `CommonAnnotationBeanPostProcessor.java:190` | Real is a two-class hierarchy; also handles `@Resource` injection; uses `LifecycleElement` / `LifecycleMetadata` inner classes with deduplication against externally managed methods |
| `buildLifecycleMetadata()` | `InitDestroyAnnotationBeanPostProcessor.buildLifecycleMetadata()` | `InitDestroyAnnotationBeanPostProcessor.java:281` | Real checks `hasAnyExternallyManagedInitMethod()` to prevent double-invocation when a `@PostConstruct` method is also the custom init method |
| `invokeInitMethods()` | `AbstractAutowireCapableBeanFactory.invokeInitMethods()` | `AbstractAutowireCapableBeanFactory.java:1856` | Real also invokes custom init methods from `@Bean(initMethod="...")` and handles deduplication if `afterPropertiesSet` is also a custom init method name |
| `close()` | `DefaultSingletonBeanRegistry.destroySingletons()` | `DefaultSingletonBeanRegistry.java:693` | Real sets `singletonsCurrentlyInDestruction` flag, uses `DisposableBeanAdapter` to sequence all destruction steps, handles `dependentBeanMap` for dependency-ordered destruction |
| `destroySingleton()` | `DisposableBeanAdapter.destroy()` | `DisposableBeanAdapter.java:197` | Real invokes `DestructionAwareBeanPostProcessor.postProcessBeforeDestruction()`, handles `AutoCloseable.close()` auto-detection, supports parameterized destroy methods |

## 3.11 Complete Code

### Production Code

#### File: `iris-framework/build.gradle` [MODIFIED]

```gradle
// iris-framework: Simplified Spring Framework (IoC container, DI, component scanning, MVC)
// No external dependencies beyond the JDK and Jakarta Servlet API (added when needed)

dependencies {
    // Jakarta Annotation API — needed for @PostConstruct / @PreDestroy (Feature 3)
    implementation 'jakarta.annotation:jakarta.annotation-api:2.1.1'

    // Jakarta Servlet API — needed for DispatcherServlet (Feature 10)
    // compileOnly because the embedded Tomcat in iris-boot-core provides the runtime impl
    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    testImplementation 'jakarta.servlet:jakarta.servlet-api:6.1.0'
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/InitializingBean.java` [NEW]

```java
package com.iris.framework.beans.factory;

/**
 * Interface to be implemented by beans that need to perform initialization
 * logic after all their properties have been set by the container.
 *
 * <p>The container calls {@link #afterPropertiesSet()} after dependency injection
 * is complete and after {@code @PostConstruct} methods have been invoked, but
 * before {@code BeanPostProcessor.postProcessAfterInitialization()}.
 *
 * <p>The recommended alternative is {@code @PostConstruct}, which doesn't
 * couple your code to the framework. {@code InitializingBean} exists for cases
 * where you want the compile-time guarantee of an interface contract.
 *
 * @see DisposableBean
 * @see org.springframework.beans.factory.InitializingBean
 */
public interface InitializingBean {

    /**
     * Invoked by the containing {@link BeanFactory} after it has set all
     * bean properties and satisfied all dependency injection.
     *
     * @throws Exception in the event of misconfiguration or initialization failure
     */
    void afterPropertiesSet() throws Exception;
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/DisposableBean.java` [NEW]

```java
package com.iris.framework.beans.factory;

/**
 * Interface to be implemented by beans that want to release resources on
 * destruction. The container calls {@link #destroy()} when the singleton
 * is being torn down during {@code DefaultBeanFactory.close()}.
 *
 * <p>Called after {@code @PreDestroy} methods. Exceptions are logged but
 * not rethrown, so one bean's failure doesn't prevent other beans from
 * being destroyed.
 *
 * <p>The recommended alternative is {@code @PreDestroy}, which doesn't
 * couple your code to the framework.
 *
 * @see InitializingBean
 * @see org.springframework.beans.factory.DisposableBean
 */
public interface DisposableBean {

    /**
     * Invoked by the containing {@link BeanFactory} on destruction of a singleton.
     *
     * @throws Exception in the event of shutdown errors — implementations
     *         should catch and log where possible rather than throwing
     */
    void destroy() throws Exception;
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/annotation/LifecycleBeanPostProcessor.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanPostProcessor;

/**
 * {@link BeanPostProcessor} that invokes {@link PostConstruct @PostConstruct}
 * methods during initialization and {@link PreDestroy @PreDestroy} methods
 * during destruction.
 *
 * <p>Maps to Spring's {@code CommonAnnotationBeanPostProcessor} (which extends
 * {@code InitDestroyAnnotationBeanPostProcessor}). We simplify the two-class
 * hierarchy into a single class that directly scans for the Jakarta annotations.
 *
 * <p>Lifecycle metadata is discovered once per class and cached. The class
 * hierarchy is walked from concrete class up to {@code Object}, with parent
 * class methods prepended so that superclass callbacks run first — matching
 * Spring's behavior.
 *
 * @see PostConstruct
 * @see PreDestroy
 * @see org.springframework.context.annotation.CommonAnnotationBeanPostProcessor
 * @see org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor
 */
public class LifecycleBeanPostProcessor implements BeanPostProcessor {

    /** Cache of LifecycleMetadata per bean class — avoids repeated reflection. */
    private final Map<Class<?>, LifecycleMetadata> metadataCache = new ConcurrentHashMap<>();

    /**
     * Invoke all {@code @PostConstruct} methods on the given bean.
     * Runs during the "before initialization" phase — after dependency injection
     * but before {@code InitializingBean.afterPropertiesSet()}.
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        for (Method method : metadata.initMethods()) {
            try {
                method.setAccessible(true);
                method.invoke(bean);
            } catch (InvocationTargetException ex) {
                throw new BeanCreationException(beanName,
                        "Invocation of @PostConstruct method '" + method.getName() + "' failed",
                        ex.getTargetException());
            } catch (IllegalAccessException ex) {
                throw new BeanCreationException(beanName,
                        "Failed to invoke @PostConstruct method '" + method.getName() + "'",
                        ex);
            }
        }
        return bean;
    }

    /**
     * Invoke all {@code @PreDestroy} methods on the given bean.
     * Called by {@code DefaultBeanFactory.close()} before
     * {@code DisposableBean.destroy()}.
     *
     * <p>Exceptions are caught and logged rather than rethrown, so that
     * one bean's cleanup failure doesn't prevent other beans from being destroyed.
     */
    public void postProcessBeforeDestruction(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        for (Method method : metadata.destroyMethods()) {
            try {
                method.setAccessible(true);
                method.invoke(bean);
            } catch (Exception ex) {
                System.err.println("Invocation of @PreDestroy method '"
                        + method.getName() + "' on bean '" + beanName + "' failed: " + ex);
            }
        }
    }

    private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
        return metadataCache.computeIfAbsent(clazz, this::buildLifecycleMetadata);
    }

    /**
     * Walk the class hierarchy (concrete → Object) and collect annotated methods.
     * Each level's methods are prepended so that superclass callbacks run first.
     *
     * <p>Matches the real Spring's {@code InitDestroyAnnotationBeanPostProcessor
     * .buildLifecycleMetadata()} which uses the same prepend strategy at
     * {@code InitDestroyAnnotationBeanPostProcessor.java:281-322}.
     */
    private LifecycleMetadata buildLifecycleMetadata(Class<?> clazz) {
        List<Method> initMethods = new ArrayList<>();
        List<Method> destroyMethods = new ArrayList<>();

        Class<?> targetClass = clazz;
        while (targetClass != null && targetClass != Object.class) {
            List<Method> currentInit = new ArrayList<>();
            List<Method> currentDestroy = new ArrayList<>();

            for (Method method : targetClass.getDeclaredMethods()) {
                if (method.isAnnotationPresent(PostConstruct.class)) {
                    if (method.getParameterCount() != 0) {
                        throw new IllegalStateException(
                                "@PostConstruct method must have no arguments: " + method);
                    }
                    currentInit.add(method);
                }
                if (method.isAnnotationPresent(PreDestroy.class)) {
                    if (method.getParameterCount() != 0) {
                        throw new IllegalStateException(
                                "@PreDestroy method must have no arguments: " + method);
                    }
                    currentDestroy.add(method);
                }
            }

            // Prepend: parent class methods end up at the beginning of the list
            initMethods.addAll(0, currentInit);
            destroyMethods.addAll(0, currentDestroy);

            targetClass = targetClass.getSuperclass();
        }

        return new LifecycleMetadata(
                List.copyOf(initMethods),
                List.copyOf(destroyMethods));
    }

    /**
     * Cached lifecycle metadata for a bean class — holds the ordered lists
     * of {@code @PostConstruct} and {@code @PreDestroy} methods.
     */
    private record LifecycleMetadata(List<Method> initMethods, List<Method> destroyMethods) {
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java` [MODIFIED]

```java
package com.iris.framework.beans.factory.support;

import java.lang.reflect.Constructor;
import java.util.ArrayList;
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
            // 1. Instantiate (supplier or constructor injection)
            Object bean;
            if (bd.getSupplier() != null) {
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

### Test Code

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/annotation/LifecycleBeanPostProcessorTest.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import java.util.ArrayList;
import java.util.List;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

import com.iris.framework.beans.factory.BeanCreationException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link LifecycleBeanPostProcessor} — tests the processor
 * in isolation by calling its methods directly (not through DefaultBeanFactory).
 */
class LifecycleBeanPostProcessorTest {

    private LifecycleBeanPostProcessor processor;

    @BeforeEach
    void setUp() {
        processor = new LifecycleBeanPostProcessor();
    }

    // ── Helper beans ──────────────────────────────────────────────

    static class BeanWithPostConstruct {
        boolean initialized = false;

        @PostConstruct
        void init() {
            initialized = true;
        }
    }

    static class BeanWithPreDestroy {
        boolean destroyed = false;

        @PreDestroy
        void cleanup() {
            destroyed = true;
        }
    }

    static class BeanWithBothCallbacks {
        final List<String> events = new ArrayList<>();

        @PostConstruct
        void init() {
            events.add("postConstruct");
        }

        @PreDestroy
        void cleanup() {
            events.add("preDestroy");
        }
    }

    static class PlainBean {
        String value = "plain";
    }

    static class ParentBean {
        final List<String> events = new ArrayList<>();

        @PostConstruct
        void parentInit() {
            events.add("parent-init");
        }

        @PreDestroy
        void parentCleanup() {
            events.add("parent-destroy");
        }
    }

    static class ChildBean extends ParentBean {
        @PostConstruct
        void childInit() {
            events.add("child-init");
        }

        @PreDestroy
        void childCleanup() {
            events.add("child-destroy");
        }
    }

    static class BeanWithFailingPostConstruct {
        @PostConstruct
        void init() {
            throw new RuntimeException("init failed");
        }
    }

    static class BeanWithPrivatePostConstruct {
        boolean initialized = false;

        @PostConstruct
        private void init() {
            initialized = true;
        }
    }

    // ── @PostConstruct tests ─────────────────────────────────────

    @Test
    void shouldInvokePostConstructMethod_WhenAnnotated() {
        BeanWithPostConstruct bean = new BeanWithPostConstruct();

        processor.postProcessBeforeInitialization(bean, "testBean");

        assertThat(bean.initialized).isTrue();
    }

    @Test
    void shouldNotInvokeAnyMethod_WhenNoAnnotations() {
        PlainBean bean = new PlainBean();

        Object result = processor.postProcessBeforeInitialization(bean, "plainBean");

        assertThat(result).isSameAs(bean);
        assertThat(bean.value).isEqualTo("plain");
    }

    @Test
    void shouldInvokeParentPostConstructFirst_WhenInherited() {
        ChildBean bean = new ChildBean();

        processor.postProcessBeforeInitialization(bean, "childBean");

        assertThat(bean.events).containsExactly("parent-init", "child-init");
    }

    @Test
    void shouldInvokePrivatePostConstruct_WhenAnnotated() {
        BeanWithPrivatePostConstruct bean = new BeanWithPrivatePostConstruct();

        processor.postProcessBeforeInitialization(bean, "testBean");

        assertThat(bean.initialized).isTrue();
    }

    @Test
    void shouldThrowBeanCreationException_WhenPostConstructFails() {
        BeanWithFailingPostConstruct bean = new BeanWithFailingPostConstruct();

        assertThatThrownBy(() ->
                processor.postProcessBeforeInitialization(bean, "failingBean"))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("@PostConstruct")
                .hasMessageContaining("init");
    }

    // ── @PreDestroy tests ────────────────────────────────────────

    @Test
    void shouldInvokePreDestroyMethod_WhenAnnotated() {
        BeanWithPreDestroy bean = new BeanWithPreDestroy();

        processor.postProcessBeforeDestruction(bean, "testBean");

        assertThat(bean.destroyed).isTrue();
    }

    @Test
    void shouldInvokeParentPreDestroyFirst_WhenInherited() {
        ChildBean bean = new ChildBean();

        processor.postProcessBeforeDestruction(bean, "childBean");

        assertThat(bean.events).containsExactly("parent-destroy", "child-destroy");
    }

    // ── Metadata caching ─────────────────────────────────────────

    @Test
    void shouldCacheMetadata_WhenSameClassProcessedTwice() {
        BeanWithPostConstruct bean1 = new BeanWithPostConstruct();
        BeanWithPostConstruct bean2 = new BeanWithPostConstruct();

        processor.postProcessBeforeInitialization(bean1, "bean1");
        processor.postProcessBeforeInitialization(bean2, "bean2");

        // Both beans should be initialized — proves the cached metadata works
        assertThat(bean1.initialized).isTrue();
        assertThat(bean2.initialized).isTrue();
    }
}
```

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/integration/BeanLifecycleIntegrationTest.java` [NEW]

```java
package com.iris.framework.beans.factory.integration;

import java.util.ArrayList;
import java.util.List;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

import com.iris.framework.beans.factory.DisposableBean;
import com.iris.framework.beans.factory.InitializingBean;
import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for Bean Lifecycle — tests how @PostConstruct, @PreDestroy,
 * InitializingBean, DisposableBean, and dependency injection work together
 * through the full {@link DefaultBeanFactory} lifecycle.
 */
class BeanLifecycleIntegrationTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        factory.addBeanPostProcessor(new AutowiredBeanPostProcessor(factory));
        factory.addBeanPostProcessor(new LifecycleBeanPostProcessor());
    }

    // ── Shared event log ─────────────────────────────────────────
    // Static list so inner beans can record events across the test
    static final List<String> events = new ArrayList<>();

    @BeforeEach
    void clearEvents() {
        events.clear();
    }

    // ── Helper beans ──────────────────────────────────────────────

    static class Repository {
        String getData() {
            return "data";
        }
    }

    /**
     * Bean that uses @PostConstruct to validate injected dependencies.
     */
    static class ServiceWithPostConstruct {
        @Autowired
        Repository repository;

        String cachedData;

        @PostConstruct
        void init() {
            // @PostConstruct can use injected dependencies
            cachedData = repository.getData() + "-cached";
        }
    }

    /**
     * Bean that implements all four lifecycle callbacks.
     * Records event ordering in the static events list.
     */
    static class FullLifecycleBean implements InitializingBean, DisposableBean {
        final String name;

        FullLifecycleBean(String name) {
            this.name = name;
        }

        @PostConstruct
        void postConstruct() {
            events.add(name + ":postConstruct");
        }

        @Override
        public void afterPropertiesSet() {
            events.add(name + ":afterPropertiesSet");
        }

        @PreDestroy
        void preDestroy() {
            events.add(name + ":preDestroy");
        }

        @Override
        public void destroy() {
            events.add(name + ":destroy");
        }
    }

    /**
     * Bean that only implements InitializingBean (no annotations).
     */
    static class InitializingOnlyBean implements InitializingBean {
        boolean initialized = false;

        @Override
        public void afterPropertiesSet() {
            initialized = true;
        }
    }

    /**
     * Bean that only implements DisposableBean (no annotations).
     */
    static class DisposableOnlyBean implements DisposableBean {
        boolean destroyed = false;

        @Override
        public void destroy() {
            destroyed = true;
        }
    }

    /**
     * Bean with only annotation-based callbacks.
     */
    static class AnnotationOnlyBean {
        boolean initialized = false;
        boolean destroyed = false;

        @PostConstruct
        void init() {
            initialized = true;
        }

        @PreDestroy
        void cleanup() {
            destroyed = true;
        }
    }

    // ── Initialization tests ─────────────────────────────────────

    @Test
    void shouldCallPostConstructAfterInjection_WhenBeanHasDependencies() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(Repository.class));
        factory.registerBeanDefinition("service",
                new BeanDefinition(ServiceWithPostConstruct.class));

        ServiceWithPostConstruct service = factory.getBean("service",
                ServiceWithPostConstruct.class);

        assertThat(service.repository).isNotNull();
        assertThat(service.cachedData).isEqualTo("data-cached");
    }

    @Test
    void shouldCallInitializingBeanAfterPostConstruct_WhenBothPresent() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("A")));

        factory.getBean("bean");

        assertThat(events).containsExactly(
                "A:postConstruct",
                "A:afterPropertiesSet");
    }

    @Test
    void shouldCallAfterPropertiesSet_WhenOnlyInitializingBean() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(InitializingOnlyBean.class));

        InitializingOnlyBean bean = factory.getBean("bean",
                InitializingOnlyBean.class);

        assertThat(bean.initialized).isTrue();
    }

    @Test
    void shouldCallPostConstruct_WhenOnlyAnnotation() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(AnnotationOnlyBean.class));

        AnnotationOnlyBean bean = factory.getBean("bean",
                AnnotationOnlyBean.class);

        assertThat(bean.initialized).isTrue();
    }

    // ── Destruction tests ────────────────────────────────────────

    @Test
    void shouldCallPreDestroyBeforeDisposableBean_WhenBothPresent() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("A")));

        factory.getBean("bean"); // trigger creation
        events.clear();          // clear init events

        factory.close();

        assertThat(events).containsExactly(
                "A:preDestroy",
                "A:destroy");
    }

    @Test
    void shouldCallDisposableBeanDestroy_WhenOnlyInterface() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(DisposableOnlyBean.class));

        DisposableOnlyBean bean = factory.getBean("bean",
                DisposableOnlyBean.class);

        factory.close();

        assertThat(bean.destroyed).isTrue();
    }

    @Test
    void shouldCallPreDestroy_WhenOnlyAnnotation() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(AnnotationOnlyBean.class));

        AnnotationOnlyBean bean = factory.getBean("bean",
                AnnotationOnlyBean.class);

        factory.close();

        assertThat(bean.destroyed).isTrue();
    }

    @Test
    void shouldDestroyInReverseOrder_WhenCloseCalled() {
        factory.registerBeanDefinition("first",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("first")));
        factory.registerBeanDefinition("second",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("second")));
        factory.registerBeanDefinition("third",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("third")));

        // Create all singletons
        factory.getBean("first");
        factory.getBean("second");
        factory.getBean("third");
        events.clear();

        factory.close();

        // Destruction order: third → second → first (reverse of registration)
        assertThat(events).containsExactly(
                "third:preDestroy", "third:destroy",
                "second:preDestroy", "second:destroy",
                "first:preDestroy", "first:destroy");
    }

    // ── Full lifecycle test ──────────────────────────────────────

    @Test
    void shouldCallFullLifecycle_WhenBeanImplementsAllCallbacks() {
        factory.registerBeanDefinition("bean",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("A")));

        factory.getBean("bean");
        factory.close();

        assertThat(events).containsExactly(
                "A:postConstruct",
                "A:afterPropertiesSet",
                "A:preDestroy",
                "A:destroy");
    }

    @Test
    void shouldNotDestroyUncreatedSingletons_WhenCloseCalled() {
        factory.registerBeanDefinition("created",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("created")));
        factory.registerBeanDefinition("notCreated",
                new BeanDefinition(FullLifecycleBean.class,
                        () -> new FullLifecycleBean("notCreated")));

        factory.getBean("created"); // only create the first one
        events.clear();

        factory.close();

        // Only the created bean should be destroyed
        assertThat(events).containsExactly(
                "created:preDestroy",
                "created:destroy");
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@PostConstruct`** | Jakarta annotation marking a method to be called after dependency injection — runs during `postProcessBeforeInitialization()`, before `InitializingBean` |
| **`@PreDestroy`** | Jakarta annotation marking a method to be called before the singleton is destroyed — runs before `DisposableBean.destroy()` |
| **`InitializingBean`** | Framework interface with `afterPropertiesSet()` — a compile-time contract for initialization, called after `@PostConstruct` |
| **`DisposableBean`** | Framework interface with `destroy()` — a compile-time contract for cleanup, called after `@PreDestroy` |
| **`LifecycleBeanPostProcessor`** | The `BeanPostProcessor` that scans for `@PostConstruct`/`@PreDestroy` and invokes them — maps to Spring's `CommonAnnotationBeanPostProcessor` |
| **`close()`** | Destroys all created singletons in reverse registration order (LIFO) — the container's shutdown hook |
| **LIFO destruction** | Last registered = first destroyed — ensures higher-level beans are torn down before the infrastructure they depend on |

**Next: Chapter 4 — Annotation Configuration** — Beans can now be created, wired, and lifecycle-managed, but every bean must be registered manually with `registerBeanDefinition()`. We'll add `@Configuration` classes with `@Bean` factory methods so you can declare beans in Java code instead of imperative registration.
