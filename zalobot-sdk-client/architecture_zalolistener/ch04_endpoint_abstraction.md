# Chapter 4: The Endpoint Abstraction — Bundling "What" with "How"

## What You'll Learn
- What a "listener endpoint" represents and why it's needed as an abstraction
- The interface → abstract class → concrete class hierarchy
- How the Template Method pattern cleanly separates concerns
- How `MethodZaloListenerEndpoint` uses `UpdateListenerAdapter` from Chapter 3

---

## 4.1 Why Do We Need an Endpoint Abstraction?

After Chapter 3, we have an `UpdateListenerAdapter` that can invoke an annotated method. But where do we store the **configuration** for that listener?

Consider what we know at annotation-processing time:
- The **target bean** and **method** (discovered by BPP)
- The **id** (from annotation)
- The **concurrency** (from annotation)
- The **autoStartup** setting (from annotation)

And what we need at container-creation time:
- An `UpdateListener` to set on the container
- The concurrency to configure on the container
- Whether to auto-start

The **endpoint** is the data structure that carries all this information from discovery time to creation time. It's the "work order" that the BPP writes and the factory reads.

```
BPP discovers @ZaloListener                  Factory creates container
         │                                           │
         ▼                                           ▼
┌─────────────────────────────┐            ┌─────────────────────┐
│   MethodZaloListenerEndpoint│───────────▶│ ConcurrentUpdate    │
│   ─────────────────────     │  "build    │ ListenerContainer   │
│   id = "text-handler"       │   this"    │                     │
│   bean = myHandlerBean      │            │ concurrency = 3     │
│   method = handleText()     │            │ listener = adapter  │
│   concurrency = 3           │            │ running = true      │
│   autoStartup = true        │            └─────────────────────┘
└─────────────────────────────┘
```

---

## 4.2 The ZaloListenerEndpoint Interface

```java
package dev.linhvu.zalobot.boot.config;

import dev.linhvu.zalobot.listener.UpdateListenerContainer;

/**
 * Model for a Zalo listener endpoint. Can be used against a
 * {@link ZaloListenerContainerFactory} to create a container.
 *
 * @author Linh Vu
 * @since 0.1.0
 * @see MethodZaloListenerEndpoint
 * @see ZaloListenerContainerFactory
 */
public interface ZaloListenerEndpoint {

    /**
     * Return the id of this endpoint.
     * @return the endpoint id, or {@code null} if not set
     */
    String getId();

    /**
     * Return the concurrency for this endpoint's container.
     * @return the concurrency, or {@code null} to use the factory default
     */
    Integer getConcurrency();

    /**
     * Return the auto-startup flag for this endpoint's container.
     * @return the auto-startup flag, or {@code null} to use the factory default
     */
    Boolean getAutoStartup();

    /**
     * Setup the specified listener container with the model defined
     * by this endpoint.
     *
     * <p>This method is called by the factory after creating the container
     * but before starting it.
     *
     * @param listenerContainer the container to configure
     */
    void setupListenerContainer(UpdateListenerContainer listenerContainer);
}
```

**Why `Integer` and `Boolean` (boxed) instead of `int` and `boolean`?**

Because `null` means "not specified — use the factory default." With primitive types, you can't distinguish "user set concurrency to 0" from "user didn't specify concurrency." Spring Kafka uses the same convention: `KafkaListenerEndpoint.getConcurrency()` returns `Integer`.

---

## 4.3 The Abstract Base Class

```java
package dev.linhvu.zalobot.boot.config;

import dev.linhvu.zalobot.listener.UpdateListener;
import dev.linhvu.zalobot.listener.UpdateListenerContainer;

/**
 * Base implementation of {@link ZaloListenerEndpoint} providing common
 * configuration storage and the Template Method for container setup.
 *
 * @author Linh Vu
 * @since 0.1.0
 */
public abstract class AbstractZaloListenerEndpoint implements ZaloListenerEndpoint {

    private String id;
    private Integer concurrency;
    private Boolean autoStartup;

    @Override
    public String getId() {
        return this.id;
    }

    public void setId(String id) {
        this.id = id;
    }

    @Override
    public Integer getConcurrency() {
        return this.concurrency;
    }

    public void setConcurrency(Integer concurrency) {
        this.concurrency = concurrency;
    }

    @Override
    public Boolean getAutoStartup() {
        return this.autoStartup;
    }

    public void setAutoStartup(Boolean autoStartup) {
        this.autoStartup = autoStartup;
    }

    /**
     * Template Method: sets up the listener container by:
     * 1. Creating an {@link UpdateListener} (delegated to subclass)
     * 2. Setting the listener on the container
     */
    @Override
    public void setupListenerContainer(UpdateListenerContainer listenerContainer) {
        UpdateListener listener = createUpdateListener();
        if (listener == null) {
            throw new IllegalStateException(
                    "Endpoint [" + this.id + "] failed to create an UpdateListener");
        }
        listenerContainer.setUpdateListener(listener);
    }

    /**
     * Create the {@link UpdateListener} for this endpoint.
     * <p>Subclasses implement this to determine HOW the listener is created
     * (e.g., from an annotated method, from a lambda, etc.).
     *
     * @return the update listener, never {@code null}
     */
    protected abstract UpdateListener createUpdateListener();
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Template Method pattern:** `setupListenerContainer()` defines the algorithm (create listener → set on container) while `createUpdateListener()` is the "hook" that subclasses override. This means if you later add a `LambdaZaloListenerEndpoint` or `ClassLevelZaloListenerEndpoint`, the setup flow stays the same — only the listener creation changes. Spring Kafka's `AbstractKafkaListenerEndpoint.setupListenerContainer()` (line ~506) follows this exact pattern.
> - **Why separate `setupListenerContainer()` from the constructor?** Because the container doesn't exist when the endpoint is created. The BPP creates endpoints during bean post-processing, but containers are created later by the factory. This temporal separation is why the endpoint has a setup method rather than injecting the container at construction time.
> ─────────────────────────────────────────────────────

---

## 4.4 The Method Endpoint

```java
package dev.linhvu.zalobot.boot.config;

import java.lang.reflect.Method;

import dev.linhvu.zalobot.boot.listener.adapter.UpdateListenerAdapter;
import dev.linhvu.zalobot.listener.UpdateListener;

/**
 * A {@link ZaloListenerEndpoint} backed by a specific bean and method.
 * Created by the {@link dev.linhvu.zalobot.boot.annotation.ZaloListenerAnnotationBeanPostProcessor}
 * for each {@code @ZaloListener} annotated method.
 *
 * @author Linh Vu
 * @since 0.1.0
 */
public class MethodZaloListenerEndpoint extends AbstractZaloListenerEndpoint {

    private Object bean;
    private Method method;

    public Object getBean() {
        return this.bean;
    }

    public void setBean(Object bean) {
        this.bean = bean;
    }

    public Method getMethod() {
        return this.method;
    }

    public void setMethod(Method method) {
        this.method = method;
    }

    @Override
    protected UpdateListener createUpdateListener() {
        if (this.bean == null) {
            throw new IllegalStateException("No bean set on endpoint [" + getId() + "]");
        }
        if (this.method == null) {
            throw new IllegalStateException("No method set on endpoint [" + getId() + "]");
        }
        return new UpdateListenerAdapter(this.bean, this.method);
    }

    @Override
    public String toString() {
        return "MethodZaloListenerEndpoint [id=" + getId()
                + ", bean=" + (this.bean != null ? this.bean.getClass().getSimpleName() : "null")
                + ", method=" + (this.method != null ? this.method.getName() : "null")
                + "]";
    }
}
```

---

## 4.5 How It All Connects

```
                             ┌─────────────────────┐
                             │ ZaloListenerEndpoint │
                             │    «interface»       │
                             └──────────┬──────────┘
                                        │
                                        │ implements
                                        │
                             ┌──────────┴──────────┐
                             │ AbstractZaloListener │
                             │    Endpoint          │
                             │ ──────────────────── │
                             │ - id, concurrency,   │
                             │   autoStartup        │
                             │ ──────────────────── │
                             │ + setupListenerCont. │
                             │ # createUpdateList.  │ ← abstract hook
                             └──────────┬──────────┘
                                        │
                                        │ extends
                                        │
                             ┌──────────┴──────────┐
                             │ MethodZaloListener   │
                             │    Endpoint          │
                             │ ──────────────────── │
                             │ - bean: Object       │
                             │ - method: Method     │
                             │ ──────────────────── │
                             │ # createUpdateList.  │ ← returns UpdateListenerAdapter
                             └─────────────────────┘
                                        │
                                        │ creates
                                        ▼
                             ┌─────────────────────┐
                             │ UpdateListenerAdapter│
                             │   implements         │
                             │   UpdateListener     │
                             │ ──────────────────── │
                             │ + onUpdate(result)   │
                             │   → bean.method()    │
                             └─────────────────────┘
```

The flow at container setup time:

```
factory.createListenerContainer(endpoint)
    │
    ├── 1. factory creates ConcurrentUpdateListenerContainer
    │
    ├── 2. factory calls endpoint.setupListenerContainer(container)
    │          │
    │          ├── 3. AbstractZaloListenerEndpoint calls createUpdateListener()
    │          │          │
    │          │          └── 4. MethodZaloListenerEndpoint creates
    │          │                 UpdateListenerAdapter(bean, method)
    │          │
    │          └── 5. AbstractZaloListenerEndpoint calls
    │                 container.setUpdateListener(adapter)
    │
    └── 6. factory returns configured container
```

> ★ **Insight** ─────────────────────────────────────
> - **The endpoint is a "recipe", not a "meal."** It describes what should be built (method + config) but doesn't build it until `setupListenerContainer()` is called. This lazy creation is important because the container factory might need to configure the container BEFORE the listener is set. Spring Kafka's `AbstractKafkaListenerContainerFactory.createListenerContainer()` (line ~147) calls `initializeContainer()` first, THEN `endpoint.setupListenerContainer()` — order matters.
> - **Future extensibility:** If you later want to support class-level `@ZaloListener` (like `@KafkaListener` on a class with `@KafkaHandler` methods), you'd create a `MultiMethodZaloListenerEndpoint` that extends `AbstractZaloListenerEndpoint` and overrides `createUpdateListener()` to return a dispatching listener. The rest of the infrastructure (factory, registry, BPP flow) stays unchanged.
> ─────────────────────────────────────────────────────

---

## 4.6 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `ZaloListenerEndpoint` | `KafkaListenerEndpoint` | `config/KafkaListenerEndpoint.java` |
| `AbstractZaloListenerEndpoint` | `AbstractKafkaListenerEndpoint` | `config/AbstractKafkaListenerEndpoint.java` |
| `MethodZaloListenerEndpoint` | `MethodKafkaListenerEndpoint` | `config/MethodKafkaListenerEndpoint.java` |
| `setupListenerContainer(container)` | `setupListenerContainer(container, converter)` | `AbstractKafkaListenerEndpoint.java:506` |
| `createUpdateListener()` | `createMessageListener(container, converter)` | `AbstractKafkaListenerEndpoint.java` (abstract) |
| Endpoint stores `bean` + `method` | Endpoint stores `bean` + `method` + `messageHandlerMethodFactory` | `MethodKafkaListenerEndpoint.java` |
| No `MessageConverter` parameter | `setupListenerContainer(container, messageConverter)` | Kafka converts ConsumerRecord → Message; ZaloBot has fixed format |

**Key simplification:** Spring Kafka's endpoint also handles:
- `RecordFilterStrategy` (wrapping listener in `FilteringMessageListenerAdapter`)
- `BatchToRecordAdapter` (converting batch to individual records)
- Message converter negotiation
- Reply template configuration

We strip all of these. Our endpoint ONLY creates an `UpdateListenerAdapter` and sets it on the container.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ZaloListenerEndpoint** | Interface: describes a listener endpoint (id, concurrency, setup method) |
| **AbstractZaloListenerEndpoint** | Base class: stores config, implements Template Method for setup |
| **MethodZaloListenerEndpoint** | Concrete: holds bean + method, creates `UpdateListenerAdapter` |
| **Template Method** | `setupListenerContainer()` defines the flow; `createUpdateListener()` is the hook |
| **Boxed types** | `Integer`/`Boolean` allow `null` to mean "use factory default" |

**Next: Chapter 5** — We build the `ZaloListenerContainerFactory` that takes an endpoint and produces a fully configured `ConcurrentUpdateListenerContainer`.
