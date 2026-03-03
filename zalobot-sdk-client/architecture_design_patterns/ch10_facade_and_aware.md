# Chapter 10: Facade & Aware — Unified API and Framework Injection

## What You'll Learn
- How the **Facade** pattern provides a unified API over complex subsystems
- How the **Aware** pattern enables controlled injection of framework objects
- How `ApplicationContext` combines Bean Factory, Events, Resources, and Environment
- Why Aware interfaces exist alongside `@Autowired`

---

## 10.1 The Problem: Too Many Entry Points

After building chapters 2–9, our mini-framework has separate components:

```java
// A developer must juggle all of these!
SimpleContainer container = new SimpleContainer();
SimpleEventMulticaster events = new SimpleEventMulticaster();
SimpleConversionService conversion = new SimpleConversionService();
SimpleResourceLoader resources = new SimpleResourceLoader();
SimpleEnvironment environment = new SimpleEnvironment();
```

Each has its own API, its own lifecycle. Users must wire them together manually. This is the **subsystem complexity** problem.

## 10.2 Facade: One Interface to Rule Them All

```
     ┌────────────────────────────────────────────────┐
     │            ApplicationContext (Facade)          │
     │                                                │
     │  ┌──────────┐  ┌────────────┐  ┌────────────┐ │
     │  │BeanFactory│  │EventPublisher│ │ResourceLoader│
     │  └──────────┘  └────────────┘  └────────────┘ │
     │  ┌──────────┐  ┌────────────┐                 │
     │  │Environment│  │Conversion  │                 │
     │  │           │  │Service     │                 │
     │  └──────────┘  └────────────┘                 │
     │                                                │
     │  Client uses ONE object for everything         │
     └────────────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - Facade vs Mediator: **Facade** provides a simplified interface; the subsystems don't know about the facade. **Mediator** coordinates bidirectional communication between subsystems. `ApplicationContext` is a Facade — `JdbcTemplate` doesn't know about `ApplicationContext`.
> - Spring's `ApplicationContext` extends FIVE interfaces simultaneously: `ListableBeanFactory`, `HierarchicalBeanFactory`, `MessageSource`, `ApplicationEventPublisher`, `ResourcePatternResolver`. This is **interface composition** — the facade inherits all APIs through multiple inheritance of interfaces.
> - **When to use Facade:** When a subsystem has 5+ classes that clients must coordinate. The facade doesn't add new functionality — it just simplifies access to existing functionality.
> ─────────────────────────────────────────────────────

## 10.3 Building SimpleApplicationContext

### Step 1: Resource Loading (one of the subsystem capabilities)

```java
// === SimpleResource.java ===
/**
 * Abstraction for a resource (file, classpath, URL).
 */
public class SimpleResource {
    private final String location;
    private final String content;

    public SimpleResource(String location, String content) {
        this.location = location;
        this.content = content;
    }

    public String getLocation() { return location; }
    public String getContent() { return content; }
    public boolean exists() { return content != null; }
}
```

```java
// === SimpleResourceLoader.java ===
import java.util.HashMap;
import java.util.Map;

/**
 * Loads resources by location string.
 */
public class SimpleResourceLoader {

    private final Map<String, String> resources = new HashMap<>();

    public void registerResource(String location, String content) {
        resources.put(location, content);
    }

    public SimpleResource getResource(String location) {
        String content = resources.get(location);
        return new SimpleResource(location, content);
    }
}
```

### Step 2: Environment (another subsystem capability)

```java
// === SimpleEnvironment.java ===
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * Manages profiles and properties.
 */
public class SimpleEnvironment {

    private final Set<String> activeProfiles = new HashSet<>();
    private final Map<String, String> properties = new HashMap<>();

    public void setActiveProfile(String profile) {
        activeProfiles.add(profile);
    }

    public String[] getActiveProfiles() {
        return activeProfiles.toArray(new String[0]);
    }

    public boolean isProfileActive(String profile) {
        return activeProfiles.contains(profile);
    }

    public void setProperty(String key, String value) {
        properties.put(key, value);
    }

    public String getProperty(String key) {
        return properties.get(key);
    }

    public String getProperty(String key, String defaultValue) {
        return properties.getOrDefault(key, defaultValue);
    }
}
```

### Step 3: The Facade — SimpleApplicationContext

```java
// === SimpleApplicationContext.java ===
import java.util.function.Supplier;

/**
 * FACADE: Unified interface combining BeanFactory, EventPublisher,
 * ResourceLoader, Environment, and ConversionService.
 *
 * Equivalent to Spring's ApplicationContext which extends:
 * - ListableBeanFactory
 * - HierarchicalBeanFactory
 * - ApplicationEventPublisher
 * - ResourcePatternResolver
 * - MessageSource
 * - EnvironmentCapable
 */
public class SimpleApplicationContext implements BeanFactory, SimpleEventPublisher {

    private final String id;
    private final SimpleContainer beanFactory;
    private final SimpleEventMulticaster eventMulticaster;
    private final SimpleResourceLoader resourceLoader;
    private final SimpleEnvironment environment;
    private final SimpleConversionService conversionService;
    private boolean active = false;

    public SimpleApplicationContext() {
        this("SimpleAppContext-" + System.identityHashCode(this));
    }

    public SimpleApplicationContext(String id) {
        this.id = id;
        this.beanFactory = new SimpleContainer();
        this.eventMulticaster = new SimpleEventMulticaster();
        this.resourceLoader = new SimpleResourceLoader();
        this.environment = new SimpleEnvironment();
        this.conversionService = new SimpleConversionService();
    }

    // ══════════════════════════════════════════════════════════
    // FACADE methods — delegate to subsystems
    // ══════════════════════════════════════════════════════════

    // ── BeanFactory delegation ──
    @Override
    public Object getBean(String name) { return beanFactory.getBean(name); }

    @Override
    public <T> T getBean(String name, Class<T> type) { return beanFactory.getBean(name, type); }

    @Override
    public boolean containsBean(String name) { return beanFactory.containsBean(name); }

    public void registerBean(String name, Supplier<?> factory) {
        beanFactory.registerBean(name, factory);
    }

    public void addBeanPostProcessor(SimpleBeanPostProcessor bpp) {
        beanFactory.addBeanPostProcessor(bpp);
    }

    // ── EventPublisher delegation ──
    @Override
    public void publishEvent(SimpleApplicationEvent event) {
        eventMulticaster.publishEvent(event);
    }

    public void addListener(SimpleApplicationListener<?> listener) {
        eventMulticaster.addListener(listener);
    }

    // ── ResourceLoader delegation ──
    public SimpleResource getResource(String location) {
        return resourceLoader.getResource(location);
    }

    public void registerResource(String location, String content) {
        resourceLoader.registerResource(location, content);
    }

    // ── Environment delegation ──
    public SimpleEnvironment getEnvironment() { return environment; }

    // ── ConversionService delegation ──
    public SimpleConversionService getConversionService() { return conversionService; }

    // ── Context lifecycle ──
    public String getId() { return id; }

    /**
     * Initialize the context: apply BeanFactoryPostProcessors,
     * register aware-processing BPP, and pre-instantiate singletons.
     */
    public void refresh() {
        // Register the Aware-injecting BeanPostProcessor
        beanFactory.addBeanPostProcessor(new AwareBeanPostProcessor(this));

        // Register context infrastructure as beans
        beanFactory.registerSingleton("environment", environment);
        beanFactory.registerSingleton("conversionService", conversionService);

        // Pre-instantiate all singleton beans
        beanFactory.refresh();

        this.active = true;

        // Publish context refreshed event
        publishEvent(new ContextRefreshedEvent(this));
    }

    public boolean isActive() { return active; }
}
```

```java
// === ContextRefreshedEvent.java ===
public class ContextRefreshedEvent extends SimpleApplicationEvent {
    public ContextRefreshedEvent(Object source) {
        super(source);
    }
}
```

## 10.4 The Aware Pattern: Controlled Framework Injection

### Why Aware Exists

```java
// Without Aware: beans can't access the context
public class MyService {
    // How does this bean get a reference to ApplicationContext?
    // @Autowired works, but what if we want explicit, typed injection?
}
```

Aware interfaces provide explicit, typed callbacks for framework object injection:

```java
// === SimpleAware interfaces ===

public interface BeanNameAware {
    void setBeanName(String name);
}

public interface BeanFactoryAware {
    void setBeanFactory(BeanFactory beanFactory);
}

public interface ApplicationContextAware {
    void setApplicationContext(SimpleApplicationContext context);
}

public interface EnvironmentAware {
    void setEnvironment(SimpleEnvironment environment);
}

public interface EventPublisherAware {
    void setEventPublisher(SimpleEventPublisher publisher);
}

public interface ResourceLoaderAware {
    void setResourceLoader(SimpleResourceLoader loader);
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Aware vs @Autowired:** Both inject framework objects, but Aware is **explicit and typed** — it documents exactly what framework capability a bean needs. `@Autowired ApplicationContext ctx` is implicit and doesn't communicate intent. In library/framework code, Aware is preferred because it's self-documenting.
> - The Aware pattern is a **callback** pattern: the container detects the interface and calls the setter during bean initialization. The bean declares what it needs; the container fulfills it. This is **Interface Injection** — the third form of DI alongside Constructor and Setter injection.
> - **When to use Aware:** When writing framework-level code that needs specific container capabilities. In application code, prefer `@Autowired` or constructor injection — Aware creates a dependency on Spring interfaces.
> ─────────────────────────────────────────────────────

### The Aware-Processing BeanPostProcessor

```java
// === AwareBeanPostProcessor.java ===
/**
 * BeanPostProcessor that detects Aware interfaces and injects
 * the corresponding framework objects.
 *
 * This is how Spring processes Aware callbacks — through a BPP,
 * not hard-coded in the bean factory.
 */
public class AwareBeanPostProcessor implements SimpleBeanPostProcessor {

    private final SimpleApplicationContext applicationContext;

    public AwareBeanPostProcessor(SimpleApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Detect and invoke each Aware interface

        if (bean instanceof BeanNameAware aware) {
            aware.setBeanName(beanName);
        }

        if (bean instanceof BeanFactoryAware aware) {
            aware.setBeanFactory(applicationContext);
        }

        if (bean instanceof EnvironmentAware aware) {
            aware.setEnvironment(applicationContext.getEnvironment());
        }

        if (bean instanceof EventPublisherAware aware) {
            aware.setEventPublisher(applicationContext);
        }

        if (bean instanceof ApplicationContextAware aware) {
            aware.setApplicationContext(applicationContext);
        }

        return bean;
    }
}
```

## 10.5 Usage

```java
// === AwareService.java ===
/**
 * A bean that uses Aware interfaces to get framework capabilities.
 */
public class AwareService implements BeanNameAware, EnvironmentAware,
        EventPublisherAware, SimpleInitializingBean {

    private String beanName;
    private SimpleEnvironment environment;
    private SimpleEventPublisher eventPublisher;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("[AWARE] Bean name set: " + name);
    }

    @Override
    public void setEnvironment(SimpleEnvironment environment) {
        this.environment = environment;
        System.out.println("[AWARE] Environment injected");
    }

    @Override
    public void setEventPublisher(SimpleEventPublisher publisher) {
        this.eventPublisher = publisher;
        System.out.println("[AWARE] Event publisher injected");
    }

    @Override
    public void afterPropertiesSet() {
        // Now we can use the injected framework objects
        String profile = environment.getProperty("app.profile", "default");
        System.out.println("[INIT] " + beanName + " initialized with profile: " + profile);

        // Publish an event using the injected publisher
        eventPublisher.publishEvent(new ServiceReadyEvent(this, beanName));
    }

    public String getBeanName() { return beanName; }
}
```

```java
// === ServiceReadyEvent.java ===
public class ServiceReadyEvent extends SimpleApplicationEvent {
    private final String serviceName;

    public ServiceReadyEvent(Object source, String serviceName) {
        super(source);
        this.serviceName = serviceName;
    }

    public String getServiceName() { return serviceName; }
}
```

```java
// === FacadeAwareDemo.java ===
public class FacadeAwareDemo {
    public static void main(String[] args) {
        // Create the Facade — one object for everything
        SimpleApplicationContext context = new SimpleApplicationContext("MyApp");

        // Configure environment
        context.getEnvironment().setProperty("app.profile", "production");
        context.getEnvironment().setActiveProfile("prod");

        // Register resources
        context.registerResource("classpath:config.xml", "<beans>...</beans>");

        // Register conversion strategies
        context.getConversionService().addConverter(String.class, Integer.class, Integer::parseInt);

        // Register event listeners
        context.addListener(new SimpleApplicationListener<ContextRefreshedEvent>() {
            @Override
            public void onApplicationEvent(ContextRefreshedEvent event) {
                System.out.println("[EVENT] Context refreshed!");
            }
        });

        context.addListener(new SimpleApplicationListener<ServiceReadyEvent>() {
            @Override
            public void onApplicationEvent(ServiceReadyEvent event) {
                System.out.println("[EVENT] Service ready: " + event.getServiceName());
            }
        });

        // Register beans
        context.registerBean("awareService", AwareService::new);

        // Refresh — this triggers the entire lifecycle!
        System.out.println("=== Refreshing Context ===");
        context.refresh();

        System.out.println("\n=== Using Context ===");
        // Use the Facade for everything
        AwareService service = context.getBean("awareService", AwareService.class);
        System.out.println("Bean name: " + service.getBeanName());

        SimpleResource config = context.getResource("classpath:config.xml");
        System.out.println("Resource exists: " + config.exists());

        Integer num = context.getConversionService().convert("42", Integer.class);
        System.out.println("Converted: " + num);
    }
}
```

**Output:**
```
=== Refreshing Context ===
[AWARE] Bean name set: awareService
[AWARE] Environment injected
[AWARE] Event publisher injected
[INIT] awareService initialized with profile: production
[EVENT] Service ready: awareService
[EVENT] Context refreshed!

=== Using Context ===
Bean name: awareService
Resource exists: true
Converted: 42
```

## 10.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleApplicationContext` | `ApplicationContext` | `:59-60` (interface) |
| Context extends BeanFactory + EventPublisher + ... | `ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver` | `:59-60` |
| `refresh()` | `AbstractApplicationContext.refresh()` | 12-step boot in `spring-context` |
| `BeanNameAware` | `BeanNameAware` | `spring-beans` |
| `ApplicationContextAware` | `ApplicationContextAware` | `spring-context` |
| `EnvironmentAware` | `EnvironmentAware` | `spring-context` |
| `AwareBeanPostProcessor` | `ApplicationContextAwareProcessor` | `spring-context` |
| `SimpleEnvironment` | `StandardEnvironment` | `spring-core` |
| `SimpleResource` | `Resource` interface | `:66-224` |
| `SimpleResourceLoader` | `ResourceLoader` | `:46-80` |

**What real Spring adds:**
- `AbstractApplicationContext.refresh()` has 12 ordered steps (not just 3)
- `ConfigurableEnvironment` with property sources hierarchy (system, env, application)
- `ResourcePatternResolver` for wildcard patterns (`classpath*:com/example/**/*.xml`)
- `MessageSource` for i18n message resolution
- Parent context support for hierarchical beans
- `@Profile` annotation processing linked to Environment
- `ContextClosedEvent`, `ContextStartedEvent`, `ContextStoppedEvent` lifecycle events

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Facade** | One unified API over multiple subsystems |
| **Aware Pattern** | Callback interfaces for injecting framework objects into beans |
| **Interface Injection** | The container calls setter methods declared by marker interfaces |
| **ApplicationContext** | The supreme Facade — BeanFactory + Events + Resources + Environment |
| **refresh()** | Lifecycle method that initializes the entire context |

**Next: Chapter 11** — We build `BeanDefinitionVisitor` and callback objects using the **Visitor** and **Command** patterns.
