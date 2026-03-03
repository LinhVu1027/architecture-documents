# Chapter 6: Registry & Registrar — Managing Lifecycles and Collecting Endpoints

## What You'll Learn
- Why endpoint collection (registrar) and container management (registry) are separate concerns
- How `ZaloListenerEndpointRegistry` manages container lifecycle via `SmartLifecycle`
- How `ZaloListenerEndpointRegistrar` collects endpoints and defers container creation
- The two-phase registration pattern: collect first, materialize later

---

## 6.1 Two Components, One Job

The path from "BPP discovers annotation" to "container is running" involves TWO intermediaries:

```
BPP ───▶ Registrar ───▶ Registry ───▶ Container
 discovers    collects      creates &     runs
 annotations  endpoints     manages
```

**Why not combine them?** Because they operate at different lifecycle phases:

| Phase | Who | What happens |
|---|---|---|
| Bean post-processing | **Registrar** | Collects endpoints as BPP discovers them |
| After all singletons initialized | **Registrar** | Flushes all endpoints to Registry |
| Registry receives endpoint | **Registry** | Creates container via factory, stores it |
| Application context refresh | **Registry** | Starts all `autoStartup=true` containers |
| Application context close | **Registry** | Stops all containers |

If the BPP created containers directly during post-processing, beans referenced by one `@ZaloListener` might not exist yet when another is being processed. The registrar buffers everything until ALL beans are ready.

> ★ **Insight** ─────────────────────────────────────
> - **Two-phase commit in bean processing:** This is the same pattern used by Spring's `@EventListener` infrastructure (`EventListenerMethodProcessor` collects methods, `ApplicationEventMulticaster` manages them) and by `@Scheduled` (`ScheduledAnnotationBeanPostProcessor` collects, `TaskScheduler` manages). The reason is always the same: you can't safely create cross-bean infrastructure during post-processing because not all beans exist yet. Collection during post-processing + materialization after `afterSingletonsInstantiated()` is the standard Spring idiom.
> ─────────────────────────────────────────────────────

---

## 6.2 ZaloListenerEndpointRegistry

The registry is the **lifecycle owner** for all listener containers. It implements Spring's `SmartLifecycle` to integrate with the application context:

```java
package dev.linhvu.zalobot.boot.config;

import java.util.Collection;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

import dev.linhvu.zalobot.listener.UpdateListenerContainer;

import org.springframework.context.SmartLifecycle;

/**
 * Registry that manages all listener containers created from
 * {@link ZaloListenerEndpoint} instances.
 *
 * <p>Implements {@link SmartLifecycle} to start/stop all containers
 * as part of the Spring application context lifecycle.
 *
 * @author Linh Vu
 * @since 0.1.0
 * @see ZaloListenerEndpointRegistrar
 * @see ZaloListenerContainerFactory
 */
public class ZaloListenerEndpointRegistry implements SmartLifecycle {

    private final Map<String, UpdateListenerContainer> listenerContainers =
            new ConcurrentHashMap<>();

    private final AtomicInteger counter = new AtomicInteger();
    private volatile boolean running = false;

    /**
     * Create a container from the given endpoint and factory, then register it.
     *
     * @param endpoint the endpoint describing the listener
     * @param factory the factory to create the container
     */
    public void registerListenerContainer(ZaloListenerEndpoint endpoint,
            ZaloListenerContainerFactory<?> factory) {

        String id = resolveId(endpoint);

        if (this.listenerContainers.containsKey(id)) {
            throw new IllegalStateException(
                    "A listener container with id '" + id + "' is already registered");
        }

        UpdateListenerContainer container = factory.createListenerContainer(endpoint);
        this.listenerContainers.put(id, container);
    }

    /**
     * Return the container with the given id, or {@code null}.
     */
    public UpdateListenerContainer getListenerContainer(String id) {
        return this.listenerContainers.get(id);
    }

    /**
     * Return the ids of all registered containers.
     */
    public Set<String> getListenerContainerIds() {
        return Set.copyOf(this.listenerContainers.keySet());
    }

    /**
     * Return all registered containers.
     */
    public Collection<UpdateListenerContainer> getAllListenerContainers() {
        return this.listenerContainers.values();
    }

    // ── SmartLifecycle implementation ──────────────────────────

    @Override
    public void start() {
        for (UpdateListenerContainer container : this.listenerContainers.values()) {
            if (!container.isRunning()) {
                container.start();
            }
        }
        this.running = true;
    }

    @Override
    public void stop() {
        this.running = false;
        for (UpdateListenerContainer container : this.listenerContainers.values()) {
            if (container.isRunning()) {
                container.stop();
            }
        }
    }

    @Override
    public boolean isRunning() {
        return this.running;
    }

    @Override
    public boolean isAutoStartup() {
        return true;
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE;  // start last, stop first
    }

    // ── Private helpers ───────────────────────────────────────

    private String resolveId(ZaloListenerEndpoint endpoint) {
        String id = endpoint.getId();
        if (id == null || id.isBlank()) {
            id = "dev.linhvu.zalobot.boot.config.ZaloListenerEndpointRegistry#"
                    + this.counter.getAndIncrement();
        }
        return id;
    }
}
```

---

## 6.3 Key Registry Design Decisions

### Why `ConcurrentHashMap`?

The BPP processes beans on the main thread, but containers start on their own threads. `ConcurrentHashMap` provides thread-safe reads without locking. Spring Kafka's `KafkaListenerEndpointRegistry` also uses a `ConcurrentHashMap` (with a `ReentrantLock` for extra safety during registration).

### Why `getPhase() = Integer.MAX_VALUE`?

`SmartLifecycle.getPhase()` determines start/stop ordering:
- **Higher phase → starts later, stops earlier**
- Listener containers should start AFTER all business beans are ready
- They should stop BEFORE business beans are destroyed

`Integer.MAX_VALUE` ensures containers are the LAST to start and FIRST to stop. Spring Kafka uses the same value.

### Why auto-generate IDs?

If `@ZaloListener` doesn't specify `id`, the registry generates one:
```
dev.linhvu.zalobot.boot.config.ZaloListenerEndpointRegistry#0
dev.linhvu.zalobot.boot.config.ZaloListenerEndpointRegistry#1
```

Spring Kafka uses a similar pattern with `KafkaListenerEndpointRegistry#0`. The fully-qualified class name prevents collisions with user-defined IDs.

---

## 6.4 ZaloListenerEndpointRegistrar

The registrar collects endpoints during bean post-processing and flushes them to the registry after all beans are initialized:

```java
package dev.linhvu.zalobot.boot.config;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.InitializingBean;

/**
 * Helper that collects {@link ZaloListenerEndpoint} instances and
 * registers them with the {@link ZaloListenerEndpointRegistry}
 * after all beans have been initialized.
 *
 * <p>This two-phase approach ensures all beans are available before
 * any container is created.
 *
 * @author Linh Vu
 * @since 0.1.0
 * @see ZaloListenerAnnotationBeanPostProcessor
 * @see ZaloListenerEndpointRegistry
 */
public class ZaloListenerEndpointRegistrar implements InitializingBean {

    private ZaloListenerEndpointRegistry endpointRegistry;
    private String defaultContainerFactoryBeanName;
    private BeanFactory beanFactory;

    private final List<ZaloListenerEndpointDescriptor> endpointDescriptors =
            new ArrayList<>();

    public void setEndpointRegistry(ZaloListenerEndpointRegistry endpointRegistry) {
        this.endpointRegistry = endpointRegistry;
    }

    public ZaloListenerEndpointRegistry getEndpointRegistry() {
        return this.endpointRegistry;
    }

    public void setDefaultContainerFactoryBeanName(String factoryBeanName) {
        this.defaultContainerFactoryBeanName = factoryBeanName;
    }

    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    /**
     * Register an endpoint with an optional factory bean name.
     * The endpoint is stored and will be materialized into a container
     * when {@link #registerAllEndpoints()} is called.
     *
     * @param endpoint the endpoint to register
     * @param factoryBeanName the factory bean name, or {@code null} for default
     */
    public void registerEndpoint(ZaloListenerEndpoint endpoint,
            String factoryBeanName) {

        ZaloListenerEndpointDescriptor descriptor =
                new ZaloListenerEndpointDescriptor(endpoint, factoryBeanName);
        this.endpointDescriptors.add(descriptor);
    }

    /**
     * Called after all properties are set. Materializes all collected
     * endpoints into containers via the registry.
     */
    @Override
    public void afterPropertiesSet() {
        registerAllEndpoints();
    }

    /**
     * Flush all collected endpoints to the registry.
     */
    private void registerAllEndpoints() {
        for (ZaloListenerEndpointDescriptor descriptor : this.endpointDescriptors) {
            ZaloListenerContainerFactory<?> factory = resolveFactory(descriptor);
            this.endpointRegistry.registerListenerContainer(
                    descriptor.endpoint, factory);
        }
        this.endpointDescriptors.clear();
    }

    /**
     * Resolve the container factory for the given descriptor.
     * Uses the descriptor's factory name if specified, otherwise the default.
     */
    private ZaloListenerContainerFactory<?> resolveFactory(
            ZaloListenerEndpointDescriptor descriptor) {

        String factoryName = descriptor.factoryBeanName;
        if (factoryName == null || factoryName.isBlank()) {
            factoryName = this.defaultContainerFactoryBeanName;
        }
        if (factoryName == null || factoryName.isBlank()) {
            throw new IllegalStateException(
                    "No container factory bean name specified for endpoint ["
                    + descriptor.endpoint.getId()
                    + "] and no default factory bean name configured");
        }
        return this.beanFactory.getBean(factoryName, ZaloListenerContainerFactory.class);
    }

    // ── Inner classes ─────────────────────────────────────────

    /**
     * Descriptor that pairs an endpoint with its factory bean name.
     */
    private record ZaloListenerEndpointDescriptor(
            ZaloListenerEndpoint endpoint,
            String factoryBeanName) {
    }
}
```

---

## 6.5 How Registrar and Registry Work Together

```
Timeline:
─────────────────────────────────────────────────────────────────

Phase 1: Bean post-processing
  BPP processes Bean A:
    found @ZaloListener on method handleText()
    → registrar.registerEndpoint(endpoint1, null)  // stored in list

  BPP processes Bean B:
    found @ZaloListener on method handleImage()
    → registrar.registerEndpoint(endpoint2, "customFactory")  // stored in list

Phase 2: After all singletons instantiated
  BPP calls registrar.afterPropertiesSet()
    → registrar.registerAllEndpoints()
      │
      ├── for endpoint1:
      │   factory = beanFactory.getBean("zaloListenerContainerFactory")
      │   registry.registerListenerContainer(endpoint1, factory)
      │     → factory creates container1
      │     → endpoint1.setupListenerContainer(container1)
      │     → registry stores container1 as "...#0"
      │
      └── for endpoint2:
          factory = beanFactory.getBean("customFactory")
          registry.registerListenerContainer(endpoint2, factory)
            → factory creates container2
            → endpoint2.setupListenerContainer(container2)
            → registry stores container2 as "...#1"

Phase 3: Application context refresh (SmartLifecycle)
  registry.start()
    → container1.start()
    → container2.start()
    → All listener threads begin polling Zalo API
```

> ★ **Insight** ─────────────────────────────────────
> - **Factory resolved by bean name, not by injection.** Notice `beanFactory.getBean(factoryName, ZaloListenerContainerFactory.class)`. The factory is looked up by name at registration time, not injected at construction time. This is critical because different `@ZaloListener` methods can specify different factory bean names. Spring Kafka's `KafkaListenerEndpointRegistrar.resolveContainerFactory()` (line ~260) does the same lazy lookup.
> - **Why `endpointDescriptors.clear()` after registration?** To release references. Once endpoints are materialized into containers, keeping the descriptors around would waste memory and create confusing state (are they registered or not?). It also prevents double-registration if `afterPropertiesSet()` is called twice.
> ─────────────────────────────────────────────────────

---

## 6.6 Interaction Between Registry and the Existing Lifecycle Bean

Currently, `ZaloBotListenerAutoConfiguration` creates a `ZaloBotListenerContainerLifecycle` that wraps a single container:

```java
// Current: ZaloBotListenerContainerLifecycle.java (existing)
class ZaloBotListenerContainerLifecycle implements SmartLifecycle {
    private final UpdateListenerContainer container;  // single container
    // start(), stop(), isRunning() delegate to container
}
```

With the new `@ZaloListener` infrastructure, the **Registry** replaces this lifecycle bean. The registry manages ALL containers (not just one) and implements `SmartLifecycle` itself.

The existing `ZaloBotListenerContainerLifecycle` will still be used for the **non-annotation** path (when users define `UpdateListener` beans manually). Chapter 9 covers how auto-configuration chooses between the two paths.

---

## 6.7 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `ZaloListenerEndpointRegistry` | `KafkaListenerEndpointRegistry` | `config/KafkaListenerEndpointRegistry.java` |
| `registerListenerContainer(ep, factory)` | `registerListenerContainer(ep, factory, startImmediately)` | `KafkaListenerEndpointRegistry.java:258` |
| `getListenerContainer(id)` | `getListenerContainer(String id)` | `KafkaListenerEndpointRegistry.java:117` |
| `SmartLifecycle` implementation | `SmartLifecycle` implementation | `KafkaListenerEndpointRegistry.java` |
| `getPhase() = MAX_VALUE` | `getPhase()` (configurable, default varies) | `KafkaListenerEndpointRegistry.java` |
| `ZaloListenerEndpointRegistrar` | `KafkaListenerEndpointRegistrar` | `config/KafkaListenerEndpointRegistrar.java` |
| `registerEndpoint(ep, factoryName)` | `registerEndpoint(ep, factory)` | `KafkaListenerEndpointRegistrar.java:241` |
| `registerAllEndpoints()` | `registerAllEndpoints()` | `KafkaListenerEndpointRegistrar.java:193` |
| `ZaloListenerEndpointDescriptor` | `KafkaListenerEndpointDescriptor` | Inner class in `KafkaListenerEndpointRegistrar.java` |

**Key simplifications:**
- Spring Kafka's registry supports `startImmediately` flag, container groups, and `ListenerContainerRegistry` interface. We skip all of these.
- Spring Kafka's registrar supports `KafkaListenerConfigurer` callback for programmatic registration. We skip this.
- Spring Kafka resolves factories by direct reference OR bean name. We only use bean name (simpler).

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ZaloListenerEndpointRegistry** | Manages all listener containers; implements `SmartLifecycle` |
| **ZaloListenerEndpointRegistrar** | Collects endpoints during BPP phase; flushes to registry after all beans ready |
| **Two-phase registration** | Collect during post-processing → materialize after `afterSingletonsInstantiated()` |
| **Auto-generated IDs** | `ZaloListenerEndpointRegistry#0`, `#1`, etc. when no `id` specified |
| **Factory by bean name** | `beanFactory.getBean(name)` — allows different factories per listener |
| **SmartLifecycle** | Phase `MAX_VALUE` → starts last, stops first |

**Next: Chapter 7** — We build the `ZaloListenerAnnotationBeanPostProcessor`, the component that scans all beans for `@ZaloListener` annotations and feeds endpoints to the registrar.
