# Chapter 5: The Container Factory — Creating Containers from Endpoints

## What You'll Learn
- Why a factory interface decouples endpoint processing from container creation
- How `ConcurrentZaloListenerContainerFactory` creates and configures containers
- The order of operations: create → configure → setup listener
- How factory defaults interact with endpoint overrides

---

## 5.1 Why a Factory?

At this point we have:
- **Endpoint** (Ch04): knows the method, bean, and per-listener configuration
- **Adapter** (Ch03): knows how to invoke the method when an update arrives

But who creates the `ConcurrentUpdateListenerContainer`? And who injects the `ZaloBotClient` and `ContainerProperties`?

The **factory** is the answer. It holds the "shared" configuration (client, default properties, default concurrency) and produces containers on demand. Each `@ZaloListener` method gets its own container from the factory.

```
Factory (shared state)                 Endpoint (per-listener state)
┌──────────────────────┐              ┌─────────────────────────┐
│ - client: ZaloBotClient             │ - id: "text-handler"    │
│ - containerProperties │              │ - bean: myBean          │
│ - concurrency: 1      │              │ - method: handleText()  │
│   (default)           │              │ - concurrency: 3        │
└──────────┬───────────┘              │   (override)            │
           │                          └──────────┬──────────────┘
           │       createListenerContainer(endpoint)
           ├──────────────────────────────────────┤
           │                                      │
           ▼                                      │
    ┌─────────────────────────────┐               │
    │ ConcurrentUpdateListener    │               │
    │ Container                   │               │
    │ ───────────────────────     │               │
    │ concurrency = 3 (from EP)  │◄──── endpoint.setupListenerContainer()
    │ listener = adapter          │        sets the UpdateListenerAdapter
    │ client = factory's client   │
    └─────────────────────────────┘
```

---

## 5.2 The Factory Interface

```java
package dev.linhvu.zalobot.boot.config;

import dev.linhvu.zalobot.listener.UpdateListenerContainer;

/**
 * Factory for creating {@link UpdateListenerContainer} instances
 * from {@link ZaloListenerEndpoint} metadata.
 *
 * @param <C> the container type
 * @author Linh Vu
 * @since 0.1.0
 * @see ConcurrentZaloListenerContainerFactory
 */
public interface ZaloListenerContainerFactory<C extends UpdateListenerContainer> {

    /**
     * Create and configure a listener container for the given endpoint.
     *
     * @param endpoint the endpoint describing the listener
     * @return a fully configured, but NOT yet started, container
     */
    C createListenerContainer(ZaloListenerEndpoint endpoint);
}
```

**Why generic `<C extends UpdateListenerContainer>`?**

This allows the factory to declare exactly what type of container it creates. The registry knows it gets an `UpdateListenerContainer`, but testing code or custom factories can be more specific. Spring Kafka's `KafkaListenerContainerFactory<C extends MessageListenerContainer>` uses the same pattern.

---

## 5.3 The Concrete Factory

```java
package dev.linhvu.zalobot.boot.config;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.listener.ConcurrentUpdateListenerContainer;
import dev.linhvu.zalobot.listener.ContainerProperties;

/**
 * A {@link ZaloListenerContainerFactory} that creates
 * {@link ConcurrentUpdateListenerContainer} instances.
 *
 * <p>This factory holds shared defaults (client, container properties,
 * concurrency) that apply to all containers it creates. Per-listener
 * overrides from the endpoint take precedence.
 *
 * @author Linh Vu
 * @since 0.1.0
 */
public class ConcurrentZaloListenerContainerFactory
        implements ZaloListenerContainerFactory<ConcurrentUpdateListenerContainer> {

    private ZaloBotClient client;
    private ContainerProperties containerProperties;
    private Integer concurrency;
    private Boolean autoStartup;

    public void setClient(ZaloBotClient client) {
        this.client = client;
    }

    public void setContainerProperties(ContainerProperties containerProperties) {
        this.containerProperties = containerProperties;
    }

    public void setConcurrency(Integer concurrency) {
        this.concurrency = concurrency;
    }

    public void setAutoStartup(Boolean autoStartup) {
        this.autoStartup = autoStartup;
    }

    @Override
    public ConcurrentUpdateListenerContainer createListenerContainer(
            ZaloListenerEndpoint endpoint) {

        // 1. Create fresh ContainerProperties (copy defaults)
        ContainerProperties properties = cloneContainerProperties();

        // 2. Create the container
        ConcurrentUpdateListenerContainer container =
                new ConcurrentUpdateListenerContainer(this.client, properties);

        // 3. Apply concurrency: endpoint override > factory default > 1
        Integer endpointConcurrency = endpoint.getConcurrency();
        if (endpointConcurrency != null) {
            container.setConcurrency(endpointConcurrency);
        }
        else if (this.concurrency != null) {
            container.setConcurrency(this.concurrency);
        }

        // 4. Let the endpoint install its listener on the container
        endpoint.setupListenerContainer(container);

        return container;
    }

    /**
     * Clone the container properties so each container gets its own instance.
     * This prevents containers from sharing mutable state.
     */
    private ContainerProperties cloneContainerProperties() {
        ContainerProperties clone = new ContainerProperties();
        if (this.containerProperties != null) {
            clone.setPollTimeout(this.containerProperties.getPollTimeout());
            clone.setPollInterval(this.containerProperties.getPollInterval());
            clone.setShutdownTimeout(this.containerProperties.getShutdownTimeout());
            clone.setBackOffInterval(this.containerProperties.getBackOffInterval());
            clone.setMaxBackOffInterval(this.containerProperties.getMaxBackOffInterval());
            clone.setErrorHandler(this.containerProperties.getErrorHandler());
        }
        return clone;
    }
}
```

---

## 5.4 The Order of Operations

The factory follows a strict order:

```
createListenerContainer(endpoint)
    │
    ├── Step 1: Clone ContainerProperties
    │   (each container gets its OWN properties — no shared mutable state)
    │
    ├── Step 2: Create ConcurrentUpdateListenerContainer(client, properties)
    │   (container exists but has NO listener yet)
    │
    ├── Step 3: Apply concurrency override
    │   endpoint.getConcurrency() != null ? use it : factory default
    │
    ├── Step 4: endpoint.setupListenerContainer(container)
    │   │
    │   ├── 4a: endpoint.createUpdateListener()
    │   │        → creates UpdateListenerAdapter(bean, method)
    │   │
    │   └── 4b: container.setUpdateListener(adapter)
    │
    └── Return: configured container (NOT started)
```

**Why clone ContainerProperties?**

Each `@ZaloListener` method gets its own container. If they shared the same `ContainerProperties` object, one container's `setUpdateListener()` would overwrite another's. Cloning ensures isolation.

> ★ **Insight** ─────────────────────────────────────
> - **Factory configures FIRST, endpoint sets listener SECOND.** This ordering matters. The factory handles "infrastructure" concerns (concurrency, properties). The endpoint handles "application" concerns (which listener to install). Spring Kafka's `AbstractKafkaListenerContainerFactory.createListenerContainer()` follows the same order: `createContainerInstance()` → `initializeContainer()` → `endpoint.setupListenerContainer()`. If you reverse the order (set listener first, then configure), you risk overwriting the listener during configuration.
> - **Why the factory returns a NOT-started container:** Starting is the registry's job, not the factory's. The factory creates and configures; the registry manages lifecycle (start, stop, pause, resume). This separation means the registry can decide WHEN to start (e.g., after all containers are registered, or based on `autoStartup` flag).
> ─────────────────────────────────────────────────────

---

## 5.5 Factory Defaults vs. Endpoint Overrides

The precedence chain for concurrency:

```
@ZaloListener(concurrency = "5")     →  endpoint.getConcurrency() = 5   →  USE THIS
          OR
@ZaloListener()                      →  endpoint.getConcurrency() = null
                                     →  factory.concurrency = 3          →  USE THIS
          OR
@ZaloListener()                      →  endpoint.getConcurrency() = null
                                     →  factory.concurrency = null
                                     →  ConcurrentUpdateListenerContainer default = 1  →  USE THIS
```

This three-level fallback (endpoint → factory → container default) mirrors Spring Kafka's `ConcurrentKafkaListenerContainerFactory.initializeContainer()`:

```java
// ConcurrentKafkaListenerContainerFactory.java:85 (Spring Kafka)
@Override
protected void initializeContainer(ConcurrentMessageListenerContainer<K, V> instance,
        KafkaListenerEndpoint endpoint) {
    super.initializeContainer(instance, endpoint);
    Integer conc = endpoint.getConcurrency();
    if (conc != null) {
        instance.setConcurrency(conc);
    }
    else if (this.concurrency != null) {
        instance.setConcurrency(this.concurrency);
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Three-level configuration cascade** is a common Spring pattern: annotation → factory → framework default. You see it everywhere: `@Transactional(timeout=5)` → `TransactionManager.defaultTimeout` → no timeout. This gives maximum flexibility: users who care can override at the annotation level, teams can set organizational defaults at the factory level, and the framework provides sensible fallbacks.
> ─────────────────────────────────────────────────────

---

## 5.6 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `ZaloListenerContainerFactory<C>` | `KafkaListenerContainerFactory<C>` | `config/KafkaListenerContainerFactory.java` |
| `ConcurrentZaloListenerContainerFactory` | `ConcurrentKafkaListenerContainerFactory` | `config/ConcurrentKafkaListenerContainerFactory.java` |
| `createListenerContainer(endpoint)` | `createListenerContainer(endpoint)` | `AbstractKafkaListenerContainerFactory.java` |
| Clone `ContainerProperties` | `ConsumerFactory` creates new consumers per container | Different mechanism, same goal: isolation |
| Concurrency override chain | `initializeContainer()` concurrency logic | `ConcurrentKafkaListenerContainerFactory.java:85` |
| No `createContainer(topics...)` | `createContainer(String... topics)` convenience methods | Not needed — ZaloBot has no topics |
| No `AbstractKafkaListenerContainerFactory` | Abstract factory base class | Simplified — single concrete factory is sufficient |

**Key simplification:** Spring Kafka has `AbstractKafkaListenerContainerFactory` (base) → `ConcurrentKafkaListenerContainerFactory` (concrete). The base class handles common setup (error handler, batch listener, record interceptor, etc.). We skip the abstract base because our factory has no complex shared logic worth abstracting yet.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ZaloListenerContainerFactory** | Interface: creates containers from endpoint descriptions |
| **ConcurrentZaloListenerContainerFactory** | Concrete: creates `ConcurrentUpdateListenerContainer` instances |
| **Clone properties** | Each container gets its own `ContainerProperties` to avoid shared state |
| **Order: configure → setup** | Factory configures infrastructure, then endpoint installs listener |
| **Three-level cascade** | Concurrency: endpoint → factory → container default |
| **Container NOT started** | Factory creates, registry manages lifecycle |

**Next: Chapter 6** — We build the `ZaloListenerEndpointRegistry` (lifecycle manager) and `ZaloListenerEndpointRegistrar` (endpoint collector) that connect the BPP to the factory.
