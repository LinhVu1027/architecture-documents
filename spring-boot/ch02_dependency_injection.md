# Chapter 2: Dependency Injection

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Beans are created with no-arg constructors or suppliers — each bean is an isolated island with no awareness of other beans in the container | A `GreetingService` that needs a `GreetingRepository` must obtain it manually; the container can't wire them together | Build constructor injection and `@Autowired` field injection so the container automatically resolves and injects dependencies between beans |

---

## 2.1 The Integration Point

The integration point is `DefaultBeanFactory.createBean()`. In Chapter 1, this method was a simple 4-line bridge: call supplier or reflective constructor, return. Now it becomes the **orchestration hub** for the entire bean creation pipeline.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Expand `createBean()` into a 3-phase pipeline: instantiate → post-process before init → post-process after init. Add constructor injection in `instantiate()`.

```java
private Object createBean(String name, BeanDefinition bd) {
    try {
        // 1. Instantiate (supplier or constructor injection)
        Object bean;
        if (bd.getSupplier() != null) {
            bean = bd.getSupplier().get();
        } else {
            bean = instantiate(name, bd);   // ← NEW: constructor injection
        }

        // 2. BeanPostProcessors — before initialization
        bean = applyBeanPostProcessorsBeforeInitialization(bean, name);  // ← NEW

        // (Ch03: @PostConstruct / InitializingBean callbacks will go here)

        // 3. BeanPostProcessors — after initialization
        bean = applyBeanPostProcessorsAfterInitialization(bean, name);   // ← NEW

        return bean;
    } catch (BeanCreationException ex) {
        throw ex;
    } catch (Exception ex) {
        throw new BeanCreationException(name, "Instantiation of bean failed", ex);
    }
}
```

Two key decisions here:

1. **Constructor injection lives in the factory, not in a post-processor.** The bean must be constructed before any post-processor can inspect it. In the real Spring, constructor selection is delegated to `SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()`, but the actual invocation still happens in `AbstractAutowireCapableBeanFactory.createBeanInstance()` — inside the factory itself.

2. **The post-processor hook sits between construction and caching.** The `doGetBean()` method caches the singleton *after* `createBean()` returns, so post-processors see the fully-wired bean before it enters the singleton cache. This means a post-processor could even replace the bean with a proxy — and the proxy is what gets cached.

This connects **instantiation** to **dependency resolution** and **post-processing hooks**. To make it work, we need to build:
- `@Autowired` annotation — marks constructors and fields for injection
- `BeanPostProcessor` interface — the extensible hook for post-instantiation processing
- `AutowiredBeanPostProcessor` — the concrete processor that does field injection
- `instantiate()` method — constructor selection and parameter resolution

## 2.2 @Autowired Annotation

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/annotation/Autowired.java`

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.CONSTRUCTOR, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
}
```

The real `@Autowired` also targets `METHOD` and `PARAMETER`, and has a `required` attribute (defaulting to `true`). We only need constructor and field targets for our simplified version.

## 2.3 BeanPostProcessor Interface

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/config/BeanPostProcessor.java`

```java
package com.iris.framework.beans.factory.config;

public interface BeanPostProcessor {

    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

Both methods are `default` (return the bean unchanged), so implementations only need to override the hooks they care about. The methods return `Object` rather than `void` because a post-processor may **replace** the bean entirely — this is how AOP proxies are created in the real framework.

The `DefaultBeanFactory` gains a list of post-processors and a registration method:

```java
/** Ordered list of BeanPostProcessors to apply during bean creation. */
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();

public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
    this.beanPostProcessors.remove(beanPostProcessor);  // remove from old position
    this.beanPostProcessors.add(beanPostProcessor);       // add at end
}
```

The remove-then-add pattern mirrors `AbstractBeanFactory.addBeanPostProcessor()` — it prevents duplicates while ensuring the latest registration wins.

## 2.4 Constructor Injection — The `instantiate()` Method

**Modifying:** `DefaultBeanFactory.java`
**Change:** Extract instantiation logic into a new `instantiate()` method that supports three strategies.

```java
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
                                + paramTypes[i].getName() + "'", ex);
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
```

The three strategies, in priority order:
1. **`@Autowired` constructor** — explicitly marked. The real Spring enforces at most one `required=true` constructor.
2. **Single constructor with parameters** — implicit autowiring, a Spring 4.3+ feature. If a class has exactly one constructor and it takes parameters, Spring assumes you want them injected.
3. **No-arg constructor** — the original Chapter 1 behavior, now made accessible via `setAccessible(true)`.

The recursive call `getBean(paramTypes[i])` is the key insight: to create bean A, we ask the container for bean B, which might itself trigger creation of bean C. Dependency resolution is a depth-first walk through the bean graph.

## 2.5 AutowiredBeanPostProcessor — Field Injection

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/annotation/AutowiredBeanPostProcessor.java`

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.reflect.Field;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.config.BeanPostProcessor;

public class AutowiredBeanPostProcessor implements BeanPostProcessor {

    private final BeanFactory beanFactory;

    public AutowiredBeanPostProcessor(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        Class<?> targetClass = bean.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Field field : targetClass.getDeclaredFields()) {
                if (field.isAnnotationPresent(Autowired.class)) {
                    injectField(bean, beanName, field);
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return bean;
    }

    private void injectField(Object bean, String beanName, Field field) {
        try {
            Object dependency = beanFactory.getBean(field.getType());
            field.setAccessible(true);
            field.set(bean, dependency);
        } catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(beanName,
                    "Unsatisfied dependency for field '" + field.getName()
                            + "': No qualifying bean of type '"
                            + field.getType().getName() + "' available", ex);
        } catch (IllegalAccessException ex) {
            throw new BeanCreationException(beanName,
                    "Failed to inject @Autowired field '" + field.getName() + "'", ex);
        }
    }
}
```

Key design choices:
- **Class hierarchy walking:** The `while` loop walks from the concrete class up to `Object`, scanning each level's declared fields. This means `@Autowired` fields in a superclass are injected too — matching the real Spring's `buildAutowiringMetadata()`.
- **`postProcessBeforeInitialization` (not `after`):** Field injection happens *before* init callbacks so that `@PostConstruct` methods (Chapter 3) can use the injected dependencies.
- **Needs a `BeanFactory` reference:** The processor resolves dependencies via `beanFactory.getBean(field.getType())`. In the real Spring, `AutowiredAnnotationBeanPostProcessor` implements `BeanFactoryAware` to receive this reference.

## 2.6 Wiring It All Together

With all pieces in place, here's how a user creates a fully-wired container:

```java
// 1. Create the factory
DefaultBeanFactory factory = new DefaultBeanFactory();

// 2. Register the field-injection processor
factory.addBeanPostProcessor(new AutowiredBeanPostProcessor(factory));

// 3. Register bean definitions
factory.registerBeanDefinition("repository",
        new BeanDefinition(GreetingRepository.class));
factory.registerBeanDefinition("service",
        new BeanDefinition(GreetingService.class));

// 4. Get a bean — dependencies are resolved automatically
GreetingService service = factory.getBean("service", GreetingService.class);
service.greet("World"); // → "Hello, World!"
```

The creation flow for `GreetingService`:

```
getBean("service")
    │
    ├──> doGetBean("service")
    │     ├──> singletonObjects cache → miss
    │     └──> createBean("service", bd)
    │           │
    │           ├──> instantiate() → no-arg constructor (no @Autowired ctor)
    │           │
    │           ├──> applyBeanPostProcessorsBeforeInitialization()
    │           │     └──> AutowiredBeanPostProcessor
    │           │           └──> field: @Autowired GreetingRepository
    │           │                 └──> getBean(GreetingRepository.class)
    │           │                       └──> doGetBean("repository")
    │           │                             └──> createBean(...) → new GreetingRepository()
    │           │
    │           └──> applyBeanPostProcessorsAfterInitialization() → no-op
    │
    └──> cache in singletonObjects → return
```

## 2.7 Try It Yourself

<details>
<summary>Challenge: Implement constructor selection that picks the @Autowired constructor or the only constructor</summary>

Given a class with multiple constructors, only the `@Autowired`-annotated one should be selected. Given a class with a single parameterized constructor (no annotation), it should be autowired implicitly. Think about: what should happen when a class has multiple constructors and none is annotated?

```java
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

    // 3. Resolve and invoke
    if (selected != null) {
        Class<?>[] paramTypes = selected.getParameterTypes();
        Object[] args = new Object[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++) {
            args[i] = getBean(paramTypes[i]);
        }
        selected.setAccessible(true);
        return selected.newInstance(args);
    }

    // 4. Fallback: no-arg constructor
    Constructor<?> defaultCtor = clazz.getDeclaredConstructor();
    defaultCtor.setAccessible(true);
    return defaultCtor.newInstance();
}
```

When a class has multiple constructors and none is annotated, we fall through to the no-arg constructor. If there's no no-arg constructor either, `getDeclaredConstructor()` throws `NoSuchMethodException`, which gets wrapped in `BeanCreationException`. The real Spring has more sophisticated fallback logic, but this covers the common cases.

</details>

<details>
<summary>Challenge: Implement field injection that walks the class hierarchy</summary>

A `DerivedService extends BaseService` should have `@Autowired` fields in `BaseService` injected too. `getDeclaredFields()` only returns the current class's fields — you need to walk up the hierarchy.

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    Class<?> targetClass = bean.getClass();
    while (targetClass != null && targetClass != Object.class) {
        for (Field field : targetClass.getDeclaredFields()) {
            if (field.isAnnotationPresent(Autowired.class)) {
                injectField(bean, beanName, field);
            }
        }
        targetClass = targetClass.getSuperclass();
    }
    return bean;
}
```

This mirrors the real Spring's `AutowiredAnnotationBeanPostProcessor.buildAutowiringMetadata()` which calls `ReflectionUtils.doWithLocalFields()` at each level of the hierarchy, prepending parent fields.

</details>

## 2.8 Tests

### Unit Tests — AutowiredBeanPostProcessor

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/annotation/AutowiredBeanPostProcessorTest.java`

```java
@Test
void shouldInjectAutowiredField_WhenFieldAnnotated()          // basic field injection
@Test
void shouldNotInjectField_WhenNotAnnotated()                  // non-@Autowired fields stay null
@Test
void shouldInjectPrivateField_WhenAnnotatedWithAutowired()    // private fields work too
@Test
void shouldInjectInheritedField_WhenSuperclassFieldAnnotated() // hierarchy walking
@Test
void shouldThrowBeanCreationException_WhenDependencyNotFound() // missing dependency
@Test
void shouldReturnSameBean_WhenNoAutowiredFields()             // passthrough for plain beans
```

### Unit Tests — BeanPostProcessor

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/config/BeanPostProcessorTest.java`

```java
@Test
void shouldInvokeBeforeAndAfterInitialization_WhenBeanCreated()   // lifecycle hooks fire
@Test
void shouldApplyPostProcessorsInRegistrationOrder()               // ordering matters
@Test
void shouldAllowReplacingBean_InPostProcessor()                   // proxy pattern
@Test
void shouldNotInvokePostProcessors_WhenNoneRegistered()           // backward compat with ch01
@Test
void shouldInvokePostProcessorsOnlyOnce_WhenSingletonRequestedTwice() // singleton caching
```

### Integration Tests

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/integration/DependencyInjectionIntegrationTest.java`

```java
@Test
void shouldInjectDependencyViaField_WhenAutowiredAnnotated()          // field injection through factory
@Test
void shouldInjectDependencyViaConstructor_WhenSingleConstructorWithParams() // implicit ctor injection
@Test
void shouldInjectDependencyViaConstructor_WhenExplicitlyAnnotated()   // @Autowired constructor
@Test
void shouldInjectBothConstructorAndFieldDependencies()                // combined injection
@Test
void shouldSupportTransitiveDependencies()                            // A → B → C chain
@Test
void shouldShareSingletonInstance_WhenInjectedIntoMultipleBeans()     // singleton identity
@Test
void shouldCreateBeanWithoutDependencies_WhenNoneRequired()           // no-dep beans still work
@Test
void shouldThrowBeanCreationException_WhenFieldDependencyMissing()    // missing field dep
@Test
void shouldThrowBeanCreationException_WhenConstructorDependencyMissing() // missing ctor dep
```

**Run:** `./gradlew :iris-framework:test` — expected: all 41 tests pass (19 from Ch01 + 22 new)

---

## 2.9 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why split constructor injection and field injection into different mechanisms?** Constructor injection happens *during* instantiation (the object doesn't exist yet — you can't post-process it), while field injection happens *after* instantiation (the object exists but its fields are null). This isn't arbitrary — it's a fundamental ordering constraint. The real Spring enforces this same split: `createBeanInstance()` handles constructors, then `populateBean()` handles fields/methods via `InstantiationAwareBeanPostProcessor.postProcessProperties()`.
> - **When constructor injection is better:** Constructor injection makes dependencies explicit and immutable (`final` fields), enables compile-time checking, and prevents partially-initialized objects. The real Spring community strongly recommends constructor injection over field injection for production code.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why does `BeanPostProcessor` return `Object` instead of `void`?** The return value is the bean that downstream processors (and the singleton cache) will see. Returning a *different* object is how Spring creates AOP proxies: `postProcessAfterInitialization()` wraps the real bean in a proxy, and the proxy is what gets cached. This single design decision — returning the bean rather than modifying it in place — enables Spring's entire AOP and transaction management infrastructure without modifying the core container.
> - **Trade-off:** This flexibility means you can't assume `getBean()` returns the original class. It might return a proxy. This is why Spring recommends programming to interfaces.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why does single-constructor autowiring work without `@Autowired`?** Added in Spring 4.3, this convention eliminates boilerplate for the overwhelmingly common case: a class with one constructor. Since there's only one way to construct the object, the container has no ambiguity about what to inject. The real Spring's `AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors()` implements this at lines 349-452 — if no annotated constructors exist and there's exactly one constructor with parameters, it returns that constructor.
> -----------------------------------------------------------

## 2.10 What We Enhanced

| Aspect | Before (ch01) | Current (ch02) | Real Framework |
|--------|---------------|-----------------|----------------|
| **Instantiation** | No-arg constructor or supplier only | Constructor injection with parameter resolution from the container | `ConstructorResolver.autowireConstructor()` with type conversion, qualifier matching, and greedy resolution |
| **Post-processing** | None — `createBean()` returned the raw instance | `BeanPostProcessor` pipeline with before/after init hooks | 7 distinct post-processor invocation points across `createBean()` → `doCreateBean()` → `populateBean()` → `initializeBean()` |
| **Field wiring** | Manual — beans had no way to reference each other | `@Autowired` field injection via `AutowiredBeanPostProcessor` | `AutowiredAnnotationBeanPostProcessor.postProcessProperties()` with `InjectionMetadata` caching, `@Value` support, optional dependencies, and method injection |
| **Constructor accessibility** | `getDeclaredConstructor().newInstance()` — failed on non-public constructors | `setAccessible(true)` on all constructors | `BeanUtils.instantiateClass()` with `ReflectionUtils.makeAccessible()` |

## 2.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@Autowired` (constructor + field) | `@Autowired` | `Autowired.java:1` | Real also targets `METHOD`, `PARAMETER`, `ANNOTATION_TYPE`; has `required` attribute |
| `BeanPostProcessor` (single interface) | `BeanPostProcessor` → `InstantiationAwareBeanPostProcessor` → `SmartInstantiationAwareBeanPostProcessor` | `BeanPostProcessor.java:1` | Real has a deep hierarchy: `InstantiationAwareBPP` adds `postProcessProperties()`, `SmartInstantiationAwareBPP` adds `determineCandidateConstructors()` |
| `AutowiredBeanPostProcessor` (field injection only) | `AutowiredAnnotationBeanPostProcessor` | `AutowiredAnnotationBeanPostProcessor.java:1` | Real handles fields, methods, and constructors; caches `InjectionMetadata`; supports `@Value` and `@Inject` |
| `instantiate()` 3-strategy selection | `AbstractAutowireCapableBeanFactory.createBeanInstance()` + `ConstructorResolver` | `AbstractAutowireCapableBeanFactory.java:1190` | Real delegates constructor selection to post-processors, uses `ConstructorResolver.autowireConstructor()` with type difference weighting |
| `getBean(paramTypes[i])` for constructor args | `ConstructorResolver.resolveAutowiredArgument()` → `beanFactory.resolveDependency()` | `ConstructorResolver.java:900` | Real uses `DependencyDescriptor` with type conversion, qualifiers, `@Primary`, lazy resolution proxies |
| `addBeanPostProcessor()` | `AbstractBeanFactory.addBeanPostProcessor()` | `AbstractBeanFactory.java:968` | Real uses `synchronized` block, sets flag for `InstantiationAware` and `DestructionAware` processors |
| `applyBeanPostProcessorsBeforeInitialization()` | `AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization()` | `AbstractAutowireCapableBeanFactory.java:1809` | Same pattern — iterates processors, chains return values |

## 2.12 Complete Code

### Production Code

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/annotation/Autowired.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a constructor or field for automatic dependency injection.
 *
 * <p><b>Constructor injection:</b> annotate a constructor to tell the container to
 * resolve its parameters from the bean factory. If a class has exactly one constructor
 * (even without this annotation), the container will autowire it automatically.
 *
 * <p><b>Field injection:</b> annotate a field and the {@link AutowiredBeanPostProcessor}
 * will resolve the dependency by type and inject it after construction.
 *
 * <p>In the real Spring Framework, {@code @Autowired} also supports method injection,
 * optional dependencies ({@code required = false}), and collection injection.
 * We simplify to required-only constructor and field injection.
 *
 * @see AutowiredBeanPostProcessor
 * @see org.springframework.beans.factory.annotation.Autowired
 */
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/config/BeanPostProcessor.java` [NEW]

```java
package com.iris.framework.beans.factory.config;

/**
 * Factory hook that allows custom modification of new bean instances.
 *
 * <p>The container calls {@link #postProcessBeforeInitialization} after a bean is
 * instantiated and its dependencies are injected, but before any initialization
 * callbacks ({@code @PostConstruct}, {@code InitializingBean}). It then calls
 * {@link #postProcessAfterInitialization} after initialization callbacks complete.
 *
 * <p>Both methods may return a <b>replacement</b> object (e.g., a proxy), or the
 * original bean unchanged.
 *
 * <p>In the real Spring Framework, {@code BeanPostProcessor} is extended by
 * {@code InstantiationAwareBeanPostProcessor} (field/method injection hooks),
 * {@code SmartInstantiationAwareBeanPostProcessor} (constructor selection),
 * and {@code MergedBeanDefinitionPostProcessor} (metadata caching).
 * We collapse the essential behavior into this single interface.
 *
 * @see com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor
 * @see org.springframework.beans.factory.config.BeanPostProcessor
 */
public interface BeanPostProcessor {

    /**
     * Apply this processor to the given bean instance <i>before</i> any
     * initialization callbacks (e.g., {@code @PostConstruct}).
     *
     * @param bean     the new bean instance
     * @param beanName the name of the bean
     * @return the bean instance to use — either the original or a wrapped one
     */
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    /**
     * Apply this processor to the given bean instance <i>after</i> any
     * initialization callbacks (e.g., {@code @PostConstruct}).
     *
     * @param bean     the bean instance (possibly already wrapped by a prior processor)
     * @param beanName the name of the bean
     * @return the bean instance to use — either the original or a wrapped one
     */
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/annotation/AutowiredBeanPostProcessor.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.reflect.Field;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.config.BeanPostProcessor;

/**
 * {@link BeanPostProcessor} that performs field injection for {@link Autowired @Autowired}
 * annotated fields.
 *
 * <p>After the container instantiates a bean (with constructor injection already
 * resolved), this processor scans the bean's class hierarchy for {@code @Autowired}
 * fields, resolves each dependency by type from the {@link BeanFactory}, and injects
 * it via reflection.
 *
 * <p>In the real Spring Framework, {@code AutowiredAnnotationBeanPostProcessor}
 * handles both field and method injection via {@code postProcessProperties()},
 * and also participates in constructor selection via
 * {@code determineCandidateConstructors()}. We simplify to field-only injection
 * in {@code postProcessBeforeInitialization()}, with constructor injection handled
 * directly by the {@code DefaultBeanFactory}.
 *
 * @see Autowired
 * @see org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor
 */
public class AutowiredBeanPostProcessor implements BeanPostProcessor {

    private final BeanFactory beanFactory;

    public AutowiredBeanPostProcessor(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    /**
     * Scan the bean for {@code @Autowired} fields and inject dependencies.
     * Walks the class hierarchy from the concrete class up to (but not including)
     * {@code Object}, mirroring Spring's {@code buildAutowiringMetadata()}.
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        Class<?> targetClass = bean.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Field field : targetClass.getDeclaredFields()) {
                if (field.isAnnotationPresent(Autowired.class)) {
                    injectField(bean, beanName, field);
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return bean;
    }

    private void injectField(Object bean, String beanName, Field field) {
        try {
            Object dependency = beanFactory.getBean(field.getType());
            field.setAccessible(true);
            field.set(bean, dependency);
        } catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(beanName,
                    "Unsatisfied dependency for field '" + field.getName()
                            + "': No qualifying bean of type '"
                            + field.getType().getName() + "' available",
                    ex);
        } catch (IllegalAccessException ex) {
            throw new BeanCreationException(beanName,
                    "Failed to inject @Autowired field '" + field.getName() + "'",
                    ex);
        }
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
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.annotation.Autowired;
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
            bean = applyBeanPostProcessorsBeforeInitialization(bean, name);

            // (Ch03: @PostConstruct / InitializingBean callbacks will go here)

            // 3. BeanPostProcessors — after initialization
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
}
```

### Test Code

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/annotation/AutowiredBeanPostProcessorTest.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link AutowiredBeanPostProcessor} — tests the post-processor
 * in isolation by calling {@code postProcessBeforeInitialization()} directly.
 */
class AutowiredBeanPostProcessorTest {

    private DefaultBeanFactory factory;
    private AutowiredBeanPostProcessor processor;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        processor = new AutowiredBeanPostProcessor(factory);
    }

    // ── Helper beans ──────────────────────────────────────────────

    static class Repository {
    }

    static class ServiceWithAutowiredField {
        @Autowired
        Repository repository;
    }

    static class ServiceWithoutAutowired {
        Repository repository;
    }

    static class ServiceWithPrivateField {
        @Autowired
        private Repository repository;

        Repository getRepository() {
            return repository;
        }
    }

    static class BaseService {
        @Autowired
        Repository repository;
    }

    static class DerivedService extends BaseService {
    }

    // ── Tests ─────────────────────────────────────────────────────

    @Test
    void shouldInjectAutowiredField_WhenFieldAnnotated() {
        Repository repo = new Repository();
        factory.registerBeanDefinition("repository",
                new BeanDefinition(Repository.class, () -> repo));

        ServiceWithAutowiredField service = new ServiceWithAutowiredField();
        processor.postProcessBeforeInitialization(service, "service");

        assertThat(service.repository).isSameAs(repo);
    }

    @Test
    void shouldNotInjectField_WhenNotAnnotated() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(Repository.class));

        ServiceWithoutAutowired service = new ServiceWithoutAutowired();
        processor.postProcessBeforeInitialization(service, "service");

        assertThat(service.repository).isNull();
    }

    @Test
    void shouldInjectPrivateField_WhenAnnotatedWithAutowired() {
        Repository repo = new Repository();
        factory.registerBeanDefinition("repository",
                new BeanDefinition(Repository.class, () -> repo));

        ServiceWithPrivateField service = new ServiceWithPrivateField();
        processor.postProcessBeforeInitialization(service, "service");

        assertThat(service.getRepository()).isSameAs(repo);
    }

    @Test
    void shouldInjectInheritedField_WhenSuperclassFieldAnnotated() {
        Repository repo = new Repository();
        factory.registerBeanDefinition("repository",
                new BeanDefinition(Repository.class, () -> repo));

        DerivedService service = new DerivedService();
        processor.postProcessBeforeInitialization(service, "derivedService");

        assertThat(service.repository).isSameAs(repo);
    }

    @Test
    void shouldThrowBeanCreationException_WhenDependencyNotFound() {
        ServiceWithAutowiredField service = new ServiceWithAutowiredField();

        assertThatThrownBy(() ->
                processor.postProcessBeforeInitialization(service, "service"))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("Unsatisfied dependency")
                .hasMessageContaining("repository");
    }

    @Test
    void shouldReturnSameBean_WhenNoAutowiredFields() {
        StringBuilder bean = new StringBuilder("original");

        Object result = processor.postProcessBeforeInitialization(bean, "myBean");

        assertThat(result).isSameAs(bean);
    }
}
```

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/config/BeanPostProcessorTest.java` [NEW]

```java
package com.iris.framework.beans.factory.config;

import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for the {@link BeanPostProcessor} contract — verifies that
 * {@link DefaultBeanFactory} invokes before/after initialization hooks
 * at the right times and in the right order.
 */
class BeanPostProcessorTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
    }

    @Test
    void shouldInvokeBeforeAndAfterInitialization_WhenBeanCreated() {
        List<String> log = new ArrayList<>();

        factory.addBeanPostProcessor(new BeanPostProcessor() {
            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName) {
                log.add("before:" + beanName);
                return bean;
            }

            @Override
            public Object postProcessAfterInitialization(Object bean, String beanName) {
                log.add("after:" + beanName);
                return bean;
            }
        });

        factory.registerBeanDefinition("myBean",
                new BeanDefinition(StringBuilder.class));
        factory.getBean("myBean");

        assertThat(log).containsExactly("before:myBean", "after:myBean");
    }

    @Test
    void shouldApplyPostProcessorsInRegistrationOrder() {
        List<String> log = new ArrayList<>();

        factory.addBeanPostProcessor(new BeanPostProcessor() {
            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName) {
                log.add("first");
                return bean;
            }
        });

        factory.addBeanPostProcessor(new BeanPostProcessor() {
            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName) {
                log.add("second");
                return bean;
            }
        });

        factory.registerBeanDefinition("x",
                new BeanDefinition(StringBuilder.class));
        factory.getBean("x");

        assertThat(log).containsExactly("first", "second");
    }

    @Test
    void shouldAllowReplacingBean_InPostProcessor() {
        factory.addBeanPostProcessor(new BeanPostProcessor() {
            @Override
            public Object postProcessAfterInitialization(Object bean, String beanName) {
                if (bean instanceof StringBuilder) {
                    return new StringBuilder("wrapped");
                }
                return bean;
            }
        });

        factory.registerBeanDefinition("myBean",
                new BeanDefinition(StringBuilder.class));
        StringBuilder bean = factory.getBean("myBean", StringBuilder.class);

        assertThat(bean.toString()).isEqualTo("wrapped");
    }

    @Test
    void shouldNotInvokePostProcessors_WhenNoneRegistered() {
        factory.registerBeanDefinition("myBean",
                new BeanDefinition(StringBuilder.class));

        // Just verify it doesn't throw — no post-processors registered
        Object bean = factory.getBean("myBean");
        assertThat(bean).isInstanceOf(StringBuilder.class);
    }

    @Test
    void shouldInvokePostProcessorsOnlyOnce_WhenSingletonRequestedTwice() {
        List<String> log = new ArrayList<>();

        factory.addBeanPostProcessor(new BeanPostProcessor() {
            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName) {
                log.add("processed:" + beanName);
                return bean;
            }
        });

        factory.registerBeanDefinition("myBean",
                new BeanDefinition(StringBuilder.class));
        factory.getBean("myBean");
        factory.getBean("myBean"); // second call — should hit cache

        assertThat(log).containsExactly("processed:myBean");
    }
}
```

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/integration/DependencyInjectionIntegrationTest.java` [NEW]

```java
package com.iris.framework.beans.factory.integration;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Integration tests for Dependency Injection — tests how constructor injection,
 * field injection, and the BeanPostProcessor pipeline work together through
 * the full {@link DefaultBeanFactory} lifecycle.
 */
class DependencyInjectionIntegrationTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        factory.addBeanPostProcessor(new AutowiredBeanPostProcessor(factory));
    }

    // ── Helper beans ──────────────────────────────────────────────

    static class GreetingRepository {
        String getGreeting() {
            return "Hello";
        }
    }

    static class GreetingService {
        @Autowired
        GreetingRepository repository;

        String greet(String name) {
            return repository.getGreeting() + ", " + name + "!";
        }
    }

    /**
     * Single constructor with parameters → implicit autowiring.
     */
    static class NotificationService {
        final GreetingService greetingService;

        NotificationService(GreetingService greetingService) {
            this.greetingService = greetingService;
        }

        String notify(String name) {
            return greetingService.greet(name);
        }
    }

    /**
     * Explicit {@code @Autowired} constructor + {@code @Autowired} field
     * on the same bean — both injection styles combined.
     */
    static class AuditService {
        @Autowired
        GreetingRepository repository;

        final NotificationService notificationService;

        @Autowired
        AuditService(NotificationService notificationService) {
            this.notificationService = notificationService;
        }
    }

    /**
     * A bean with no dependencies — should still work normally.
     */
    static class StandaloneService {
        String name() {
            return "standalone";
        }
    }

    /**
     * Bean that depends on a missing dependency — for error testing.
     */
    static class BrokenService {
        @Autowired
        Runnable missingDependency;
    }

    /**
     * Bean with a constructor dependency that doesn't exist.
     */
    static class BrokenConstructorService {
        BrokenConstructorService(Runnable missing) {
        }
    }

    // ── Field injection ───────────────────────────────────────────

    @Test
    void shouldInjectDependencyViaField_WhenAutowiredAnnotated() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(GreetingRepository.class));
        factory.registerBeanDefinition("greetingService",
                new BeanDefinition(GreetingService.class));

        GreetingService service = factory.getBean("greetingService",
                GreetingService.class);

        assertThat(service.repository).isNotNull();
        assertThat(service.greet("World")).isEqualTo("Hello, World!");
    }

    // ── Constructor injection ─────────────────────────────────────

    @Test
    void shouldInjectDependencyViaConstructor_WhenSingleConstructorWithParams() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(GreetingRepository.class));
        factory.registerBeanDefinition("greetingService",
                new BeanDefinition(GreetingService.class));
        factory.registerBeanDefinition("notificationService",
                new BeanDefinition(NotificationService.class));

        NotificationService service = factory.getBean("notificationService",
                NotificationService.class);

        assertThat(service.greetingService).isNotNull();
        assertThat(service.notify("World")).isEqualTo("Hello, World!");
    }

    @Test
    void shouldInjectDependencyViaConstructor_WhenExplicitlyAnnotated() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(GreetingRepository.class));
        factory.registerBeanDefinition("greetingService",
                new BeanDefinition(GreetingService.class));
        factory.registerBeanDefinition("notificationService",
                new BeanDefinition(NotificationService.class));
        factory.registerBeanDefinition("auditService",
                new BeanDefinition(AuditService.class));

        AuditService service = factory.getBean("auditService",
                AuditService.class);

        assertThat(service.notificationService).isNotNull();
    }

    // ── Combined injection ────────────────────────────────────────

    @Test
    void shouldInjectBothConstructorAndFieldDependencies() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(GreetingRepository.class));
        factory.registerBeanDefinition("greetingService",
                new BeanDefinition(GreetingService.class));
        factory.registerBeanDefinition("notificationService",
                new BeanDefinition(NotificationService.class));
        factory.registerBeanDefinition("auditService",
                new BeanDefinition(AuditService.class));

        AuditService service = factory.getBean("auditService",
                AuditService.class);

        assertThat(service.repository).isNotNull();
        assertThat(service.notificationService).isNotNull();
    }

    // ── Transitive dependencies ───────────────────────────────────

    @Test
    void shouldSupportTransitiveDependencies() {
        // Chain: NotificationService → GreetingService → GreetingRepository
        factory.registerBeanDefinition("repository",
                new BeanDefinition(GreetingRepository.class));
        factory.registerBeanDefinition("greetingService",
                new BeanDefinition(GreetingService.class));
        factory.registerBeanDefinition("notificationService",
                new BeanDefinition(NotificationService.class));

        NotificationService service = factory.getBean("notificationService",
                NotificationService.class);

        assertThat(service.notify("Iris")).isEqualTo("Hello, Iris!");
    }

    // ── Singleton identity ────────────────────────────────────────

    @Test
    void shouldShareSingletonInstance_WhenInjectedIntoMultipleBeans() {
        factory.registerBeanDefinition("repository",
                new BeanDefinition(GreetingRepository.class));
        factory.registerBeanDefinition("greetingService",
                new BeanDefinition(GreetingService.class));

        GreetingService service = factory.getBean("greetingService",
                GreetingService.class);
        GreetingRepository directRepo = factory.getBean("repository",
                GreetingRepository.class);

        // The repository injected into the service is the same singleton
        assertThat(service.repository).isSameAs(directRepo);
    }

    // ── No-dependency beans ───────────────────────────────────────

    @Test
    void shouldCreateBeanWithoutDependencies_WhenNoneRequired() {
        factory.registerBeanDefinition("standalone",
                new BeanDefinition(StandaloneService.class));

        StandaloneService service = factory.getBean("standalone",
                StandaloneService.class);

        assertThat(service.name()).isEqualTo("standalone");
    }

    // ── Error handling ────────────────────────────────────────────

    @Test
    void shouldThrowBeanCreationException_WhenFieldDependencyMissing() {
        factory.registerBeanDefinition("broken",
                new BeanDefinition(BrokenService.class));

        assertThatThrownBy(() -> factory.getBean("broken"))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("Unsatisfied dependency")
                .hasMessageContaining("missingDependency");
    }

    @Test
    void shouldThrowBeanCreationException_WhenConstructorDependencyMissing() {
        factory.registerBeanDefinition("broken",
                new BeanDefinition(BrokenConstructorService.class));

        assertThatThrownBy(() -> factory.getBean("broken"))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("Unsatisfied dependency")
                .hasMessageContaining("constructor parameter");
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@Autowired`** | Marker annotation that tells the container to inject a dependency — on a constructor (resolve parameters) or a field (resolve by type after construction) |
| **`BeanPostProcessor`** | Extensible hook invoked on every bean after instantiation — enables cross-cutting concerns (DI, proxying, validation) without modifying the core container |
| **`AutowiredBeanPostProcessor`** | The concrete `BeanPostProcessor` that scans for `@Autowired` fields and injects them from the container via reflection |
| **Constructor injection** | Dependencies resolved during instantiation — supports `@Autowired` constructors and implicit single-constructor autowiring |
| **Transitive resolution** | `getBean(A)` triggers `getBean(B)` which triggers `getBean(C)` — the container resolves the full dependency graph lazily |

**Next: Chapter 3 — Bean Lifecycle** — Beans can now be wired together, but they have no initialization or cleanup hooks. We'll add `@PostConstruct`, `@PreDestroy`, and the `InitializingBean`/`DisposableBean` interfaces so beans can set up resources after injection and release them on shutdown.
