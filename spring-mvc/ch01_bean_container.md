# Chapter 1: Bean Container

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| No simplified framework exists yet | No central registry to hold controllers, services, and other components — future features (DispatcherServlet, HandlerMapping) would have no way to discover the objects they need | Build a `SimpleBeanContainer` that stores singleton beans in a `Map`, supports registration by type/name, retrieval by type (including interface and superclass lookup), and singleton semantics |

---

## 1.1 The Core Data Structure

This is a foundation chapter — there is no existing system to plug into. The `SimpleBeanContainer` itself IS the integration point that every future feature will connect to:

- **Chapter 2 (DispatcherServlet)** will call `container.getBeansOfType(HandlerMapping.class)` to find strategy beans
- **Chapter 3 (HandlerMapping)** will call `container.getBeanNames()` and iterate all beans to find `@Controller` classes
- **Chapter 17 (Component Scanning)** will call `container.registerBean(name, instance)` for each discovered class

The core data structure is a `ConcurrentHashMap<String, Object>` — the same thread-safe map that the real Spring uses at `DefaultSingletonBeanRegistry.java:86`.

**New file:** `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java`

```java
public class SimpleBeanContainer implements BeanContainer {

    // The central data structure — maps bean name → singleton instance.
    // Real Spring equivalent: DefaultSingletonBeanRegistry.singletonObjects
    private final Map<String, Object> singletonMap = new ConcurrentHashMap<>(64);

    // Preserves registration order, matching real Spring's beanDefinitionNames list.
    private final List<String> beanNames = new ArrayList<>();
```

Two key decisions here:

1. **`ConcurrentHashMap` over `HashMap`** — In a web server, multiple threads handle requests simultaneously and all read from this map. `ConcurrentHashMap` provides lock-free reads and segmented writes. The real `DefaultSingletonBeanRegistry` makes the same choice.

2. **Separate name list for ordering** — `ConcurrentHashMap` doesn't preserve insertion order. We maintain a parallel `ArrayList<String>` (like real Spring's `beanDefinitionNames`) so that `getBeansOfType()` returns beans in a deterministic, registration-order sequence. This matters when multiple beans match and the first one wins.

To make this work, we need:
- A `BeanContainer` interface (the API contract)
- Exception types for "not found" and "not unique"
- The `SimpleBeanContainer` implementation

## 1.2 The BeanContainer Interface

The interface defines the contract that all other features will program against. It's modeled on `BeanFactory` but drastically simplified — 6 methods instead of ~15.

**New file:** `src/main/java/com/simplespringmvc/container/BeanContainer.java`

```java
package com.simplespringmvc.container;

import java.util.List;
import java.util.Map;

public interface BeanContainer {

    void registerBean(String name, Object bean);

    void registerBean(Object bean);

    Object getBean(String name);

    <T> T getBean(Class<T> type);

    <T> Map<String, T> getBeansOfType(Class<T> type);

    List<String> getBeanNames();

    boolean containsBean(String name);
}
```

Note what's missing vs the real `BeanFactory`: no scopes, no lazy init, no aliases, no `FactoryBean`, no parent hierarchy. We'll add capabilities as later chapters demand them.

## 1.3 Exception Types

Two exception classes for the two ways a lookup can fail:

**New file:** `src/main/java/com/simplespringmvc/container/BeanNotFoundException.java`

```java
package com.simplespringmvc.container;

public class BeanNotFoundException extends RuntimeException {
    public BeanNotFoundException(String message) {
        super(message);
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/container/NoUniqueBeanException.java`

```java
package com.simplespringmvc.container;

public class NoUniqueBeanException extends RuntimeException {
    public NoUniqueBeanException(String message) {
        super(message);
    }
}
```

Both are unchecked (`RuntimeException`). The real Spring does the same — `NoSuchBeanDefinitionException` and `NoUniqueBeanDefinitionException` extend `BeansException` which extends `RuntimeException`. Container failures are programming errors, not recoverable conditions.

## 1.4 The SimpleBeanContainer Implementation

The full implementation has three groups of methods: registration, name-based lookup, and type-based lookup.

**New file:** `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java`

### Registration

```java
@Override
public void registerBean(String name, Object bean) {
    if (name == null || name.isBlank()) {
        throw new IllegalArgumentException("Bean name must not be null or blank");
    }
    if (bean == null) {
        throw new IllegalArgumentException("Bean instance must not be null");
    }
    singletonMap.put(name, bean);
    if (!beanNames.contains(name)) {
        beanNames.add(name);
    }
}

@Override
public void registerBean(Object bean) {
    String name = deriveBeanName(bean.getClass());
    registerBean(name, bean);
}
```

`registerBean(Object)` derives the name automatically — `UserController` becomes `"userController"`. This mirrors Spring's `AnnotationBeanNameGenerator` which calls `Introspector.decapitalize()`.

### Name-based lookup

```java
@Override
public Object getBean(String name) {
    Object bean = singletonMap.get(name);
    if (bean == null) {
        throw new BeanNotFoundException("No bean named '" + name + "' is registered");
    }
    return bean;
}
```

### Type-based lookup (the interesting part)

```java
@Override
public <T> T getBean(Class<T> type) {
    Map<String, T> matches = getBeansOfType(type);

    if (matches.isEmpty()) {
        throw new BeanNotFoundException(
                "No bean of type '" + type.getName() + "' is registered");
    }
    if (matches.size() > 1) {
        throw new NoUniqueBeanException(
                "Expected single bean of type '" + type.getName()
                        + "' but found " + matches.size() + ": " + matches.keySet());
    }

    return matches.values().iterator().next();
}

@Override
@SuppressWarnings("unchecked")
public <T> Map<String, T> getBeansOfType(Class<T> type) {
    Map<String, T> result = new LinkedHashMap<>();
    for (String name : beanNames) {
        Object bean = singletonMap.get(name);
        if (type.isInstance(bean)) {
            result.put(name, (T) bean);
        }
    }
    return result;
}
```

The magic is in `type.isInstance(bean)` — this single call checks the bean's concrete class, all its superclasses, AND all interfaces in the hierarchy. If you register an `OrderService implements Runnable`, then `getBean(Runnable.class)` will find it. This is the same mechanism the real Spring uses underneath `ResolvableType.isAssignableFrom()` (for raw class matching).

## 1.5 Try It Yourself

<details>
<summary>Challenge: Build the container from scratch</summary>

Delete all files under `src/main/java/com/simplespringmvc/container/` and rebuild from these requirements:

1. Create a `BeanContainer` interface with methods for:
   - Registering a bean by name + instance
   - Registering a bean by instance only (auto-derive name)
   - Retrieving by name (throws if not found)
   - Retrieving by type (throws if not found or not unique)
   - Getting all beans of a type as a `Map<String, T>`
   - Listing all bean names
   - Checking existence by name

2. Create `SimpleBeanContainer` backed by a `ConcurrentHashMap`

3. Run `./gradlew test` — all 18 tests should pass

Hint: The key to type-based lookup is `Class.isInstance(Object)` — it checks assignability including interfaces and superclasses, in a single call.

</details>

<details>
<summary>Challenge: What happens when you register two beans of the same type and call getBean(Type.class)?</summary>

`NoUniqueBeanException` is thrown. The real Spring adds disambiguation via `@Primary` and `@Priority` annotations. Our simplified version doesn't need that yet — if you need a specific bean when multiple match, use `getBean(name)` instead.

To verify:
```java
container.registerBean("a", new OrderService());
container.registerBean("b", new OrderService());
container.getBean(OrderService.class); // throws NoUniqueBeanException
```

</details>

## 1.6 Tests

**New file:** `src/test/java/com/simplespringmvc/container/SimpleBeanContainerTest.java`

```java
package com.simplespringmvc.container;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SimpleBeanContainerTest {

    private SimpleBeanContainer container;

    @BeforeEach
    void setUp() {
        container = new SimpleBeanContainer();
    }

    @Test
    void shouldRetrieveBean_WhenRegisteredByName() {
        var service = new OrderService();
        container.registerBean("orderService", service);
        assertThat(container.getBean("orderService")).isSameAs(service);
    }

    @Test
    void shouldReturnSameInstance_WhenRetrievedMultipleTimes() {
        var service = new OrderService();
        container.registerBean("orderService", service);
        assertThat(container.getBean("orderService"))
                .isSameAs(container.getBean("orderService"));
    }

    @Test
    void shouldDeriveNameFromClass_WhenRegisteredWithoutName() {
        var service = new OrderService();
        container.registerBean(service);
        assertThat(container.getBean("orderService")).isSameAs(service);
    }

    @Test
    void shouldRetrieveBean_WhenLookingUpByInterface() {
        var service = new OrderService();
        container.registerBean("orderService", service);
        assertThat(container.getBean(Runnable.class)).isSameAs(service);
    }

    @Test
    void shouldRetrieveBean_WhenLookingUpBySuperclass() {
        var service = new SpecialOrderService();
        container.registerBean("special", service);
        assertThat(container.getBean(OrderService.class)).isSameAs(service);
    }

    @Test
    void shouldThrowNoUniqueBeanException_WhenMultipleBeansMatchType() {
        container.registerBean("service1", new OrderService());
        container.registerBean("service2", new OrderService());
        assertThatThrownBy(() -> container.getBean(OrderService.class))
                .isInstanceOf(NoUniqueBeanException.class);
    }

    @Test
    void shouldReturnAllMatchingBeans_WhenCallingGetBeansOfType() {
        container.registerBean("s1", new OrderService());
        container.registerBean("s2", new OrderService());
        container.registerBean("other", "a string");
        assertThat(container.getBeansOfType(Runnable.class)).hasSize(2);
    }

    @Test
    void shouldPreserveRegistrationOrder_WhenCallingGetBeansOfType() {
        container.registerBean("c", new OrderService());
        container.registerBean("a", new OrderService());
        container.registerBean("b", new OrderService());
        assertThat(container.getBeansOfType(OrderService.class).keySet())
                .containsExactly("c", "a", "b");
    }

    // Fixtures
    static class OrderService implements Runnable {
        @Override public void run() { }
    }

    static class SpecialOrderService extends OrderService { }
}
```

**Run:** `./gradlew test` — expected: all 18 tests pass

---

## 1.7 Why This Works

> ★ **Insight** -------------------------------------------
> **The Registry Pattern simplifies everything downstream.** By making the container a simple `Map<String, Object>`, every future feature can assume its dependencies already exist and just ask for them. The DispatcherServlet doesn't need to know how a `HandlerMapping` was created — it calls `getBeansOfType(HandlerMapping.class)` and works with whatever comes back. This decoupling is the core benefit of IoC: the "what" (what objects exist) is separated from the "how" (how they're created and wired).
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **`Class.isInstance()` does all the heavy lifting for type-based lookup.** You don't need to manually walk superclass chains or interface hierarchies. Java's reflection API handles it in one call: `type.isInstance(bean)` returns `true` if the bean's class is the same as, a subclass of, or implements the given type. The real Spring's `ResolvableType.isAssignableFrom()` adds generic type handling on top of this same mechanism — but for 90% of use cases, `isInstance` is all you need.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Singleton semantics come "for free" with a Map.** Our container stores the actual object instance, so `getBean()` always returns the same reference. The real Spring has a separate `DefaultSingletonBeanRegistry` with a three-level cache to handle circular references during construction. We skip all that because our beans are pre-instantiated — there's no construction phase where circularity can occur. This simplification is why we don't need `BeanDefinition` yet.
> -----------------------------------------------------------

## 1.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `BeanContainer` | `BeanFactory` | `BeanFactory.java:155` | Real version has `getBean(name, type)`, `getBean(type, args)`, `ObjectProvider`, aliases, scope checks |
| `SimpleBeanContainer.singletonMap` | `DefaultSingletonBeanRegistry.singletonObjects` | `DefaultSingletonBeanRegistry.java:86` | Real version has 3-level cache (singletonObjects, earlySingletonObjects, singletonFactories) for circular reference resolution |
| `SimpleBeanContainer.beanNames` | `DefaultListableBeanFactory.beanDefinitionNames` | `DefaultListableBeanFactory.java:203` | Real version uses copy-on-write for concurrent registration during startup |
| `registerBean()` | `registerBeanDefinition()` | `DefaultListableBeanFactory.java:1245` | Real version registers metadata (BeanDefinition), not instances — supports lazy init, scopes, override checks |
| `getBean(Class)` | `resolveBean()` → `resolveNamedBean()` | `DefaultListableBeanFactory.java:555` | Real version disambiguates via `@Primary`, `@Priority`, `@Fallback`; checks parent factory; uses `ResolvableType` for generics |
| `getBeansOfType()` → `isInstance()` | `doGetBeanNamesForType()` → `isTypeMatch()` | `DefaultListableBeanFactory.java:619` | Real version handles FactoryBeans, lazy init, frozen config caching, and generic type matching |
| `BeanNotFoundException` | `NoSuchBeanDefinitionException` | `NoSuchBeanDefinitionException.java` | Real version includes bean type, ResolvableType, and number of beans found |
| `NoUniqueBeanException` | `NoUniqueBeanDefinitionException` | `NoUniqueBeanDefinitionException.java` | Real version lists all matching bean names and their types |

## 1.9 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/container/BeanContainer.java` [NEW]

```java
package com.simplespringmvc.container;

import java.util.List;
import java.util.Map;

/**
 * The root interface for accessing the simplified bean container.
 *
 * Maps to: {@code org.springframework.beans.factory.BeanFactory}
 *
 * Simplifications vs real Spring:
 * - No scopes (singleton only)
 * - No lazy initialization
 * - No dependency injection
 * - No aliases
 * - No FactoryBean support
 * - No parent factory hierarchy
 */
public interface BeanContainer {

    /**
     * Register a pre-instantiated singleton bean under an explicit name.
     */
    void registerBean(String name, Object bean);

    /**
     * Register a pre-instantiated singleton bean using its class simple name
     * (first letter lowercased) as the name.
     */
    void registerBean(Object bean);

    /**
     * Retrieve a bean by name.
     *
     * @throws BeanNotFoundException if no bean exists with this name
     */
    Object getBean(String name);

    /**
     * Retrieve the single bean matching the given type.
     * Checks the bean's class AND all interfaces it implements.
     *
     * @throws BeanNotFoundException if no bean matches
     * @throws NoUniqueBeanException if more than one bean matches
     */
    <T> T getBean(Class<T> type);

    /**
     * Return all beans assignable to the given type, keyed by bean name.
     */
    <T> Map<String, T> getBeansOfType(Class<T> type);

    /**
     * Return all registered bean names.
     */
    List<String> getBeanNames();

    /**
     * Check whether a bean with the given name is registered.
     */
    boolean containsBean(String name);
}
```

#### File: `src/main/java/com/simplespringmvc/container/BeanNotFoundException.java` [NEW]

```java
package com.simplespringmvc.container;

/**
 * Thrown when a requested bean cannot be found in the container.
 *
 * Maps to: {@code org.springframework.beans.factory.NoSuchBeanDefinitionException}
 */
public class BeanNotFoundException extends RuntimeException {

    public BeanNotFoundException(String message) {
        super(message);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/container/NoUniqueBeanException.java` [NEW]

```java
package com.simplespringmvc.container;

/**
 * Thrown when a type-based lookup matches more than one bean.
 *
 * Maps to: {@code org.springframework.beans.factory.NoUniqueBeanDefinitionException}
 */
public class NoUniqueBeanException extends RuntimeException {

    public NoUniqueBeanException(String message) {
        super(message);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java` [NEW]

```java
package com.simplespringmvc.container;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * A minimal IoC container that stores singleton beans in a Map and supports
 * retrieval by name or type (including interface lookup).
 *
 * Maps to: {@code org.springframework.beans.factory.support.DefaultListableBeanFactory}
 *
 * The real DefaultListableBeanFactory stores BeanDefinitions (metadata) separately
 * from singleton instances, supports lazy initialization, prototype scope, and
 * complex resolution strategies (@Primary, @Priority). Our SimpleBeanContainer
 * stores only pre-instantiated singletons — enough to demonstrate the registry
 * pattern that the entire framework depends on.
 *
 * Core data structure:
 * <pre>
 *   singletonMap: ConcurrentHashMap&lt;String, Object&gt;
 *       │
 *       ├── "userController"  → UserController instance
 *       ├── "orderService"    → OrderService instance
 *       └── "dataSource"      → DataSource instance
 * </pre>
 *
 * Every future feature connects here:
 * - DispatcherServlet (ch02) retrieves strategy beans from the container
 * - HandlerMapping (ch03) iterates all beans to find @Controller classes
 * - Component Scanning (ch17) auto-registers discovered classes
 */
public class SimpleBeanContainer implements BeanContainer {

    /**
     * The central data structure — maps bean name → singleton instance.
     *
     * Real Spring equivalent: DefaultSingletonBeanRegistry.singletonObjects
     * (ConcurrentHashMap<String, Object>) at DefaultSingletonBeanRegistry.java:86
     *
     * We use ConcurrentHashMap for thread-safety, matching what the real framework does.
     */
    private final Map<String, Object> singletonMap = new ConcurrentHashMap<>(64);

    /**
     * Preserves registration order, matching real Spring's beanDefinitionNames list.
     * This ensures getBeansOfType returns beans in a deterministic order.
     */
    private final List<String> beanNames = new ArrayList<>();

    @Override
    public void registerBean(String name, Object bean) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Bean name must not be null or blank");
        }
        if (bean == null) {
            throw new IllegalArgumentException("Bean instance must not be null");
        }
        singletonMap.put(name, bean);
        if (!beanNames.contains(name)) {
            beanNames.add(name);
        }
    }

    @Override
    public void registerBean(Object bean) {
        String name = deriveBeanName(bean.getClass());
        registerBean(name, bean);
    }

    @Override
    public Object getBean(String name) {
        Object bean = singletonMap.get(name);
        if (bean == null) {
            throw new BeanNotFoundException("No bean named '" + name + "' is registered");
        }
        return bean;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> type) {
        Map<String, T> matches = getBeansOfType(type);

        if (matches.isEmpty()) {
            throw new BeanNotFoundException(
                    "No bean of type '" + type.getName() + "' is registered");
        }
        if (matches.size() > 1) {
            throw new NoUniqueBeanException(
                    "Expected single bean of type '" + type.getName()
                            + "' but found " + matches.size() + ": " + matches.keySet());
        }

        return matches.values().iterator().next();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> Map<String, T> getBeansOfType(Class<T> type) {
        Map<String, T> result = new LinkedHashMap<>();
        for (String name : beanNames) {
            Object bean = singletonMap.get(name);
            if (type.isInstance(bean)) {
                result.put(name, (T) bean);
            }
        }
        return result;
    }

    @Override
    public List<String> getBeanNames() {
        return List.copyOf(beanNames);
    }

    @Override
    public boolean containsBean(String name) {
        return singletonMap.containsKey(name);
    }

    /**
     * Derive a bean name from a class: simple name with first letter lowercased.
     * e.g., UserController → "userController", SSEEmitter → "sSEEmitter"
     *
     * Real Spring equivalent: AnnotationBeanNameGenerator.buildDefaultBeanName()
     * which calls Introspector.decapitalize().
     */
    private String deriveBeanName(Class<?> clazz) {
        String simpleName = clazz.getSimpleName();
        if (simpleName.isEmpty()) {
            return simpleName;
        }
        return Character.toLowerCase(simpleName.charAt(0)) + simpleName.substring(1);
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/container/SimpleBeanContainerTest.java` [NEW]

```java
package com.simplespringmvc.container;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for SimpleBeanContainer — verifies all core registry operations.
 */
class SimpleBeanContainerTest {

    private SimpleBeanContainer container;

    @BeforeEach
    void setUp() {
        container = new SimpleBeanContainer();
    }

    // ── Registration and retrieval by name ──────────────────────────────

    @Test
    void shouldRetrieveBean_WhenRegisteredByName() {
        var service = new OrderService();
        container.registerBean("orderService", service);

        Object result = container.getBean("orderService");

        assertThat(result).isSameAs(service);
    }

    @Test
    void shouldReturnSameInstance_WhenRetrievedMultipleTimes() {
        var service = new OrderService();
        container.registerBean("orderService", service);

        Object first = container.getBean("orderService");
        Object second = container.getBean("orderService");

        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldDeriveNameFromClass_WhenRegisteredWithoutName() {
        var service = new OrderService();
        container.registerBean(service);

        Object result = container.getBean("orderService");

        assertThat(result).isSameAs(service);
    }

    @Test
    void shouldThrowBeanNotFoundException_WhenBeanNotRegistered() {
        assertThatThrownBy(() -> container.getBean("missing"))
                .isInstanceOf(BeanNotFoundException.class)
                .hasMessageContaining("missing");
    }

    // ── Registration validation ─────────────────────────────────────────

    @Test
    void shouldThrowException_WhenNameIsNull() {
        assertThatThrownBy(() -> container.registerBean(null, new OrderService()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldThrowException_WhenNameIsBlank() {
        assertThatThrownBy(() -> container.registerBean("  ", new OrderService()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldThrowException_WhenBeanInstanceIsNull() {
        assertThatThrownBy(() -> container.registerBean("service", null))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldReplaceBean_WhenRegisteredWithSameName() {
        var first = new OrderService();
        var second = new OrderService();
        container.registerBean("service", first);
        container.registerBean("service", second);

        assertThat(container.getBean("service")).isSameAs(second);
    }

    // ── Type-based retrieval ────────────────────────────────────────────

    @Test
    void shouldRetrieveBean_WhenLookingUpByConcreteType() {
        var service = new OrderService();
        container.registerBean("orderService", service);

        OrderService result = container.getBean(OrderService.class);

        assertThat(result).isSameAs(service);
    }

    @Test
    void shouldRetrieveBean_WhenLookingUpByInterface() {
        var service = new OrderService();
        container.registerBean("orderService", service);

        Runnable result = container.getBean(Runnable.class);

        assertThat(result).isSameAs(service);
    }

    @Test
    void shouldRetrieveBean_WhenLookingUpBySuperclass() {
        var service = new SpecialOrderService();
        container.registerBean("special", service);

        OrderService result = container.getBean(OrderService.class);

        assertThat(result).isSameAs(service);
    }

    @Test
    void shouldThrowBeanNotFoundException_WhenNoTypeMatch() {
        container.registerBean("orderService", new OrderService());

        assertThatThrownBy(() -> container.getBean(String.class))
                .isInstanceOf(BeanNotFoundException.class)
                .hasMessageContaining("String");
    }

    @Test
    void shouldThrowNoUniqueBeanException_WhenMultipleBeansMatchType() {
        container.registerBean("service1", new OrderService());
        container.registerBean("service2", new OrderService());

        assertThatThrownBy(() -> container.getBean(OrderService.class))
                .isInstanceOf(NoUniqueBeanException.class)
                .hasMessageContaining("2");
    }

    // ── getBeansOfType ──────────────────────────────────────────────────

    @Test
    void shouldReturnAllMatchingBeans_WhenCallingGetBeansOfType() {
        var service1 = new OrderService();
        var service2 = new OrderService();
        container.registerBean("s1", service1);
        container.registerBean("s2", service2);
        container.registerBean("other", "a string");

        Map<String, Runnable> result = container.getBeansOfType(Runnable.class);

        assertThat(result).hasSize(2);
        assertThat(result.get("s1")).isSameAs(service1);
        assertThat(result.get("s2")).isSameAs(service2);
    }

    @Test
    void shouldReturnEmptyMap_WhenNoBeansMatchType() {
        container.registerBean("s", new OrderService());

        Map<String, String> result = container.getBeansOfType(String.class);

        assertThat(result).isEmpty();
    }

    @Test
    void shouldPreserveRegistrationOrder_WhenCallingGetBeansOfType() {
        container.registerBean("c", new OrderService());
        container.registerBean("a", new OrderService());
        container.registerBean("b", new OrderService());

        Map<String, OrderService> result = container.getBeansOfType(OrderService.class);

        assertThat(result.keySet()).containsExactly("c", "a", "b");
    }

    // ── containsBean ────────────────────────────────────────────────────

    @Test
    void shouldReturnTrue_WhenBeanExists() {
        container.registerBean("service", new OrderService());

        assertThat(container.containsBean("service")).isTrue();
    }

    @Test
    void shouldReturnFalse_WhenBeanDoesNotExist() {
        assertThat(container.containsBean("missing")).isFalse();
    }

    // ── getBeanNames ────────────────────────────────────────────────────

    @Test
    void shouldReturnAllBeanNames_InRegistrationOrder() {
        container.registerBean("alpha", new OrderService());
        container.registerBean("beta", new OrderService());
        container.registerBean("gamma", new OrderService());

        List<String> names = container.getBeanNames();

        assertThat(names).containsExactly("alpha", "beta", "gamma");
    }

    @Test
    void shouldReturnImmutableList_FromGetBeanNames() {
        container.registerBean("a", new OrderService());
        List<String> names = container.getBeanNames();

        assertThatThrownBy(() -> names.add("hacked"))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void shouldNotDuplicateName_WhenReRegisteredWithSameName() {
        container.registerBean("svc", new OrderService());
        container.registerBean("svc", new OrderService());

        assertThat(container.getBeanNames()).containsExactly("svc");
    }

    // ── Test fixtures ───────────────────────────────────────────────────

    static class OrderService implements Runnable {
        @Override
        public void run() { }
    }

    static class SpecialOrderService extends OrderService { }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **IoC Container** | A central registry that holds application objects (beans), decoupling creation from usage |
| **Singleton semantics** | Every call to `getBean()` returns the same instance — the container owns the object's lifecycle |
| **Type-based lookup** | Retrieve beans by interface or superclass, not just by name — enables programming to abstractions |
| **Registration order** | A parallel `List<String>` ensures `getBeansOfType()` returns beans in a deterministic, predictable order |

**Next: Chapter 2 — Embedded HTTP Server & DispatcherServlet** — Embed Tomcat to receive HTTP requests and route them through a front-controller Servlet that retrieves its strategy beans from the `SimpleBeanContainer` we just built.
