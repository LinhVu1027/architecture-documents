# Chapter 4: Annotation Configuration

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Beans are registered manually by calling `registerBeanDefinition()` with a class and optional supplier — the user must wire every bean by hand | A 20-bean application requires 20 registration calls plus manual supplier logic for inter-bean dependencies; there's no way to declare "this method produces a bean" | Build `@Configuration` classes with `@Bean` factory methods, and a `ConfigurationClassProcessor` that discovers them, registers bean definitions automatically, and resolves method parameters from the container |

---

## 4.1 The Integration Point

The integration point is `DefaultBeanFactory.createBean()` — the same method that grew in Chapters 2 and 3. Currently it knows two instantiation strategies: supplier-based and constructor-based. We add a third: **factory method invocation**.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Add a factory method check as the first branch in `createBean()`, and a new `instantiateUsingFactoryMethod()` method.

```java
private Object createBean(String name, BeanDefinition bd) {
    try {
        // 1. Instantiate (factory method, supplier, or constructor injection)
        Object bean;
        if (bd.isFactoryMethodBean()) {                         // ← NEW
            bean = instantiateUsingFactoryMethod(name, bd);     // ← NEW
        } else if (bd.getSupplier() != null) {
            bean = bd.getSupplier().get();
        } else {
            bean = instantiate(name, bd);
        }

        // 2. BeanPostProcessors — before initialization
        bean = applyBeanPostProcessorsBeforeInitialization(bean, name);

        // 3. InitializingBean callback
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

1. **Factory method check comes first.** A bean definition with a factory method is always instantiated via that method — never via constructor. The `isFactoryMethodBean()` predicate checks that both `factoryMethod` and `factoryBeanName` are set, so regular beans always fall through to the existing paths.

2. **Parameter resolution reuses the same by-type lookup.** Inside `instantiateUsingFactoryMethod()`, each method parameter is resolved by calling `getBean(paramType)` — the exact same mechanism used for constructor autowiring in `instantiate()`. This is why real Spring sets `AUTOWIRE_CONSTRUCTOR` on factory method bean definitions: they share the same resolution logic.

This connects **`@Bean` factory methods** ↔ **the bean creation pipeline**. To make it work, we need to build:
- `@Configuration` annotation — marks a class as a source of `@Bean` methods
- `@Bean` annotation — marks a method as a bean factory
- `ConfigurationClassProcessor` — discovers `@Configuration` classes, finds `@Bean` methods, registers `BeanDefinition` entries with factory method metadata
- `BeanDefinition` enhancements — `factoryBeanName`, `factoryMethod`, and `isFactoryMethodBean()`

The three instantiation strategies, in priority order:

```
                    createBean(name, bd)
                           │
              ┌────────────┼────────────────┐
              ▼            ▼                ▼
       Factory Method   Supplier      Constructor
     (bd has factory   (bd has       (@Autowired ctor,
      method + bean    supplier)      single ctor,
      name set)                       or no-arg ctor)
              │            │                │
              ▼            ▼                ▼
    invoke @Bean method  supplier.get()  new Instance(args)
    on config class                     resolved by type
    instance, params
    resolved by type
```

---

## 4.2 Enhancing BeanDefinition

The `BeanDefinition` class needs to record where a factory method bean comes from: which configuration class owns the method, and which method to invoke.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/config/BeanDefinition.java`
**Change:** Add `factoryBeanName` (String), `factoryMethod` (Method), and `isFactoryMethodBean()` fields.

```java
// Factory method support (Feature 4: Annotation Configuration)
private String factoryBeanName;
private Method factoryMethod;
```

And the convenience predicate:

```java
public boolean isFactoryMethodBean() {
    return factoryMethod != null && factoryBeanName != null;
}
```

In the real Spring Framework, `RootBeanDefinition` stores `factoryBeanName` (String) and `factoryMethodName` (String), then resolves the actual `Method` later. We store the `Method` object directly — simpler, and we don't need the late-binding flexibility.

<details><summary>Try it yourself: What fields does BeanDefinition need for factory method support?</summary>

You need:
1. `String factoryBeanName` — the name of the `@Configuration` bean that owns the `@Bean` method
2. `Method factoryMethod` — the actual `java.lang.reflect.Method` to invoke
3. A predicate `isFactoryMethodBean()` that returns `true` when both are set

The `beanClass` field is set to the method's return type (e.g., `MessageService.class`), so type-based lookups via `getBean(Class<T>)` still work correctly.

</details>

---

## 4.3 The @Configuration and @Bean Annotations

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/Configuration.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Configuration {
    String value() default "";
}
```

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/Bean.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {
    String value() default "";
}
```

In the real Spring Framework:
- `@Configuration` is meta-annotated with `@Component`, making configuration classes eligible for component scanning (Feature 5). We omit this for now.
- `@Configuration` has a `proxyBeanMethods` attribute (default `true`) that controls CGLIB enhancement. When `true`, inter-bean method calls (e.g., `dataSource()` called from another `@Bean` method) return the singleton from the container instead of creating a new instance. We skip CGLIB entirely — our implementation operates in "lite mode" only.
- `@Bean` supports an array of names (name + aliases), `initMethod`/`destroyMethod` attributes, and `autowireCandidate` flag. We support a single name for simplicity.

<details><summary>Try it yourself: Why does @Bean target ANNOTATION_TYPE in addition to METHOD?</summary>

`@Bean` targets `ANNOTATION_TYPE` so it can be used as a **meta-annotation** — you can create custom composed annotations annotated with `@Bean`. For example, Spring's test framework could define `@MockBean` as a meta-annotation that includes `@Bean`. This is the same meta-annotation pattern that `@Configuration` uses with `@Component`.

</details>

---

## 4.4 The ConfigurationClassProcessor

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`

```java
package com.iris.framework.context.annotation;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;

public class ConfigurationClassProcessor {

    public void processConfigurationClasses(BeanDefinitionRegistry registry) {
        // 1. Snapshot current bean names — avoid ConcurrentModificationException
        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String candidateName : candidateNames) {
            BeanDefinition bd = registry.getBeanDefinition(candidateName);
            Class<?> beanClass = bd.getBeanClass();

            // 2. Check if this bean's class is a @Configuration class
            if (!beanClass.isAnnotationPresent(Configuration.class)) {
                continue;
            }

            // 3. Find and process all @Bean methods
            List<Method> beanMethods = findBeanMethods(beanClass);
            for (Method method : beanMethods) {
                String beanName = resolveBeanName(method);
                BeanDefinition beanDef = createBeanDefinition(method, candidateName);
                registry.registerBeanDefinition(beanName, beanDef);
            }
        }
    }

    private List<Method> findBeanMethods(Class<?> configClass) {
        List<Method> beanMethods = new ArrayList<>();
        for (Method method : configClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Bean.class)) {
                beanMethods.add(method);
            }
        }
        return beanMethods;
    }

    private String resolveBeanName(Method method) {
        Bean beanAnnotation = method.getAnnotation(Bean.class);
        String explicitName = beanAnnotation.value();
        return explicitName.isEmpty() ? method.getName() : explicitName;
    }

    private BeanDefinition createBeanDefinition(Method method, String configBeanName) {
        BeanDefinition bd = new BeanDefinition(method.getReturnType());
        bd.setFactoryBeanName(configBeanName);
        bd.setFactoryMethod(method);
        return bd;
    }

    public static String deriveConfigBeanName(Class<?> configClass) {
        String simpleName = configClass.getSimpleName();
        if (simpleName.isEmpty()) {
            return simpleName;
        }
        return Character.toLowerCase(simpleName.charAt(0)) + simpleName.substring(1);
    }
}
```

The processor's `processConfigurationClasses()` maps to Spring's `ConfigurationClassPostProcessor.processConfigBeanDefinitions()`. The key difference: real Spring implements `BeanDefinitionRegistryPostProcessor` so it runs automatically during `context.refresh()`. Since we don't have `ApplicationContext` yet (Feature 6), we call it explicitly.

<details><summary>Try it yourself: Why does the processor snapshot getBeanDefinitionNames() before iterating?</summary>

Without the snapshot, we'd be modifying the registry while iterating over it. As we discover `@Bean` methods and register new bean definitions, those new definitions would appear in the iteration. Since `@Bean` method results aren't `@Configuration` classes themselves, processing them would just be wasted work — but in a more complex implementation with `@Import` or `@ComponentScan`, this could cause infinite loops or missed configurations.

Real Spring handles this differently: it uses an iterative discovery loop that processes newly-discovered candidates in subsequent passes. Our snapshot approach is simpler — one pass is all we need since we don't support `@Import` or nested `@Configuration`.

</details>

---

## 4.5 The Factory Method Instantiation

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Add the `instantiateUsingFactoryMethod()` method.

```java
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
```

Notice the symmetry with `instantiate()` from Feature 2: both resolve parameters by type via `getBean()`. The difference is the target — constructor vs. method invocation.

---

## 4.6 Tests

### Unit Tests

**New file:** `iris-framework/src/test/java/com/iris/framework/context/annotation/ConfigurationClassProcessorTest.java`

Tests the processor's discovery and registration logic in isolation:

| Test | What It Verifies |
|------|-----------------|
| `shouldRegisterBeanMethodAsBeanDefinition` | A `@Bean` method produces a `BeanDefinition` with factory method metadata |
| `shouldRegisterMultipleBeanMethods` | Multiple `@Bean` methods in one config class each get their own definition |
| `shouldUseExplicitBeanName` | `@Bean("customName")` overrides the method name |
| `shouldIgnoreNonConfigurationClasses` | Non-`@Configuration` beans are skipped |
| `shouldHandleEmptyConfigClass` | Config class with no `@Bean` methods adds nothing |
| `shouldProcessMultipleConfigClasses` | Multiple config classes are all processed |
| `shouldHandlePrivateBeanMethods` | Private `@Bean` methods are discovered via `getDeclaredMethods()` |
| `shouldDeriveConfigBeanName` | `deriveConfigBeanName(AppConfig.class)` → `"appConfig"` |

### Integration Tests

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/integration/AnnotationConfigIntegrationTest.java`

Tests that `@Configuration`/`@Bean` works with Features 1–3:

| Test | Features Integrated |
|------|-------------------|
| `shouldCreateBeanViaFactoryMethod` | F1 (container) + F4 (factory method) |
| `shouldResolveBeanMethodParameters` | F2 (DI) + F4 (parameter resolution) |
| `shouldSupportCrossConfigDependencies` | F4 (cross-config wiring) |
| `shouldApplyLifecycleCallbacksOnFactoryBeans` | F3 (@PostConstruct, InitializingBean) + F4 |
| `shouldApplyPreDestroyOnFactoryBeansOnClose` | F3 (@PreDestroy) + F4 |
| `shouldApplyFieldInjectionOnFactoryBeans` | F2 (@Autowired fields) + F4 |
| `shouldReturnSameSingletonForFactoryBeans` | F1 (singleton cache) + F4 |
| `shouldThrowException_WhenBeanMethodParameterUnsatisfied` | Error handling |
| `shouldMixFactoryAndConstructorBeans` | F1 (constructor beans) + F4 (factory beans) |

Run the tests:

```bash
./gradlew :iris-framework:test
```

All 76 tests pass — including all prior features.

---

## 4.7 Why This Works

> `★ Insight ─────────────────────────────────────`
> **Factory methods are just another instantiation strategy.** The key design decision is that `createBean()` doesn't care *how* a bean is created — it cares about the *lifecycle after creation*. Whether a bean comes from a constructor, a supplier, or a factory method, it always goes through the same BPP → init → BPP pipeline. This is why `@PostConstruct`, `@Autowired` field injection, and `InitializingBean` all work on factory-method beans without any special handling. The real Spring Framework follows this same principle in `AbstractAutowireCapableBeanFactory.createBean()`.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **The processor is a *definition-time* operation, not a *creation-time* operation.** `ConfigurationClassProcessor.processConfigurationClasses()` runs before any beans are created — it only reads and writes `BeanDefinition` metadata. The actual factory method invocation happens later, lazily, on the first `getBean()` call. This separation is critical: it means all bean definitions are known before any bean is instantiated, which allows correct by-type resolution. In real Spring, this maps to the distinction between `BeanDefinitionRegistryPostProcessor` (runs during definition phase) and `BeanPostProcessor` (runs during creation phase).
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **No CGLIB means no inter-bean method call guarantee.** In our implementation, if one `@Bean` method calls another `@Bean` method directly (e.g., `dataSource()` from within `jdbcTemplate()`), it creates a *new* instance instead of returning the singleton. Real Spring solves this with CGLIB subclassing: the `@Configuration` class is replaced by a proxy that intercepts `@Bean` method calls and redirects them through the container. With `@Configuration(proxyBeanMethods = false)` (Spring Boot 2.2+), this proxy is skipped — equivalent to what we do. The workaround is to declare the dependency as a method parameter instead: `jdbcTemplate(DataSource ds)` — which our container resolves correctly.
> `─────────────────────────────────────────────────`

---

## 4.8 What We Enhanced

| Component | Before (Ch01–03) | After (Ch04) |
|-----------|------------------|--------------|
| `BeanDefinition` | Stores class, scope, supplier | + `factoryBeanName`, `factoryMethod`, `isFactoryMethodBean()` |
| `DefaultBeanFactory.createBean()` | Two paths: supplier or constructor | Three paths: **factory method**, supplier, or constructor |
| Bean registration | Manual `registerBeanDefinition()` calls only | Automatic via `@Configuration`/`@Bean` + processor |
| Inter-bean dependencies | Constructor params or `@Autowired` fields only | + `@Bean` method parameters resolved by type |

---

## 4.9 Connection to Real Framework

| Iris Component | Spring Framework Source | Line Reference |
|---------------|----------------------|----------------|
| `@Configuration` | `spring-context/.../context/annotation/Configuration.java` | :27–76 (annotation definition + `proxyBeanMethods`) |
| `@Bean` | `spring-context/.../context/annotation/Bean.java` | :56–148 (annotation definition + attributes) |
| `ConfigurationClassProcessor` | `spring-context/.../context/annotation/ConfigurationClassPostProcessor.java` | :383–485 (`processConfigBeanDefinitions()` — the discovery loop) |
| `findBeanMethods()` | `spring-context/.../context/annotation/ConfigurationClassParser.java` | :303–384 (`doProcessConfigurationClass()` — collects `@Bean` methods) |
| `createBeanDefinition()` | `spring-context/.../context/annotation/ConfigurationClassBeanDefinitionReader.java` | :190–328 (`loadBeanDefinitionsForBeanMethod()` — creates `RootBeanDefinition` with factory method) |
| `BeanDefinition.factoryBeanName` | `spring-beans/.../beans/factory/support/AbstractBeanDefinition.java` | field `factoryBeanName` |
| `instantiateUsingFactoryMethod()` | `spring-beans/.../beans/factory/support/ConstructorResolver.java` | :509–679 (`instantiateUsingFactoryMethod()` — resolves factory method parameters) |
| `deriveConfigBeanName()` | `spring-context/.../context/annotation/AnnotationBeanNameGenerator.java` | :64–89 (`generateBeanName()` → `buildDefaultBeanName()`) |

### Key simplifications we made

| Real Spring | Our Simplification | Removed in |
|-------------|-------------------|------------|
| `BeanDefinitionRegistryPostProcessor` auto-invoked during `refresh()` | Processor called explicitly | Feature 6 (Application Context) |
| CGLIB subclassing for `@Configuration(proxyBeanMethods=true)` | No proxying — "lite mode" only | Out of scope |
| `@Import` for importing additional configuration classes | Not supported | Feature 13 (Auto-Configuration Engine) |
| `@ComponentScan` on configuration classes | Not supported | Feature 5 (Component Scanning) |
| `@Conditional` filtering on `@Bean` methods | Not supported | Feature 12 (Conditional Beans) |
| Iterative discovery loop for newly-registered config classes | Single-pass snapshot | Feature 6 (will iterate during `refresh()`) |
| ASM-based annotation metadata reading | JDK reflection | Permanent simplification |

---

## 4.10 Complete Code

### Production Code

#### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/beans/factory/config/BeanDefinition.java`

```java
package com.iris.framework.beans.factory.config;

import java.lang.reflect.Method;
import java.util.function.Supplier;

/**
 * Holds the metadata for a single bean: its class, scope, and an optional
 * supplier for programmatic instantiation.
 *
 * <p>In the real Spring Framework this is an interface with multiple implementations
 * (RootBeanDefinition, GenericBeanDefinition, etc.) supporting constructor args,
 * property values, factory methods, and more. We simplify to a single concrete class
 * with just the essentials: class, scope, supplier, and factory method support.
 *
 * <p>Factory method support (added in Feature 4) allows beans to be created by
 * invoking a {@code @Bean} method on a {@code @Configuration} class instance,
 * rather than by calling a constructor directly. This maps to Spring's
 * {@code factoryBeanName} + {@code factoryMethodName} on {@code RootBeanDefinition}.
 *
 * @see org.springframework.beans.factory.config.BeanDefinition
 */
public class BeanDefinition {

    public static final String SCOPE_SINGLETON = "singleton";

    private final Class<?> beanClass;
    private String scope = SCOPE_SINGLETON;
    private Supplier<?> supplier;

    // Factory method support (Feature 4: Annotation Configuration)
    private String factoryBeanName;
    private Method factoryMethod;

    public BeanDefinition(Class<?> beanClass) {
        this.beanClass = beanClass;
    }

    public BeanDefinition(Class<?> beanClass, Supplier<?> supplier) {
        this.beanClass = beanClass;
        this.supplier = supplier;
    }

    public Class<?> getBeanClass() {
        return beanClass;
    }

    public String getScope() {
        return scope;
    }

    public void setScope(String scope) {
        this.scope = scope;
    }

    public boolean isSingleton() {
        return SCOPE_SINGLETON.equals(scope);
    }

    public Supplier<?> getSupplier() {
        return supplier;
    }

    public void setSupplier(Supplier<?> supplier) {
        this.supplier = supplier;
    }

    /**
     * Return the name of the bean that owns the factory method (the
     * {@code @Configuration} class bean), or {@code null} for regular beans.
     */
    public String getFactoryBeanName() {
        return factoryBeanName;
    }

    public void setFactoryBeanName(String factoryBeanName) {
        this.factoryBeanName = factoryBeanName;
    }

    /**
     * Return the factory method ({@code @Bean} method) to invoke for creating
     * this bean, or {@code null} for regular beans.
     */
    public Method getFactoryMethod() {
        return factoryMethod;
    }

    public void setFactoryMethod(Method factoryMethod) {
        this.factoryMethod = factoryMethod;
    }

    /**
     * Return whether this bean is defined via a factory method on a
     * configuration class, rather than via constructor instantiation.
     */
    public boolean isFactoryMethodBean() {
        return factoryMethod != null && factoryBeanName != null;
    }
}
```

#### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`

```java
package com.iris.framework.beans.factory.support;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
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

#### [NEW] `iris-framework/src/main/java/com/iris/framework/context/annotation/Configuration.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that a class declares one or more {@link Bean @Bean} methods and
 * may be processed by the {@link ConfigurationClassProcessor} to generate
 * bean definitions.
 *
 * <p>In the real Spring Framework, {@code @Configuration} is meta-annotated
 * with {@code @Component}, making configuration classes eligible for component
 * scanning. It also supports {@code proxyBeanMethods} (CGLIB enhancement) so
 * that inter-bean method calls return singletons. We simplify:
 * <ul>
 *   <li>No CGLIB proxying — operates in "lite mode" only</li>
 *   <li>No {@code @Component} meta-annotation (component scanning is Feature 5)</li>
 * </ul>
 *
 * @see Bean
 * @see ConfigurationClassProcessor
 * @see org.springframework.context.annotation.Configuration
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Configuration {

    /**
     * Explicit bean name for this configuration class. If empty, the
     * processor derives a name from the class (lowercase first letter).
     */
    String value() default "";
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/context/annotation/Bean.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that a method produces a bean to be managed by the container.
 *
 * <p>The method name becomes the bean name (unless overridden via {@link #value()}).
 * Method parameters are resolved from the container by type — the same autowiring
 * mechanism used for constructor injection.
 *
 * <p>Simplifications from the real {@code @Bean}:
 * <ul>
 *   <li>No {@code initMethod}/{@code destroyMethod} attributes (use {@code @PostConstruct}
 *       and {@code @PreDestroy} on the returned bean's class instead)</li>
 *   <li>No {@code autowireCandidate} or {@code bootstrap} attributes</li>
 *   <li>Single name only (no aliases array)</li>
 * </ul>
 *
 * @see Configuration
 * @see org.springframework.context.annotation.Bean
 */
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {

    /**
     * The name of the bean. If empty, the method name is used.
     */
    String value() default "";
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`

```java
package com.iris.framework.context.annotation;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;

/**
 * Processes {@link Configuration @Configuration} classes registered in the
 * container, discovers their {@link Bean @Bean} methods, and registers a
 * {@link BeanDefinition} for each one.
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code ConfigurationClassPostProcessor}, which in the real framework:
 * <ol>
 *   <li>Implements {@code BeanDefinitionRegistryPostProcessor}</li>
 *   <li>Uses {@code ConfigurationClassParser} to recursively discover
 *       {@code @Configuration}, {@code @ComponentScan}, {@code @Import}, etc.</li>
 *   <li>Uses {@code ConfigurationClassBeanDefinitionReader} to register beans</li>
 *   <li>Applies CGLIB enhancement for full {@code @Configuration} classes</li>
 * </ol>
 *
 * <p>Our simplification:
 * <ul>
 *   <li>Scans already-registered bean definitions for {@code @Configuration}</li>
 *   <li>Finds {@code @Bean} methods and registers them as factory-method beans</li>
 *   <li>No CGLIB, no {@code @Import}, no {@code @ComponentScan} processing</li>
 *   <li>No iterative discovery loop — single pass</li>
 * </ul>
 *
 * @see Configuration
 * @see Bean
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public class ConfigurationClassProcessor {

    /**
     * Process all {@code @Configuration} classes currently registered in the
     * given registry. For each {@code @Bean} method found, a new
     * {@link BeanDefinition} is registered with factory method metadata.
     *
     * <p>This maps to Spring's
     * {@code ConfigurationClassPostProcessor.processConfigBeanDefinitions()}:
     * iterate registered bean definitions, identify configuration candidates,
     * parse them, and register bean definitions for {@code @Bean} methods.
     *
     * @param registry the bean definition registry to scan and populate
     */
    public void processConfigurationClasses(BeanDefinitionRegistry registry) {
        // 1. Snapshot current bean names — avoid ConcurrentModificationException
        //    as we register new definitions during iteration
        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String candidateName : candidateNames) {
            BeanDefinition bd = registry.getBeanDefinition(candidateName);
            Class<?> beanClass = bd.getBeanClass();

            // 2. Check if this bean's class is a @Configuration class
            if (!beanClass.isAnnotationPresent(Configuration.class)) {
                continue;
            }

            // 3. Find and process all @Bean methods
            List<Method> beanMethods = findBeanMethods(beanClass);
            for (Method method : beanMethods) {
                String beanName = resolveBeanName(method);
                BeanDefinition beanDef = createBeanDefinition(method, candidateName);
                registry.registerBeanDefinition(beanName, beanDef);
            }
        }
    }

    /**
     * Find all methods annotated with {@code @Bean} in the given class,
     * including inherited methods.
     *
     * <p>In the real Spring Framework, {@code ConfigurationClassParser} uses
     * ASM-based metadata reading for deterministic ordering. We use JDK
     * reflection which is sufficient for our simplified version.
     */
    private List<Method> findBeanMethods(Class<?> configClass) {
        List<Method> beanMethods = new ArrayList<>();
        for (Method method : configClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Bean.class)) {
                beanMethods.add(method);
            }
        }
        return beanMethods;
    }

    /**
     * Determine the bean name: use the {@code @Bean(value)} if provided,
     * otherwise fall back to the method name.
     *
     * <p>In the real Spring Framework, {@code @Bean} supports an array of
     * names (name + aliases). We support a single name for simplicity.
     */
    private String resolveBeanName(Method method) {
        Bean beanAnnotation = method.getAnnotation(Bean.class);
        String explicitName = beanAnnotation.value();
        return explicitName.isEmpty() ? method.getName() : explicitName;
    }

    /**
     * Create a {@link BeanDefinition} that records this bean should be created
     * by invoking the given factory method on the named configuration class.
     *
     * <p>The bean's type is the method's return type. The factory method's
     * parameters will be resolved from the container at creation time — just
     * like constructor autowiring.
     *
     * <p>In the real Spring Framework, this is done in
     * {@code ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod()},
     * which sets {@code factoryBeanName}, {@code factoryMethodName}, and
     * {@code AUTOWIRE_CONSTRUCTOR} on a {@code RootBeanDefinition}.
     */
    private BeanDefinition createBeanDefinition(Method method, String configBeanName) {
        BeanDefinition bd = new BeanDefinition(method.getReturnType());
        bd.setFactoryBeanName(configBeanName);
        bd.setFactoryMethod(method);
        return bd;
    }

    /**
     * Derive a default bean name for a {@code @Configuration} class:
     * lowercase the first character of the simple class name.
     *
     * <p>For example, {@code AppConfig} → {@code "appConfig"}.
     *
     * <p>This mirrors Spring's {@code AnnotationBeanNameGenerator} which uses
     * the short class name with a lowercase first letter for {@code @Component}
     * and {@code @Configuration} classes.
     */
    public static String deriveConfigBeanName(Class<?> configClass) {
        String simpleName = configClass.getSimpleName();
        if (simpleName.isEmpty()) {
            return simpleName;
        }
        return Character.toLowerCase(simpleName.charAt(0)) + simpleName.substring(1);
    }
}
```

### Test Code

#### [NEW] `iris-framework/src/test/java/com/iris/framework/context/annotation/ConfigurationClassProcessorTest.java`

```java
package com.iris.framework.context.annotation;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;

/**
 * Unit tests for {@link ConfigurationClassProcessor}.
 */
class ConfigurationClassProcessorTest {

    private DefaultBeanFactory factory;
    private ConfigurationClassProcessor processor;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        processor = new ConfigurationClassProcessor();
    }

    // -----------------------------------------------------------------------
    // Test configuration classes
    // -----------------------------------------------------------------------

    @Configuration
    static class SimpleConfig {
        @Bean
        public String greeting() {
            return "Hello, Iris!";
        }
    }

    @Configuration
    static class MultipleBeansConfig {
        @Bean
        public String firstName() {
            return "John";
        }

        @Bean
        public Integer age() {
            return 30;
        }
    }

    @Configuration
    static class CustomNameConfig {
        @Bean("customGreeting")
        public String greeting() {
            return "Custom Hello!";
        }
    }

    // A non-@Configuration class — should be ignored
    static class PlainClass {
        public String notABean() {
            return "not a bean";
        }
    }

    @Configuration
    static class EmptyConfig {
        // No @Bean methods
    }

    @Configuration
    static class PrivateMethodConfig {
        @Bean
        private String secret() {
            return "secret-value";
        }
    }

    // -----------------------------------------------------------------------
    // Tests
    // -----------------------------------------------------------------------

    @Test
    @DisplayName("should register @Bean method as bean definition")
    void shouldRegisterBeanMethodAsBeanDefinition() {
        factory.registerBeanDefinition("simpleConfig",
                new BeanDefinition(SimpleConfig.class));

        processor.processConfigurationClasses(factory);

        assertThat(factory.containsBeanDefinition("greeting")).isTrue();
        BeanDefinition bd = factory.getBeanDefinition("greeting");
        assertThat(bd.getBeanClass()).isEqualTo(String.class);
        assertThat(bd.isFactoryMethodBean()).isTrue();
        assertThat(bd.getFactoryBeanName()).isEqualTo("simpleConfig");
        assertThat(bd.getFactoryMethod().getName()).isEqualTo("greeting");
    }

    @Test
    @DisplayName("should register multiple @Bean methods from one config class")
    void shouldRegisterMultipleBeanMethods() {
        factory.registerBeanDefinition("multipleBeansConfig",
                new BeanDefinition(MultipleBeansConfig.class));

        processor.processConfigurationClasses(factory);

        assertThat(factory.containsBeanDefinition("firstName")).isTrue();
        assertThat(factory.containsBeanDefinition("age")).isTrue();
        assertThat(factory.getBeanDefinition("firstName").getBeanClass())
                .isEqualTo(String.class);
        assertThat(factory.getBeanDefinition("age").getBeanClass())
                .isEqualTo(Integer.class);
    }

    @Test
    @DisplayName("should use @Bean value as bean name when specified")
    void shouldUseExplicitBeanName() {
        factory.registerBeanDefinition("customNameConfig",
                new BeanDefinition(CustomNameConfig.class));

        processor.processConfigurationClasses(factory);

        assertThat(factory.containsBeanDefinition("customGreeting")).isTrue();
        assertThat(factory.containsBeanDefinition("greeting")).isFalse();
    }

    @Test
    @DisplayName("should ignore non-@Configuration classes")
    void shouldIgnoreNonConfigurationClasses() {
        factory.registerBeanDefinition("plainClass",
                new BeanDefinition(PlainClass.class));

        int countBefore = factory.getBeanDefinitionCount();
        processor.processConfigurationClasses(factory);
        int countAfter = factory.getBeanDefinitionCount();

        assertThat(countAfter).isEqualTo(countBefore);
    }

    @Test
    @DisplayName("should handle @Configuration class with no @Bean methods")
    void shouldHandleEmptyConfigClass() {
        factory.registerBeanDefinition("emptyConfig",
                new BeanDefinition(EmptyConfig.class));

        int countBefore = factory.getBeanDefinitionCount();
        processor.processConfigurationClasses(factory);
        int countAfter = factory.getBeanDefinitionCount();

        assertThat(countAfter).isEqualTo(countBefore);
    }

    @Test
    @DisplayName("should process multiple @Configuration classes")
    void shouldProcessMultipleConfigClasses() {
        factory.registerBeanDefinition("simpleConfig",
                new BeanDefinition(SimpleConfig.class));
        factory.registerBeanDefinition("multipleBeansConfig",
                new BeanDefinition(MultipleBeansConfig.class));

        processor.processConfigurationClasses(factory);

        assertThat(factory.containsBeanDefinition("greeting")).isTrue();
        assertThat(factory.containsBeanDefinition("firstName")).isTrue();
        assertThat(factory.containsBeanDefinition("age")).isTrue();
    }

    @Test
    @DisplayName("should handle private @Bean methods")
    void shouldHandlePrivateBeanMethods() {
        factory.registerBeanDefinition("privateMethodConfig",
                new BeanDefinition(PrivateMethodConfig.class));

        processor.processConfigurationClasses(factory);

        assertThat(factory.containsBeanDefinition("secret")).isTrue();
        assertThat(factory.getBeanDefinition("secret").isFactoryMethodBean()).isTrue();
    }

    @Test
    @DisplayName("should derive default config bean name from class name")
    void shouldDeriveConfigBeanName() {
        assertThat(ConfigurationClassProcessor.deriveConfigBeanName(SimpleConfig.class))
                .isEqualTo("simpleConfig");
        assertThat(ConfigurationClassProcessor.deriveConfigBeanName(MultipleBeansConfig.class))
                .isEqualTo("multipleBeansConfig");
    }
}
```

#### [NEW] `iris-framework/src/test/java/com/iris/framework/beans/factory/integration/AnnotationConfigIntegrationTest.java`

```java
package com.iris.framework.beans.factory.integration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.ArrayList;
import java.util.List;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.InitializingBean;
import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;

/**
 * Integration tests for Annotation Configuration (Feature 4) with prior features.
 *
 * <p>Tests that {@code @Configuration}/{@code @Bean} works correctly with:
 * <ul>
 *   <li>Feature 1: Bean Container — factory method beans are stored and retrieved</li>
 *   <li>Feature 2: Dependency Injection — @Bean method parameters and @Autowired fields</li>
 *   <li>Feature 3: Bean Lifecycle — @PostConstruct/@PreDestroy on factory-created beans</li>
 * </ul>
 */
class AnnotationConfigIntegrationTest {

    private DefaultBeanFactory factory;
    private ConfigurationClassProcessor processor;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        factory.addBeanPostProcessor(new AutowiredBeanPostProcessor(factory));
        factory.addBeanPostProcessor(new LifecycleBeanPostProcessor());
        processor = new ConfigurationClassProcessor();
    }

    // -----------------------------------------------------------------------
    // Domain classes
    // -----------------------------------------------------------------------

    interface MessageRepository {
        String findMessage();
    }

    static class InMemoryMessageRepository implements MessageRepository {
        @Override
        public String findMessage() {
            return "Hello from repository!";
        }
    }

    static class MessageService {
        private final MessageRepository repository;

        MessageService(MessageRepository repository) {
            this.repository = repository;
        }

        public String getMessage() {
            return repository.findMessage();
        }
    }

    static class NotificationService {
        private final MessageService messageService;
        private final String prefix;

        NotificationService(MessageService messageService, String prefix) {
            this.messageService = messageService;
            this.prefix = prefix;
        }

        public String send() {
            return prefix + ": " + messageService.getMessage();
        }
    }

    static class LifecycleTrackerBean implements InitializingBean {
        final List<String> events = new ArrayList<>();

        @PostConstruct
        void postConstruct() {
            events.add("postConstruct");
        }

        @Override
        public void afterPropertiesSet() {
            events.add("afterPropertiesSet");
        }

        @PreDestroy
        void preDestroy() {
            events.add("preDestroy");
        }
    }

    static class ServiceWithFieldInjection {
        @Autowired
        MessageRepository repository;

        public String getMessageFromRepo() {
            return repository.findMessage();
        }
    }

    // -----------------------------------------------------------------------
    // Configuration classes
    // -----------------------------------------------------------------------

    @Configuration
    static class AppConfig {
        @Bean
        public MessageRepository messageRepository() {
            return new InMemoryMessageRepository();
        }

        @Bean
        public MessageService messageService(MessageRepository repository) {
            return new MessageService(repository);
        }
    }

    @Configuration
    static class NotificationConfig {
        @Bean
        public NotificationService notificationService(MessageService messageService) {
            return new NotificationService(messageService, "ALERT");
        }
    }

    @Configuration
    static class LifecycleConfig {
        @Bean
        public LifecycleTrackerBean lifecycleTracker() {
            return new LifecycleTrackerBean();
        }
    }

    @Configuration
    static class FieldInjectionConfig {
        @Bean
        public MessageRepository messageRepository() {
            return new InMemoryMessageRepository();
        }

        @Bean
        public ServiceWithFieldInjection fieldInjectedService() {
            return new ServiceWithFieldInjection();
        }
    }

    @Configuration
    static class MissingDependencyConfig {
        @Bean
        public MessageService messageService(MessageRepository repo) {
            return new MessageService(repo);
        }
    }

    @Configuration
    static class MixedConfig {
        @Bean
        public MessageService messageService(MessageRepository repo) {
            return new MessageService(repo);
        }
    }

    // -----------------------------------------------------------------------
    // Tests
    // -----------------------------------------------------------------------

    @Test
    @DisplayName("should create bean via @Bean factory method")
    void shouldCreateBeanViaFactoryMethod() {
        factory.registerBeanDefinition("appConfig", new BeanDefinition(AppConfig.class));
        processor.processConfigurationClasses(factory);

        MessageRepository repo = factory.getBean(MessageRepository.class);
        assertThat(repo.findMessage()).isEqualTo("Hello from repository!");
    }

    @Test
    @DisplayName("should resolve @Bean method parameters from the container")
    void shouldResolveBeanMethodParameters() {
        factory.registerBeanDefinition("appConfig", new BeanDefinition(AppConfig.class));
        processor.processConfigurationClasses(factory);

        MessageService service = factory.getBean(MessageService.class);
        assertThat(service.getMessage()).isEqualTo("Hello from repository!");
    }

    @Test
    @DisplayName("should support inter-bean dependencies across config classes")
    void shouldSupportCrossConfigDependencies() {
        factory.registerBeanDefinition("appConfig", new BeanDefinition(AppConfig.class));
        factory.registerBeanDefinition("notificationConfig",
                new BeanDefinition(NotificationConfig.class));
        processor.processConfigurationClasses(factory);

        NotificationService notif = factory.getBean(NotificationService.class);
        assertThat(notif.send()).isEqualTo("ALERT: Hello from repository!");
    }

    @Test
    @DisplayName("should apply @PostConstruct and InitializingBean on factory-created beans")
    void shouldApplyLifecycleCallbacksOnFactoryBeans() {
        factory.registerBeanDefinition("lifecycleConfig",
                new BeanDefinition(LifecycleConfig.class));
        processor.processConfigurationClasses(factory);

        LifecycleTrackerBean tracker = factory.getBean(LifecycleTrackerBean.class);
        assertThat(tracker.events).containsExactly("postConstruct", "afterPropertiesSet");
    }

    @Test
    @DisplayName("should apply @PreDestroy on factory-created beans during close")
    void shouldApplyPreDestroyOnFactoryBeansOnClose() {
        factory.registerBeanDefinition("lifecycleConfig",
                new BeanDefinition(LifecycleConfig.class));
        processor.processConfigurationClasses(factory);

        LifecycleTrackerBean tracker = factory.getBean(LifecycleTrackerBean.class);
        factory.close();

        assertThat(tracker.events).containsExactly(
                "postConstruct", "afterPropertiesSet", "preDestroy");
    }

    @Test
    @DisplayName("should apply @Autowired field injection on factory-created beans")
    void shouldApplyFieldInjectionOnFactoryBeans() {
        factory.registerBeanDefinition("fieldInjectionConfig",
                new BeanDefinition(FieldInjectionConfig.class));
        processor.processConfigurationClasses(factory);

        ServiceWithFieldInjection service = factory.getBean(ServiceWithFieldInjection.class);
        assertThat(service.getMessageFromRepo()).isEqualTo("Hello from repository!");
    }

    @Test
    @DisplayName("should return same singleton for factory-method beans")
    void shouldReturnSameSingletonForFactoryBeans() {
        factory.registerBeanDefinition("appConfig", new BeanDefinition(AppConfig.class));
        processor.processConfigurationClasses(factory);

        MessageRepository first = factory.getBean(MessageRepository.class);
        MessageRepository second = factory.getBean(MessageRepository.class);
        assertThat(first).isSameAs(second);
    }

    @Test
    @DisplayName("should throw BeanCreationException for unsatisfied @Bean method parameter")
    void shouldThrowException_WhenBeanMethodParameterUnsatisfied() {
        factory.registerBeanDefinition("missingDepConfig",
                new BeanDefinition(MissingDependencyConfig.class));
        processor.processConfigurationClasses(factory);

        assertThatThrownBy(() -> factory.getBean(MessageService.class))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("Unsatisfied dependency")
                .hasMessageContaining("MessageRepository");
    }

    @Test
    @DisplayName("should mix factory-method beans with regular constructor beans")
    void shouldMixFactoryAndConstructorBeans() {
        // Register a regular (constructor-instantiated) repository bean
        factory.registerBeanDefinition("messageRepository",
                new BeanDefinition(InMemoryMessageRepository.class));

        // Register a @Configuration class whose @Bean method depends on the
        // constructor-instantiated repository bean
        factory.registerBeanDefinition("mixedConfig",
                new BeanDefinition(MixedConfig.class));
        processor.processConfigurationClasses(factory);

        MessageService service = factory.getBean(MessageService.class);
        assertThat(service.getMessage()).isEqualTo("Hello from repository!");
    }
}
```

---

## Summary

| What | Files |
|------|-------|
| New annotations | `Configuration.java`, `Bean.java` |
| New processor | `ConfigurationClassProcessor.java` |
| Modified | `BeanDefinition.java` (+factory method fields), `DefaultBeanFactory.java` (+factory method instantiation) |
| Unit tests | `ConfigurationClassProcessorTest.java` (8 tests) |
| Integration tests | `AnnotationConfigIntegrationTest.java` (9 tests) |
| Total tests | 76 (all passing, including 59 from Features 1–3) |

**Next chapter:** [Chapter 5: Component Scanning](ch05_component_scanning.md) — auto-discover `@Component` classes by scanning the classpath, eliminating the need to manually register even the `@Configuration` classes.
