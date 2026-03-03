# Chapter 7: Decorator — Bean Enhancement Chain

## What You'll Learn
- How the **Decorator** pattern adds behavior to beans without modifying their source code
- How `BeanPostProcessor` enables the Spring container's extensibility
- How `BeanFactoryPostProcessor` modifies bean definitions before creation
- The difference between modifying the **blueprint** vs the **product**

---

## 7.1 The Problem: Adding Features to Beans After Creation

In Chapter 2, our container creates beans but has no way to enhance them. What if we want to:
- Automatically wrap certain beans in AOP proxies (from Chapter 5)
- Inject `@Autowired` fields after construction
- Validate beans against constraints
- Log every bean creation

We can't modify every bean class. We need a **general-purpose enhancement hook**.

## 7.2 Decorator Pattern: Wrapping Without Subclassing

Classic Decorator adds behavior by wrapping:

```
  ┌───────────────────────────────────────────────────┐
  │               Bean Creation Pipeline               │
  │                                                   │
  │  Bean Instance                                    │
  │      │                                            │
  │      ▼                                            │
  │  ┌──────────────────────┐                        │
  │  │ BPP #1: AutowireBPP  │  ← inject @Autowired   │
  │  │ postProcessBefore()  │                        │
  │  └──────────┬───────────┘                        │
  │             │                                    │
  │      ▼                                            │
  │  [InitializingBean.afterPropertiesSet()]          │
  │  [Custom init-method]                            │
  │             │                                    │
  │      ▼                                            │
  │  ┌──────────────────────┐                        │
  │  │ BPP #2: ProxyBPP     │  ← wrap in proxy       │
  │  │ postProcessAfter()   │                        │
  │  └──────────┬───────────┘                        │
  │             │                                    │
  │      ▼                                            │
  │  Enhanced Bean (may be a proxy!)                  │
  └───────────────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - Spring's `BeanPostProcessor` is a **pipeline-style Decorator**: each processor receives the bean, optionally transforms it, and passes it to the next. This differs from classic Decorator (which wraps a single interface) — it's more like a **filter chain**.
> - The distinction between `postProcessBeforeInitialization` and `postProcessAfterInitialization` is critical: `@Autowired` injection happens BEFORE init (so `afterPropertiesSet()` can use injected fields), while AOP proxying happens AFTER init (so the proxy wraps the fully-initialized bean).
> - **When to use this pattern:** When you need to apply cross-cutting enhancements to many objects of different types. If you're enhancing ONE specific type, simple inheritance or composition is clearer.
> ─────────────────────────────────────────────────────

## 7.3 Building SimpleBeanPostProcessor

### Step 1: The Post-Processor Interface

```java
// === SimpleBeanPostProcessor.java ===
/**
 * Hook for modifying or wrapping beans during creation.
 * Called for EVERY bean the container creates.
 */
public interface SimpleBeanPostProcessor {

    /**
     * Called BEFORE initialization callbacks (InitializingBean, init-method).
     * Use for: validation, field injection, early modifications.
     *
     * @param bean     the raw bean instance
     * @param beanName the name registered in the container
     * @return the (possibly modified) bean — may be the same or a wrapper
     */
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;  // default: pass through unchanged
    }

    /**
     * Called AFTER initialization callbacks.
     * Use for: AOP proxy wrapping, final validation.
     *
     * @return the (possibly proxied) bean
     */
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;  // default: pass through unchanged
    }
}
```

### Step 2: The BeanFactory Post-Processor (Definition-Level Enhancement)

```java
// === SimpleBeanFactoryPostProcessor.java ===
/**
 * Modifies bean definitions BEFORE any beans are created.
 * This works on the BLUEPRINT, not the PRODUCT.
 *
 * Use cases: property placeholder resolution, bean definition modification.
 */
@FunctionalInterface
public interface SimpleBeanFactoryPostProcessor {
    void postProcessBeanFactory(SimpleContainer container);
}
```

### Step 3: Update Container to Support Post-Processors

```java
// === SimpleContainer.java (enhanced) ===
import java.util.Map;
import java.util.List;
import java.util.ArrayList;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

public class SimpleContainer implements BeanFactory {

    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
    private final Map<String, Supplier<?>> singletonFactories = new ConcurrentHashMap<>(16);
    private final Map<String, Supplier<?>> beanDefinitions = new ConcurrentHashMap<>();
    private final java.util.Set<String> currentlyCreating = ConcurrentHashMap.newKeySet();

    // ── NEW: Post-processor chains ──
    private final List<SimpleBeanPostProcessor> beanPostProcessors = new ArrayList<>();
    private final List<SimpleBeanFactoryPostProcessor> factoryPostProcessors = new ArrayList<>();

    public void addBeanPostProcessor(SimpleBeanPostProcessor bpp) {
        this.beanPostProcessors.add(bpp);
    }

    public void addBeanFactoryPostProcessor(SimpleBeanFactoryPostProcessor bfpp) {
        this.factoryPostProcessors.add(bfpp);
    }

    /**
     * Called during container startup — applies factory post-processors
     * to modify bean definitions before any beans are created.
     */
    public void refresh() {
        // Phase 1: Let BeanFactoryPostProcessors modify definitions
        for (SimpleBeanFactoryPostProcessor bfpp : factoryPostProcessors) {
            bfpp.postProcessBeanFactory(this);
        }

        // Phase 2: Pre-instantiate singletons
        for (String name : new ArrayList<>(beanDefinitions.keySet())) {
            getBean(name);
        }
    }

    public void registerBean(String name, Supplier<?> factory) {
        this.beanDefinitions.put(name, factory);
    }

    public void registerSingleton(String name, Object instance) {
        this.singletonObjects.put(name, instance);
    }

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
        Object singleton = getSingleton(name, true);
        if (singleton != null) {
            return singleton;
        }

        Supplier<?> definition = this.beanDefinitions.get(name);
        if (definition == null) {
            throw new RuntimeException("No bean named '" + name + "' is defined");
        }

        return createSingleton(name, definition);
    }

    private Object createSingleton(String name, Supplier<?> factory) {
        if (!this.currentlyCreating.add(name)) {
            throw new RuntimeException("Circular reference for bean '" + name + "'");
        }
        try {
            this.singletonFactories.put(name, factory);

            // Step 1: Create raw bean instance
            Object bean = factory.get();

            // ── NEW: Apply BeanPostProcessor chain ──

            // Step 2: Post-process BEFORE initialization
            for (SimpleBeanPostProcessor bpp : beanPostProcessors) {
                Object processed = bpp.postProcessBeforeInitialization(bean, name);
                if (processed != null) {
                    bean = processed;
                }
            }

            // Step 3: Initialization callback
            if (bean instanceof SimpleInitializingBean initable) {
                initable.afterPropertiesSet();
            }

            // Step 4: Post-process AFTER initialization
            for (SimpleBeanPostProcessor bpp : beanPostProcessors) {
                Object processed = bpp.postProcessAfterInitialization(bean, name);
                if (processed != null) {
                    bean = processed;
                }
            }

            // Step 5: Cache in Level 1
            this.singletonObjects.put(name, bean);
            this.earlySingletonObjects.remove(name);
            this.singletonFactories.remove(name);

            return bean;
        } finally {
            this.currentlyCreating.remove(name);
        }
    }

    private Object getSingleton(String name, boolean allowEarlyReference) {
        Object instance = this.singletonObjects.get(name);
        if (instance != null) return instance;

        if (this.currentlyCreating.contains(name)) {
            instance = this.earlySingletonObjects.get(name);
            if (instance != null) return instance;

            if (allowEarlyReference) {
                Supplier<?> factory = this.singletonFactories.get(name);
                if (factory != null) {
                    instance = factory.get();
                    this.earlySingletonObjects.put(name, instance);
                    this.singletonFactories.remove(name);
                    return instance;
                }
            }
        }
        return null;
    }

    @Override
    public boolean containsBean(String name) {
        return this.singletonObjects.containsKey(name)
                || this.beanDefinitions.containsKey(name);
    }

    // Expose definitions for BeanFactoryPostProcessors
    public Map<String, Supplier<?>> getBeanDefinitions() {
        return beanDefinitions;
    }
}
```

```java
// === SimpleInitializingBean.java ===
/**
 * Lifecycle callback: called after all properties are set.
 * Equivalent to Spring's InitializingBean.
 */
public interface SimpleInitializingBean {
    void afterPropertiesSet();
}
```

> ★ **Insight** ─────────────────────────────────────
> - The key design choice: `postProcessAfterInitialization()` can return a **completely different object** (e.g., a proxy). This means the object stored in `singletonObjects` might NOT be the same class that was instantiated. This is how Spring's `@Transactional` works — the BPP replaces your service with a proxy.
> - Returning `null` from a BPP short-circuits the chain and keeps the current bean. Real Spring treats `null` differently — it logs a warning (since 7.0) and keeps the original bean. Returning the bean unchanged (the default) is the idiomatic pass-through.
> - **BeanFactoryPostProcessor vs BeanPostProcessor:** BeanFactoryPostProcessor modifies the RECIPE (definition); BeanPostProcessor modifies the DISH (instance). Running them in the wrong order breaks things — definitions must be modified BEFORE beans are created.
> ─────────────────────────────────────────────────────

## 7.4 Concrete Post-Processor Implementations

```java
// === LoggingBeanPostProcessor.java ===
/**
 * Logs every bean creation. Useful for debugging.
 */
public class LoggingBeanPostProcessor implements SimpleBeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("[BPP] Created bean: '" + beanName + "' → " + bean.getClass().getSimpleName());
        return bean;
    }
}
```

```java
// === AutoProxyBeanPostProcessor.java ===
/**
 * Automatically wraps beans implementing certain interfaces with AOP proxies.
 * This is a simplified version of how @Transactional works.
 */
public class AutoProxyBeanPostProcessor implements SimpleBeanPostProcessor {

    private final List<MethodInterceptor> interceptors;

    public AutoProxyBeanPostProcessor(MethodInterceptor... interceptors) {
        this.interceptors = List.of(interceptors);
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Only proxy beans that implement at least one interface
        if (bean.getClass().getInterfaces().length == 0) {
            return bean;
        }

        // Create proxy wrapping the bean
        SimpleAopProxy aopProxy = new SimpleAopProxy(bean);
        for (MethodInterceptor interceptor : interceptors) {
            aopProxy.addInterceptor(interceptor);
        }
        return aopProxy.getProxy();
    }
}
```

## 7.5 Usage

```java
// === DecoratorDemo.java ===
public class DecoratorDemo {
    public static void main(String[] args) {
        SimpleContainer container = new SimpleContainer();

        // Register post-processors (decorators)
        container.addBeanPostProcessor(new LoggingBeanPostProcessor());
        container.addBeanPostProcessor(new AutoProxyBeanPostProcessor(
            new TimingInterceptor()
        ));

        // Register bean definitions
        container.registerBean("dataSource", () -> new SimpleDataSource("jdbc:h2:mem:test"));
        container.registerBean("userService", () -> {
            SimpleDataSource ds = container.getBean("dataSource", SimpleDataSource.class);
            return new UserServiceImpl();
        });

        // Trigger creation
        container.refresh();

        // The retrieved bean is now a PROXY, not the original UserServiceImpl!
        UserService service = container.getBean("userService", UserService.class);
        service.findUser(42);
        // Output includes timing from the auto-proxy!
    }
}
```

**Output:**
```
[BPP] Created bean: 'dataSource' → SimpleDataSource
[BPP] Created bean: 'userService' → $Proxy0
[TIMING] findUser took 0ms
```

Notice `userService` is now `$Proxy0` — the post-processor replaced it with a proxy!

## 7.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleBeanPostProcessor` | `BeanPostProcessor` | `:74-101` |
| `postProcessBeforeInitialization()` | `BeanPostProcessor.postProcessBeforeInitialization()` | `:74` |
| `postProcessAfterInitialization()` | `BeanPostProcessor.postProcessAfterInitialization()` | `:99` |
| `SimpleBeanFactoryPostProcessor` | `BeanFactoryPostProcessor` | `:73` |
| `SimpleInitializingBean` | `InitializingBean` | `spring-beans` |
| `AutoProxyBeanPostProcessor` | `AbstractAutoProxyCreator` | `spring-aop` |
| `LoggingBeanPostProcessor` | (concept — no exact equivalent) | Custom BPP |
| `container.refresh()` | `AbstractApplicationContext.refresh()` | `spring-context` |

**What real Spring adds:**
- `InstantiationAwareBeanPostProcessor` — hooks before/after instantiation (not just initialization)
- `DestructionAwareBeanPostProcessor` — hooks for bean destruction
- `SmartInstantiationAwareBeanPostProcessor` — predict bean type, determine constructor
- `MergedBeanDefinitionPostProcessor` — post-process merged bean definitions
- `Ordered` / `PriorityOrdered` — control BPP execution order
- `@PostConstruct` / `@PreDestroy` support via `InitDestroyAnnotationBeanPostProcessor`

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Decorator (Pipeline)** | Chain of processors that enhance each bean during creation |
| **BeanPostProcessor** | Modifies bean INSTANCES before/after initialization |
| **BeanFactoryPostProcessor** | Modifies bean DEFINITIONS before beans are created |
| **InitializingBean** | Lifecycle callback after all properties are set |
| **Auto-Proxying** | BPP that replaces beans with AOP proxies automatically |

**Next: Chapter 8** — We build `SimpleDispatcher` using the **Adapter** and **Chain of Responsibility** patterns for request processing.
