# Chapter 1: Bean Container

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| No simplified framework exists yet | No way to register, store, or retrieve Java objects by name or type — every class must manually instantiate its own dependencies | Build a bean container that registers bean definitions and creates/retrieves singleton instances by name and by type |

---

## 1.1 The Core Data Structure

This is a foundation chapter — there is no prior system to plug into. The core data structure is `DefaultBeanFactory` itself: two maps that decouple "what to create" from "what's already created."

```
DefaultBeanFactory
├── beanDefinitionMap:  Map<String, BeanDefinition>   ← "what to create"
├── beanDefinitionNames: List<String>                 ← registration order
└── singletonObjects:   Map<String, Object>           ← "already created" cache
```

Every future feature plugs into this container:
- **Dependency Injection** (ch02) will read from the container to resolve `@Autowired` fields
- **Bean Lifecycle** (ch03) will hook into the creation pipeline for `@PostConstruct`/`@PreDestroy`
- **Annotation Configuration** (ch04) will register `BeanDefinition`s from `@Bean` methods
- **Application Context** (ch06) will orchestrate bulk creation of all singletons

To build this, we need:
- `BeanDefinition` — metadata describing a bean (class, scope, supplier)
- `BeanFactory` — the read API (`getBean()`)
- `BeanDefinitionRegistry` — the write API (`registerBeanDefinition()`)
- `DefaultBeanFactory` — the implementation combining both
- Three exception types for error handling

## 1.2 BeanDefinition — The Metadata

A `BeanDefinition` answers "what bean to create and how." In the real Spring Framework, this is an interface (`BeanDefinition`) with multiple implementations (`RootBeanDefinition`, `GenericBeanDefinition`, `ChildBeanDefinition`) supporting constructor arguments, property values, factory methods, parent definitions, and more. We simplify to a single concrete class with three fields: bean class, scope, and supplier.

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/config/BeanDefinition.java`

```java
package com.iris.framework.beans.factory.config;

import java.util.function.Supplier;

public class BeanDefinition {

    public static final String SCOPE_SINGLETON = "singleton";

    private final Class<?> beanClass;
    private String scope = SCOPE_SINGLETON;
    private Supplier<?> supplier;

    public BeanDefinition(Class<?> beanClass) {
        this.beanClass = beanClass;
    }

    public BeanDefinition(Class<?> beanClass, Supplier<?> supplier) {
        this.beanClass = beanClass;
        this.supplier = supplier;
    }

    public Class<?> getBeanClass() { return beanClass; }
    public String getScope() { return scope; }
    public void setScope(String scope) { this.scope = scope; }
    public boolean isSingleton() { return SCOPE_SINGLETON.equals(scope); }
    public Supplier<?> getSupplier() { return supplier; }
    public void setSupplier(Supplier<?> supplier) { this.supplier = supplier; }
}
```

The `Supplier<?>` field is a simplification of the real framework's factory methods and constructor resolution. When present, the container calls `supplier.get()` instead of using reflection — this is how `@Bean` methods will work in Chapter 4.

## 1.3 BeanFactory & BeanDefinitionRegistry — The Two Contracts

Spring separates "reading beans" from "registering beans" into two interfaces. This separation lets consumers of beans depend only on the read API, never on registration.

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/BeanFactory.java`

```java
package com.iris.framework.beans.factory;

public interface BeanFactory {
    Object getBean(String name);
    <T> T getBean(String name, Class<T> requiredType);
    <T> T getBean(Class<T> requiredType);
    boolean containsBean(String name);
    boolean isSingleton(String name);
}
```

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/BeanDefinitionRegistry.java`

```java
package com.iris.framework.beans.factory.support;

import com.iris.framework.beans.factory.config.BeanDefinition;

public interface BeanDefinitionRegistry {
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition);
    BeanDefinition getBeanDefinition(String beanName);
    boolean containsBeanDefinition(String beanName);
    String[] getBeanDefinitionNames();
    int getBeanDefinitionCount();
}
```

In the real framework, `BeanFactory` has a deep hierarchy: `ListableBeanFactory` (iterate all beans by type), `HierarchicalBeanFactory` (parent-child chains), `ConfigurableBeanFactory` (mutable settings). We collapse everything into one interface.

## 1.4 Exception Types

Three exceptions cover the error cases:

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/NoSuchBeanDefinitionException.java`

```java
package com.iris.framework.beans.factory;

public class NoSuchBeanDefinitionException extends RuntimeException {
    private final String beanName;

    public NoSuchBeanDefinitionException(String beanName) {
        super("No bean named '" + beanName + "' available");
        this.beanName = beanName;
    }

    public NoSuchBeanDefinitionException(Class<?> type) {
        super("No qualifying bean of type '" + type.getName() + "' available");
        this.beanName = null;
    }

    public String getBeanName() { return beanName; }
}
```

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/NoUniqueBeanDefinitionException.java`

```java
package com.iris.framework.beans.factory;

import java.util.Collection;

public class NoUniqueBeanDefinitionException extends NoSuchBeanDefinitionException {
    private final int numberOfBeansFound;
    private final Collection<String> beanNamesFound;

    public NoUniqueBeanDefinitionException(Class<?> type, Collection<String> beanNamesFound) {
        super("No qualifying bean of type '" + type.getName() +
                "' available: expected single matching bean but found " +
                beanNamesFound.size() + ": " + beanNamesFound);
        this.numberOfBeansFound = beanNamesFound.size();
        this.beanNamesFound = beanNamesFound;
    }

    public int getNumberOfBeansFound() { return numberOfBeansFound; }
    public Collection<String> getBeanNamesFound() { return beanNamesFound; }
}
```

Notice `NoUniqueBeanDefinitionException extends NoSuchBeanDefinitionException` — exactly as in Spring. This means any code catching "bean not found" will also catch "too many beans found," which makes sense: both mean a single-bean lookup failed.

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/BeanCreationException.java`

```java
package com.iris.framework.beans.factory;

public class BeanCreationException extends RuntimeException {
    private final String beanName;

    public BeanCreationException(String beanName, String message) {
        super("Error creating bean with name '" + beanName + "': " + message);
        this.beanName = beanName;
    }

    public BeanCreationException(String beanName, String message, Throwable cause) {
        super("Error creating bean with name '" + beanName + "': " + message, cause);
        this.beanName = beanName;
    }

    public String getBeanName() { return beanName; }
}
```

## 1.5 DefaultBeanFactory — The Implementation

**New file:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`

```java
package com.iris.framework.beans.factory.support;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.config.BeanDefinition;

public class DefaultBeanFactory implements BeanFactory, BeanDefinitionRegistry {

    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
    private final List<String> beanDefinitionNames = new ArrayList<>(256);
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // ── BeanDefinitionRegistry ────────────────────────────────────

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

    // ── BeanFactory ───────────────────────────────────────────────

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
        return getBeanDefinition(name).isSingleton();
    }

    // ── Internal ──────────────────────────────────────────────────

    private Object doGetBean(String name) {
        Object singleton = this.singletonObjects.get(name);
        if (singleton != null) {
            return singleton;
        }
        BeanDefinition bd = getBeanDefinition(name);
        Object bean = createBean(name, bd);
        if (bd.isSingleton()) {
            this.singletonObjects.put(name, bean);
        }
        return bean;
    }

    private Object createBean(String name, BeanDefinition bd) {
        try {
            if (bd.getSupplier() != null) {
                return bd.getSupplier().get();
            }
            return bd.getBeanClass().getDeclaredConstructor().newInstance();
        } catch (Exception ex) {
            throw new BeanCreationException(name, "Instantiation of bean failed", ex);
        }
    }
}
```

The `doGetBean()` flow:

```
getBean("userService")
    │
    ├──> Check singletonObjects cache
    │    └── Found? → return immediately
    │
    ├──> Get BeanDefinition from beanDefinitionMap
    │    └── Not found? → throw NoSuchBeanDefinitionException
    │
    ├──> createBean()
    │    ├── Supplier present? → supplier.get()
    │    └── No supplier? → Class.getDeclaredConstructor().newInstance()
    │
    └──> Cache in singletonObjects → return
```

## 1.6 Try It Yourself

<details>
<summary>Challenge: Implement type-based bean lookup that matches against superclasses and interfaces</summary>

Given a registered `StringBuilder` bean, calling `getBean(CharSequence.class)` should find it because `StringBuilder implements CharSequence`. Think about which method to use: `Class.isInstance()` checks an object, but we need to check at the class level.

```java
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
```

The key is `requiredType.isAssignableFrom(bd.getBeanClass())` — this returns `true` if `requiredType` is a superclass or interface of the bean's class.

</details>

<details>
<summary>Challenge: Make getBean() return the same instance for singletons</summary>

The singleton pattern here means: create once, cache forever. Where should caching happen — in `getBean()` or in a separate method?

```java
private Object doGetBean(String name) {
    // 1. Check cache FIRST — this is what makes singletons work
    Object singleton = this.singletonObjects.get(name);
    if (singleton != null) {
        return singleton;
    }

    // 2. Create if not cached
    BeanDefinition bd = getBeanDefinition(name);
    Object bean = createBean(name, bd);

    // 3. Cache AFTER creation — next call returns the same instance
    if (bd.isSingleton()) {
        this.singletonObjects.put(name, bean);
    }

    return bean;
}
```

Separating `doGetBean()` from the public `getBean()` methods follows the real Spring pattern — `AbstractBeanFactory.doGetBean()` is a 200+ line method that handles lazy singletons, circular dependencies, prototype detection, and more.

</details>

## 1.7 Tests

### Unit Tests — BeanDefinition

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/config/BeanDefinitionTest.java`

```java
package com.iris.framework.beans.factory.config;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BeanDefinitionTest {

    @Test
    void shouldStoreBeanClass_WhenConstructedWithClass() {
        BeanDefinition bd = new BeanDefinition(String.class);
        assertThat(bd.getBeanClass()).isEqualTo(String.class);
    }

    @Test
    void shouldDefaultToSingletonScope() {
        BeanDefinition bd = new BeanDefinition(String.class);
        assertThat(bd.isSingleton()).isTrue();
        assertThat(bd.getScope()).isEqualTo(BeanDefinition.SCOPE_SINGLETON);
    }

    @Test
    void shouldStoreSupplier_WhenConstructedWithSupplier() {
        BeanDefinition bd = new BeanDefinition(String.class, () -> "hello");
        assertThat(bd.getSupplier()).isNotNull();
        assertThat(bd.getSupplier().get()).isEqualTo("hello");
    }

    @Test
    void shouldAllowSettingSupplierAfterConstruction() {
        BeanDefinition bd = new BeanDefinition(String.class);
        bd.setSupplier(() -> "world");
        assertThat(bd.getSupplier().get()).isEqualTo("world");
    }

    @Test
    void shouldReportNonSingleton_WhenScopeChanged() {
        BeanDefinition bd = new BeanDefinition(String.class);
        bd.setScope("prototype");
        assertThat(bd.isSingleton()).isFalse();
    }
}
```

### Unit Tests — DefaultBeanFactory

**New file:** `iris-framework/src/test/java/com/iris/framework/beans/factory/support/DefaultBeanFactoryTest.java`

```java
package com.iris.framework.beans.factory.support;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.config.BeanDefinition;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DefaultBeanFactoryTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
    }

    @Test
    void shouldRegisterBeanDefinition_WhenGivenNameAndDefinition() { ... }
    @Test
    void shouldReturnDefinitionNames_InRegistrationOrder() { ... }
    @Test
    void shouldCreateBean_WhenGetBeanCalledByName() { ... }
    @Test
    void shouldReturnSameInstance_WhenSingletonBeanRequestedTwice() { ... }
    @Test
    void shouldUseSupplier_WhenBeanDefinitionHasOne() { ... }
    @Test
    void shouldResolveBean_WhenSingleCandidateMatchesType() { ... }
    @Test
    void shouldThrowNoUniqueBeanDefinitionException_WhenMultipleCandidatesMatchType() { ... }
    @Test
    void shouldThrowBeanCreationException_WhenClassHasNoDefaultConstructor() { ... }
    // ... 16 tests total — see Complete Code section for full listing
}
```

**Run:** `./gradlew test` — expected: all 21 tests pass (5 BeanDefinition + 16 DefaultBeanFactory)

---

## 1.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why two maps?** The `beanDefinitionMap` stores metadata (class, scope, supplier) while `singletonObjects` stores actual instances. This separation enables lazy initialization — you can register hundreds of beans but only instantiate the ones that are actually requested. In the real Spring Framework, these maps live in different classes: `DefaultListableBeanFactory` owns the definition map, while its grandparent `DefaultSingletonBeanRegistry` owns the singleton cache.
> - **When this falls apart:** Our `ConcurrentHashMap` provides basic thread safety, but the real framework needs a sophisticated three-level cache (`singletonObjects` → `earlySingletonObjects` → `singletonFactories`) to solve circular dependencies. We'll encounter this limitation in Chapter 2 when beans depend on each other.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why separate BeanFactory from BeanDefinitionRegistry?** The read/write split follows the Interface Segregation Principle. Application code that consumes beans (`getBean()`) should never know how to register them. This is why Spring's `ApplicationContext` extends `BeanFactory` but not `BeanDefinitionRegistry` — even though the underlying `DefaultListableBeanFactory` implements both.
> - **Real-world parallel:** This same pattern appears in `javax.sql.DataSource` (get connections) vs connection pool configuration (register connections). Consumers only see the read side.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `isAssignableFrom` for type lookup?** `requiredType.isAssignableFrom(bd.getBeanClass())` checks at the class level: "can a variable of type `requiredType` hold an instance of `beanClass`?" This means asking for `CharSequence.class` finds a `StringBuilder` bean, and asking for `Object.class` would find everything. The real framework adds qualifiers, `@Primary`, and `@Fallback` annotations to disambiguate when multiple beans match — we'll encounter that in later chapters.
> -----------------------------------------------------------

## 1.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `BeanDefinition` (concrete class) | `BeanDefinition` (interface) | `BeanDefinition.java:42` | Real is an interface with `RootBeanDefinition`, `GenericBeanDefinition`, etc. supporting constructor args, property values, factory methods, parent definitions |
| `BeanFactory` (single interface) | `BeanFactory` → `ListableBeanFactory` → `ConfigurableBeanFactory` | `BeanFactory.java:122` | Real has a deep hierarchy: `HierarchicalBeanFactory` (parent chains), `ListableBeanFactory` (iterate by type), `AutowireCapableBeanFactory` (DI) |
| `BeanDefinitionRegistry` | `BeanDefinitionRegistry` | `BeanDefinitionRegistry.java:47` | Real extends `AliasRegistry` for bean aliases; supports `removeBeanDefinition()` and override checking |
| `DefaultBeanFactory` | `DefaultListableBeanFactory` | `DefaultListableBeanFactory.java:131` | Real extends `AbstractAutowireCapableBeanFactory` → `AbstractBeanFactory` → `DefaultSingletonBeanRegistry`. 2000+ lines handling circular deps, type caching, factory beans, scopes |
| `doGetBean()` (10 lines) | `AbstractBeanFactory.doGetBean()` | `AbstractBeanFactory.java:245` | Real is ~200 lines: handles aliases, prototype detection, parent delegation, circular dependency checks, `FactoryBean` unwrapping |
| `singletonObjects` map | `DefaultSingletonBeanRegistry.singletonObjects` | `DefaultSingletonBeanRegistry.java:86` | Real uses a three-level cache for circular dependency resolution |
| `registerBeanDefinition()` (3 lines) | `DefaultListableBeanFactory.registerBeanDefinition()` | `DefaultListableBeanFactory.java:1245` | Real validates the definition, checks for overrides, handles alias conflicts, uses `synchronized` for thread safety during startup |

## 1.10 Complete Code

### Production Code

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/config/BeanDefinition.java` [NEW]

```java
package com.iris.framework.beans.factory.config;

import java.util.function.Supplier;

/**
 * Holds the metadata for a single bean: its class, scope, and an optional
 * supplier for programmatic instantiation.
 *
 * <p>In the real Spring Framework this is an interface with multiple implementations
 * (RootBeanDefinition, GenericBeanDefinition, etc.) supporting constructor args,
 * property values, factory methods, and more. We simplify to a single concrete class
 * with just the essentials: class, scope, and supplier.
 *
 * @see org.springframework.beans.factory.config.BeanDefinition
 */
public class BeanDefinition {

    public static final String SCOPE_SINGLETON = "singleton";

    private final Class<?> beanClass;
    private String scope = SCOPE_SINGLETON;
    private Supplier<?> supplier;

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
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/BeanFactory.java` [NEW]

```java
package com.iris.framework.beans.factory;

/**
 * The root interface for accessing the Iris bean container.
 *
 * <p>Provides methods to retrieve beans by name and by type. This is the
 * read-only client view — registration happens through
 * {@link com.iris.framework.beans.factory.support.BeanDefinitionRegistry}.
 *
 * <p>In the real Spring Framework, BeanFactory is extended by ListableBeanFactory,
 * HierarchicalBeanFactory, and ConfigurableBeanFactory. We collapse them into
 * this single interface for simplicity.
 *
 * @see com.iris.framework.beans.factory.support.DefaultBeanFactory
 * @see org.springframework.beans.factory.BeanFactory
 */
public interface BeanFactory {

    /**
     * Return a bean instance by name.
     *
     * @param name the bean name
     * @return the bean instance (never null)
     * @throws NoSuchBeanDefinitionException if no bean with this name exists
     * @throws BeanCreationException if the bean could not be instantiated
     */
    Object getBean(String name);

    /**
     * Return a bean instance by name, cast to the required type.
     *
     * @param name the bean name
     * @param requiredType the expected type
     * @return the bean instance cast to T
     * @throws NoSuchBeanDefinitionException if no bean with this name exists
     * @throws BeanCreationException if the bean could not be instantiated
     */
    <T> T getBean(String name, Class<T> requiredType);

    /**
     * Return the single bean matching the required type.
     *
     * @param requiredType the type to match
     * @return the matching bean instance
     * @throws NoSuchBeanDefinitionException if no matching bean exists
     * @throws NoUniqueBeanDefinitionException if multiple beans match
     * @throws BeanCreationException if the bean could not be instantiated
     */
    <T> T getBean(Class<T> requiredType);

    /**
     * Check whether a bean with the given name is registered.
     */
    boolean containsBean(String name);

    /**
     * Return whether the named bean is a singleton.
     */
    boolean isSingleton(String name);
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/support/BeanDefinitionRegistry.java` [NEW]

```java
package com.iris.framework.beans.factory.support;

import com.iris.framework.beans.factory.config.BeanDefinition;

/**
 * Interface for registries that hold bean definitions.
 *
 * <p>This separates the registration concern from the retrieval concern
 * ({@link com.iris.framework.beans.factory.BeanFactory}). In the real Spring
 * Framework, DefaultListableBeanFactory implements both interfaces.
 *
 * @see com.iris.framework.beans.factory.BeanFactory
 * @see DefaultBeanFactory
 * @see org.springframework.beans.factory.support.BeanDefinitionRegistry
 */
public interface BeanDefinitionRegistry {

    /**
     * Register a bean definition under the given name.
     */
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition);

    /**
     * Return the bean definition for the given name.
     *
     * @throws com.iris.framework.beans.factory.NoSuchBeanDefinitionException
     *         if no definition exists for this name
     */
    BeanDefinition getBeanDefinition(String beanName);

    /**
     * Check if a bean definition with the given name is registered.
     */
    boolean containsBeanDefinition(String beanName);

    /**
     * Return the names of all registered bean definitions.
     */
    String[] getBeanDefinitionNames();

    /**
     * Return the count of registered bean definitions.
     */
    int getBeanDefinitionCount();
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java` [NEW]

```java
package com.iris.framework.beans.factory.support;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.config.BeanDefinition;

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
            if (bd.getSupplier() != null) {
                return bd.getSupplier().get();
            }
            return bd.getBeanClass().getDeclaredConstructor().newInstance();
        } catch (Exception ex) {
            throw new BeanCreationException(name,
                    "Instantiation of bean failed", ex);
        }
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/NoSuchBeanDefinitionException.java` [NEW]

```java
package com.iris.framework.beans.factory;

/**
 * Thrown when a BeanFactory is asked for a bean that it cannot find.
 *
 * @see org.springframework.beans.factory.NoSuchBeanDefinitionException
 */
public class NoSuchBeanDefinitionException extends RuntimeException {

    private final String beanName;

    public NoSuchBeanDefinitionException(String beanName) {
        super("No bean named '" + beanName + "' available");
        this.beanName = beanName;
    }

    public NoSuchBeanDefinitionException(Class<?> type) {
        super("No qualifying bean of type '" + type.getName() + "' available");
        this.beanName = null;
    }

    public String getBeanName() {
        return beanName;
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/NoUniqueBeanDefinitionException.java` [NEW]

```java
package com.iris.framework.beans.factory;

import java.util.Collection;

/**
 * Thrown when a BeanFactory is asked for a bean by type and multiple
 * candidates are found.
 *
 * @see org.springframework.beans.factory.NoUniqueBeanDefinitionException
 */
public class NoUniqueBeanDefinitionException extends NoSuchBeanDefinitionException {

    private final int numberOfBeansFound;
    private final Collection<String> beanNamesFound;

    public NoUniqueBeanDefinitionException(Class<?> type, Collection<String> beanNamesFound) {
        super("No qualifying bean of type '" + type.getName() +
                "' available: expected single matching bean but found " +
                beanNamesFound.size() + ": " + beanNamesFound);
        this.numberOfBeansFound = beanNamesFound.size();
        this.beanNamesFound = beanNamesFound;
    }

    public int getNumberOfBeansFound() {
        return numberOfBeansFound;
    }

    public Collection<String> getBeanNamesFound() {
        return beanNamesFound;
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/beans/factory/BeanCreationException.java` [NEW]

```java
package com.iris.framework.beans.factory;

/**
 * Thrown when a BeanFactory encounters an error creating a bean instance.
 *
 * @see org.springframework.beans.factory.BeanCreationException
 */
public class BeanCreationException extends RuntimeException {

    private final String beanName;

    public BeanCreationException(String beanName, String message) {
        super("Error creating bean with name '" + beanName + "': " + message);
        this.beanName = beanName;
    }

    public BeanCreationException(String beanName, String message, Throwable cause) {
        super("Error creating bean with name '" + beanName + "': " + message, cause);
        this.beanName = beanName;
    }

    public String getBeanName() {
        return beanName;
    }
}
```

### Test Code

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/config/BeanDefinitionTest.java` [NEW]

```java
package com.iris.framework.beans.factory.config;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BeanDefinitionTest {

    @Test
    void shouldStoreBeanClass_WhenConstructedWithClass() {
        BeanDefinition bd = new BeanDefinition(String.class);

        assertThat(bd.getBeanClass()).isEqualTo(String.class);
    }

    @Test
    void shouldDefaultToSingletonScope() {
        BeanDefinition bd = new BeanDefinition(String.class);

        assertThat(bd.isSingleton()).isTrue();
        assertThat(bd.getScope()).isEqualTo(BeanDefinition.SCOPE_SINGLETON);
    }

    @Test
    void shouldStoreSupplier_WhenConstructedWithSupplier() {
        BeanDefinition bd = new BeanDefinition(String.class, () -> "hello");

        assertThat(bd.getSupplier()).isNotNull();
        assertThat(bd.getSupplier().get()).isEqualTo("hello");
    }

    @Test
    void shouldAllowSettingSupplierAfterConstruction() {
        BeanDefinition bd = new BeanDefinition(String.class);
        bd.setSupplier(() -> "world");

        assertThat(bd.getSupplier().get()).isEqualTo("world");
    }

    @Test
    void shouldReportNonSingleton_WhenScopeChanged() {
        BeanDefinition bd = new BeanDefinition(String.class);
        bd.setScope("prototype");

        assertThat(bd.isSingleton()).isFalse();
    }
}
```

#### File: `iris-framework/src/test/java/com/iris/framework/beans/factory/support/DefaultBeanFactoryTest.java` [NEW]

```java
package com.iris.framework.beans.factory.support;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.config.BeanDefinition;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DefaultBeanFactoryTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
    }

    // ── Registration ─────────────────────────────────────────────

    @Test
    void shouldRegisterBeanDefinition_WhenGivenNameAndDefinition() {
        BeanDefinition bd = new BeanDefinition(StringBuilder.class);
        factory.registerBeanDefinition("builder", bd);

        assertThat(factory.containsBeanDefinition("builder")).isTrue();
        assertThat(factory.getBeanDefinition("builder")).isSameAs(bd);
    }

    @Test
    void shouldReturnDefinitionNames_InRegistrationOrder() {
        factory.registerBeanDefinition("alpha", new BeanDefinition(String.class));
        factory.registerBeanDefinition("beta", new BeanDefinition(Integer.class));
        factory.registerBeanDefinition("gamma", new BeanDefinition(Long.class));

        assertThat(factory.getBeanDefinitionNames())
                .containsExactly("alpha", "beta", "gamma");
    }

    @Test
    void shouldReturnCorrectCount_WhenMultipleBeansRegistered() {
        factory.registerBeanDefinition("a", new BeanDefinition(String.class));
        factory.registerBeanDefinition("b", new BeanDefinition(Integer.class));

        assertThat(factory.getBeanDefinitionCount()).isEqualTo(2);
    }

    @Test
    void shouldThrowNoSuchBeanDefinitionException_WhenDefinitionNotFound() {
        assertThatThrownBy(() -> factory.getBeanDefinition("missing"))
                .isInstanceOf(NoSuchBeanDefinitionException.class)
                .hasMessageContaining("missing");
    }

    // ── getBean by name ──────────────────────────────────────────

    @Test
    void shouldCreateBean_WhenGetBeanCalledByName() {
        factory.registerBeanDefinition("builder",
                new BeanDefinition(StringBuilder.class));

        Object bean = factory.getBean("builder");

        assertThat(bean).isInstanceOf(StringBuilder.class);
    }

    @Test
    void shouldReturnSameInstance_WhenSingletonBeanRequestedTwice() {
        factory.registerBeanDefinition("builder",
                new BeanDefinition(StringBuilder.class));

        Object first = factory.getBean("builder");
        Object second = factory.getBean("builder");

        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldUseSupplier_WhenBeanDefinitionHasOne() {
        BeanDefinition bd = new BeanDefinition(String.class, () -> "custom-value");
        factory.registerBeanDefinition("myString", bd);

        Object bean = factory.getBean("myString");

        assertThat(bean).isEqualTo("custom-value");
    }

    @Test
    void shouldThrowNoSuchBeanDefinitionException_WhenBeanNotRegistered() {
        assertThatThrownBy(() -> factory.getBean("nonexistent"))
                .isInstanceOf(NoSuchBeanDefinitionException.class)
                .hasMessageContaining("nonexistent");
    }

    // ── getBean by name and type ─────────────────────────────────

    @Test
    void shouldReturnTypedBean_WhenGetBeanCalledWithNameAndType() {
        factory.registerBeanDefinition("builder",
                new BeanDefinition(StringBuilder.class));

        StringBuilder bean = factory.getBean("builder", StringBuilder.class);

        assertThat(bean).isNotNull();
    }

    @Test
    void shouldThrowBeanCreationException_WhenTypeMismatch() {
        factory.registerBeanDefinition("builder",
                new BeanDefinition(StringBuilder.class));

        assertThatThrownBy(() -> factory.getBean("builder", Integer.class))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("required type");
    }

    // ── getBean by type ──────────────────────────────────────────

    @Test
    void shouldResolveBean_WhenSingleCandidateMatchesType() {
        factory.registerBeanDefinition("builder",
                new BeanDefinition(StringBuilder.class));

        CharSequence bean = factory.getBean(CharSequence.class);

        assertThat(bean).isInstanceOf(StringBuilder.class);
    }

    @Test
    void shouldThrowNoSuchBeanDefinitionException_WhenNoCandidateMatchesType() {
        factory.registerBeanDefinition("builder",
                new BeanDefinition(StringBuilder.class));

        assertThatThrownBy(() -> factory.getBean(Integer.class))
                .isInstanceOf(NoSuchBeanDefinitionException.class);
    }

    @Test
    void shouldThrowNoUniqueBeanDefinitionException_WhenMultipleCandidatesMatchType() {
        factory.registerBeanDefinition("sb1",
                new BeanDefinition(StringBuilder.class));
        factory.registerBeanDefinition("sb2",
                new BeanDefinition(StringBuilder.class));

        assertThatThrownBy(() -> factory.getBean(CharSequence.class))
                .isInstanceOf(NoUniqueBeanDefinitionException.class)
                .hasMessageContaining("sb1")
                .hasMessageContaining("sb2");
    }

    // ── containsBean / isSingleton ───────────────────────────────

    @Test
    void shouldReturnTrue_WhenContainsBeanCalledForRegisteredBean() {
        factory.registerBeanDefinition("x", new BeanDefinition(String.class));

        assertThat(factory.containsBean("x")).isTrue();
        assertThat(factory.containsBean("y")).isFalse();
    }

    @Test
    void shouldReturnTrue_WhenIsSingletonCalledForSingletonBean() {
        factory.registerBeanDefinition("x", new BeanDefinition(String.class));

        assertThat(factory.isSingleton("x")).isTrue();
    }

    // ── Error handling ───────────────────────────────────────────

    @Test
    void shouldThrowBeanCreationException_WhenClassHasNoDefaultConstructor() {
        factory.registerBeanDefinition("bad",
                new BeanDefinition(Integer.class));

        assertThatThrownBy(() -> factory.getBean("bad"))
                .isInstanceOf(BeanCreationException.class)
                .hasMessageContaining("Instantiation of bean failed");
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **BeanDefinition** | Metadata describing a bean — its class, scope, and how to create it (supplier or reflection) |
| **BeanFactory** | The read API — `getBean()` by name or by type |
| **BeanDefinitionRegistry** | The write API — `registerBeanDefinition()` |
| **DefaultBeanFactory** | The single implementation combining both APIs, with a two-map design separating definitions from instances |
| **Singleton scope** | Create once, cache forever — the default and most common bean scope |
| **Lazy instantiation** | Beans are created on first `getBean()` call, not at registration time |

**Next: Chapter 2 — Dependency Injection** — Right now beans are isolated: each is created independently with no awareness of other beans. We'll add `@Autowired` so the container automatically wires dependencies between beans during creation.
