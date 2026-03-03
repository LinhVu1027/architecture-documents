# Chapter 2: Factory & Singleton — The IoC Container Foundation

## What You'll Learn
- How the **Factory Method** pattern decouples object creation from usage
- How the **Singleton** pattern ensures shared instances with thread safety
- How Spring's three-level cache resolves circular dependencies
- Why `FactoryBean` exists alongside `BeanFactory`

---

## 2.1 The Problem: Hardcoded Object Creation

From Chapter 1, every class creates its own dependencies:

```java
// Every service does this — tight coupling!
public class UserService {
    private final UserRepository repo = new JdbcUserRepository();  // hardcoded!
    private final EmailService email = new SmtpEmailService();      // hardcoded!
}
```

What if you want `MockUserRepository` in tests? You can't — the `new` is baked in.

## 2.2 Factory Method: Centralized Creation

The Factory Method pattern moves `new` into a single place:

```java
// === BeanFactory.java ===
// The central factory interface — clients never call "new"
public interface BeanFactory {
    Object getBean(String name);
    <T> T getBean(String name, Class<T> type);
    boolean containsBean(String name);
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Factory Method** is the most fundamental pattern in Spring. It inverts the dependency direction: instead of classes creating their dependencies, a central factory creates everything. This is the "Inversion of Control" in IoC.
> - The alternative — Service Locator — also centralizes creation but requires every class to know about the locator. Factory Method hides even the factory behind an interface.
> - **When NOT to use:** For simple utility objects (like `new ArrayList<>()`), a factory adds unnecessary indirection. Reserve factories for objects with complex lifecycles or external dependencies.
> ─────────────────────────────────────────────────────

## 2.3 Building SimpleContainer — Step by Step

### Step 1: The Registry

```java
// === SimpleContainer.java ===
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

public class SimpleContainer implements BeanFactory {

    // ── SINGLETON PATTERN: Three-level cache ──
    // Level 1: Fully initialized singletons
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // Level 2: Early references (for circular dependency resolution)
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

    // Level 3: Factory lambdas that can create early references
    private final Map<String, Supplier<?>> singletonFactories = new ConcurrentHashMap<>(16);

    // Bean definitions: name → how to create the bean
    private final Map<String, Supplier<?>> beanDefinitions = new ConcurrentHashMap<>();

    // Track which beans are currently being created (circular ref detection)
    private final java.util.Set<String> currentlyCreating = ConcurrentHashMap.newKeySet();
```

### Step 2: Registration

```java
    // Register a bean definition (a recipe for creating the bean)
    public void registerBean(String name, Supplier<?> factory) {
        this.beanDefinitions.put(name, factory);
    }

    // Register a pre-built singleton directly
    public void registerSingleton(String name, Object instance) {
        this.singletonObjects.put(name, instance);
    }
```

### Step 3: The Three-Level Lookup (getSingleton)

```java
    /**
     * Look up a singleton through all three cache levels.
     * This is how Spring resolves circular references.
     */
    private Object getSingleton(String name, boolean allowEarlyReference) {
        // Level 1: Check fully initialized singletons
        Object instance = this.singletonObjects.get(name);
        if (instance != null) {
            return instance;
        }

        // Only check deeper caches if bean is currently being created
        // (this means we're in a circular reference)
        if (this.currentlyCreating.contains(name)) {
            // Level 2: Check early singleton objects
            instance = this.earlySingletonObjects.get(name);
            if (instance != null) {
                return instance;
            }

            // Level 3: Check singleton factories
            if (allowEarlyReference) {
                Supplier<?> factory = this.singletonFactories.get(name);
                if (factory != null) {
                    instance = factory.get();
                    // Promote from Level 3 → Level 2
                    this.earlySingletonObjects.put(name, instance);
                    this.singletonFactories.remove(name);
                    return instance;
                }
            }
        }
        return null;
    }
```

> ★ **Insight** ─────────────────────────────────────
> - The **three-level cache** is Spring's elegant solution to circular dependencies (`A` depends on `B`, `B` depends on `A`). Without it, you'd get infinite recursion.
> - **Level 3 → Level 2 promotion** is key: the factory creates an early, partially-initialized reference that other beans can use. Once the bean finishes initialization, it's promoted to Level 1.
> - **Real Spring** uses `ObjectFactory<?>` instead of `Supplier<?>` and adds `ReentrantLock` for thread safety (see `DefaultSingletonBeanRegistry.java:83`). We simplify by using `ConcurrentHashMap` which provides adequate safety for our demonstration.
> ─────────────────────────────────────────────────────

### Step 4: The Core getBean() — Factory Method

```java
    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(String name, Class<T> type) {
        Object bean = getBean(name);
        if (type != null && !type.isInstance(bean)) {
            throw new RuntimeException("Bean '" + name + "' is not of type " + type.getName());
        }
        return (T) bean;
    }

    @Override
    public Object getBean(String name) {
        // 1. Check singleton cache (three-level lookup)
        Object singleton = getSingleton(name, true);
        if (singleton != null) {
            return singleton;
        }

        // 2. Look up the bean definition (factory recipe)
        Supplier<?> definition = this.beanDefinitions.get(name);
        if (definition == null) {
            throw new RuntimeException("No bean named '" + name + "' is defined");
        }

        // 3. Create the singleton
        return createSingleton(name, definition);
    }

    private Object createSingleton(String name, Supplier<?> factory) {
        // Mark as "currently creating" for circular reference detection
        if (!this.currentlyCreating.add(name)) {
            throw new RuntimeException("Circular reference detected for bean '" + name + "'");
        }

        try {
            // Register a Level 3 factory for early reference
            this.singletonFactories.put(name, factory);

            // Actually create the bean
            Object instance = factory.get();

            // Promote to Level 1 (fully initialized)
            this.singletonObjects.put(name, instance);
            this.earlySingletonObjects.remove(name);
            this.singletonFactories.remove(name);

            return instance;
        } finally {
            this.currentlyCreating.remove(name);
        }
    }

    @Override
    public boolean containsBean(String name) {
        return this.singletonObjects.containsKey(name)
                || this.beanDefinitions.containsKey(name);
    }
}
```

### Step 5: FactoryBean — A Bean That Creates Other Beans

```java
// === SimpleFactoryBean.java ===
/**
 * A special bean that acts as a factory for other objects.
 * Spring uses the "&" prefix convention:
 *   getBean("myFactory")  → returns the PRODUCT
 *   getBean("&myFactory") → returns the FACTORY itself
 */
public interface SimpleFactoryBean<T> {
    T getObject();
    Class<?> getObjectType();
    default boolean isSingleton() { return true; }
}
```

> ★ **Insight** ─────────────────────────────────────
> - `BeanFactory` vs `FactoryBean` is one of the most confusing naming decisions in Spring. **BeanFactory** is the container that manages all beans. **FactoryBean** is a bean that produces OTHER beans.
> - The `&` prefix convention (from `BeanFactory.FACTORY_BEAN_PREFIX`) lets you access the factory itself: `getBean("&dataSource")` returns the `DataSourceFactoryBean`, while `getBean("dataSource")` returns the `DataSource` it produced.
> - **When to use FactoryBean:** When object creation is complex (e.g., creating a `SessionFactory` requires configuring a mapping, dialect, pool). The factory encapsulates all that complexity.
> ─────────────────────────────────────────────────────

## 2.4 Using Our Container

```java
// === Main.java ===
public class Main {
    public static void main(String[] args) {
        SimpleContainer container = new SimpleContainer();

        // Register bean definitions (recipes)
        container.registerBean("dataSource", () -> {
            System.out.println("Creating DataSource...");
            return new SimpleDataSource("jdbc:h2:mem:test");
        });

        container.registerBean("userRepo", () -> {
            // Factory can reference other beans! (Dependency Injection)
            SimpleDataSource ds = container.getBean("dataSource", SimpleDataSource.class);
            return new SimpleUserRepository(ds);
        });

        // First call: creates the singleton
        Object repo1 = container.getBean("userRepo");
        // Second call: returns cached singleton
        Object repo2 = container.getBean("userRepo");

        System.out.println("Same instance? " + (repo1 == repo2));  // true!
    }
}

// === SimpleDataSource.java ===
class SimpleDataSource {
    private final String url;
    SimpleDataSource(String url) { this.url = url; }
    String getUrl() { return url; }
}

// === SimpleUserRepository.java ===
class SimpleUserRepository {
    private final SimpleDataSource dataSource;
    SimpleUserRepository(SimpleDataSource ds) { this.dataSource = ds; }
    SimpleDataSource getDataSource() { return dataSource; }
}
```

**Output:**
```
Creating DataSource...
Same instance? true
```

Note: "Creating DataSource..." only prints once — the singleton is cached.

## 2.5 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleContainer.getBean()` | `AbstractBeanFactory.doGetBean()` | `:236` |
| `SimpleContainer.singletonObjects` | `DefaultSingletonBeanRegistry.singletonObjects` | `:86` |
| `SimpleContainer.earlySingletonObjects` | `DefaultSingletonBeanRegistry.earlySingletonObjects` | `:95` |
| `SimpleContainer.singletonFactories` | `DefaultSingletonBeanRegistry.singletonFactories` | `:89` |
| `SimpleContainer.getSingleton()` | `DefaultSingletonBeanRegistry.getSingleton(String, boolean)` | `:208` |
| `SimpleContainer.createSingleton()` | `DefaultSingletonBeanRegistry.getSingleton(String, ObjectFactory)` | `:255` |
| `SimpleFactoryBean<T>` | `FactoryBean<T>` | `:75` |
| `BeanFactory` interface | `BeanFactory` | `:131` |

**What real Spring adds:**
- `ReentrantLock` for thread-safe singleton creation (`:83`)
- `ObjectFactory<?>` with checked exceptions instead of `Supplier<?>`
- Scope support (prototype, request, session) via `Scope` interface
- Parent bean factory delegation for hierarchical contexts
- Bean type resolution via `ResolvableType`

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Factory Method** | Centralize object creation in one place; clients call `getBean()` instead of `new` |
| **Singleton Pattern** | One shared instance per bean name; cached in `singletonObjects` map |
| **Three-Level Cache** | `singletonObjects` → `earlySingletonObjects` → `singletonFactories`; resolves circular refs |
| **FactoryBean** | A special bean that creates other beans; accessed via `&` prefix |
| **BeanFactory** | The root interface of Spring's IoC container |

**Next: Chapter 3** — We build `SimpleJdbcTemplate` using the **Template Method** pattern to eliminate JDBC boilerplate.
