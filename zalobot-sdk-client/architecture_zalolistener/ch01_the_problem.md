# Chapter 1: The Problem — Why We Need @ZaloListener

## What You'll Learn
- How ZaloBot listeners are currently configured (manual wiring)
- Why manual `UpdateListener` beans don't scale as the application grows
- What the ideal developer experience looks like (declarative annotations)
- The gap between the current state and the desired state

---

## 1.1 The Current Approach: Manual Wiring

Today, to handle incoming Zalo messages, you must define an `UpdateListener` bean and let the auto-configuration wire it into a container:

```java
@SpringBootApplication
public class MyZaloBotApp {

    public static void main(String[] args) {
        SpringApplication.run(MyZaloBotApp.class, args);
    }

    @Bean
    UpdateListener myUpdateListener() {
        return update -> {
            if (update.isTextMessage()) {
                System.out.println("Got text: " + update.message().text());
            }
        };
    }
}
```

This works because `ZaloBotListenerAutoConfiguration` detects the `UpdateListener` bean and creates a `ConcurrentUpdateListenerContainer`:

```java
// ZaloBotListenerAutoConfiguration.java (existing)
@Bean
@ConditionalOnBean(UpdateListener.class)        // ← requires YOUR bean
@ConditionalOnMissingBean(UpdateListenerContainer.class)
UpdateListenerContainer zaloBotListenerContainer(
        ZaloBotClient client, ContainerProperties containerProperties) {
    ConcurrentUpdateListenerContainer container =
            new ConcurrentUpdateListenerContainer(client, containerProperties);
    container.setConcurrency(properties.getListener().getConcurrency());
    return container;
}
```

**This approach has three fundamental problems.**

---

## 1.2 Problem 1: Only One Listener Per Application

Since `ZaloBotListenerAutoConfiguration` uses `@ConditionalOnBean(UpdateListener.class)` and `@ConditionalOnMissingBean(UpdateListenerContainer.class)`, you can only have **one** `UpdateListener` bean. If you try to define two:

```java
@Bean
UpdateListener textHandler() {
    return update -> { /* handle text */ };
}

@Bean
UpdateListener imageHandler() {
    return update -> { /* handle images */ };
}
```

Spring will complain about ambiguous bean resolution — `ObjectProvider<UpdateListener>` won't know which one to inject. You'd have to manually compose them into a single dispatcher, which defeats the purpose.

---

## 1.3 Problem 2: No Method-Level Granularity

Compare the current approach with what Spring Kafka offers:

```java
// Spring Kafka — clean, method-level declaration
@Component
public class MyKafkaConsumer {

    @KafkaListener(topics = "orders")
    public void handleOrder(ConsumerRecord<String, String> record) {
        // handle orders
    }

    @KafkaListener(topics = "notifications")
    public void handleNotification(ConsumerRecord<String, String> record) {
        // handle notifications
    }
}
```

Each `@KafkaListener` method gets its OWN container, its own concurrency, its own lifecycle. The developer doesn't think about containers, factories, or registries — they just annotate methods.

With ZaloBot today, you're forced to:
1. Define an `UpdateListener` bean (imperative)
2. Handle ALL update types inside that single listener (big `if/else` chain)
3. Manage a single container for everything

---

## 1.4 Problem 3: No Per-Listener Configuration

What if you want different concurrency for different handlers? Or you want one handler to auto-start and another to start manually? The current approach gives you one container with one set of properties:

```yaml
zalobot:
  listener:
    concurrency: 3   # applies to everything — no per-handler control
```

---

## 1.5 The Desired Developer Experience

What we WANT is this:

```java
@Component
public class MyZaloBotHandlers {

    @ZaloListener(id = "text-handler", concurrency = "3")
    public void handleTextMessage(GetUpdatesResult update) {
        if (update.isTextMessage()) {
            System.out.println("Text: " + update.message().text());
        }
    }

    @ZaloListener(id = "image-handler", concurrency = "1")
    public void handleImageMessage(GetUpdatesResult update) {
        if (update.isImageMessage()) {
            System.out.println("Image received!");
        }
    }
}
```

Each `@ZaloListener` method should:
- Get its own `ConcurrentUpdateListenerContainer`
- Have independently configurable concurrency, auto-startup, etc.
- Be automatically discovered and wired — no manual bean definitions
- Support flexible method parameters (`GetUpdatesResult`, `Message`, `String`)

---

## 1.6 The Gap: What Needs to Be Built

To go from manual wiring to `@ZaloListener`, we need **7 new components**:

```
Current:                              Needed:
─────────────────                     ──────────────────────────────────

@Bean UpdateListener ──────────┐      @ZaloListener annotation
                               │          │
                               │      Bean Post-Processor (discovers annotations)
                               │          │
                               │      Endpoint (bundles method + config)
                               │          │
                               ▼      Adapter (method → UpdateListener)
ConcurrentUpdateListener       │          │
Container (auto-configured)    │      Container Factory (creates container)
                               │          │
                               │      Registry (manages lifecycle)
                               │          │
                               └──    Registrar (collects endpoints)
```

> ★ **Insight** ─────────────────────────────────────
> - **Why not just scan for methods directly?** You could write a simpler BPP that scans for `@ZaloListener`, creates a container, and done. But that tight coupling means you can't test endpoints in isolation, can't swap container implementations, and can't programmatically register listeners. The layered architecture (endpoint → factory → registry) follows the **Open/Closed Principle** — each layer is open for extension (custom factories, custom endpoints) without modifying existing code.
> - **This is the same gap Spring Kafka faced.** Before `@KafkaListener`, Kafka consumers required manual `ConcurrentMessageListenerContainer` setup. The annotation infrastructure was added in Spring Kafka 1.1 and has remained architecturally stable for 10+ years — a sign that the layered approach is the right one.
> ─────────────────────────────────────────────────────

---

## 1.7 Connection to Real Spring Kafka

| Current ZaloBot | Spring Kafka Equivalent | What's Missing |
|---|---|---|
| `UpdateListener` bean (manual) | `MessageListener` (manual) | `@KafkaListener` annotation + infrastructure |
| `ConcurrentUpdateListenerContainer` (auto-configured) | `ConcurrentMessageListenerContainer` (auto-configured) | Per-listener container creation via factory |
| `ZaloBotListenerAutoConfiguration` | `KafkaAutoConfiguration` | Container factory + registry auto-config |

The rest of this guide builds each component from the inside out, starting with the lowest-level piece (the adapter) and working up to the highest-level orchestrator (the auto-configuration).

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Manual wiring** | Current approach: define `UpdateListener` bean, auto-config creates container |
| **Single listener limit** | Only one `UpdateListener` bean supported per application context |
| **No method-level control** | Can't annotate individual methods as update handlers |
| **No per-listener config** | All listeners share one container's concurrency/lifecycle settings |
| **@ZaloListener** | Desired annotation that enables declarative, per-method listener setup |

**Next: Chapter 2** — We define the `@ZaloListener` annotation itself, the user-facing API that developers will interact with.
