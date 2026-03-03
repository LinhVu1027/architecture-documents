# Chapter 2: Annotation Design — Defining @ZaloListener

## What You'll Learn
- How to design the `@ZaloListener` annotation with the right attributes
- Why each attribute exists and what it maps to in the infrastructure
- How property placeholder resolution works in annotation attributes
- The design trade-offs compared to `@KafkaListener`

---

## 2.1 What Attributes Does @ZaloListener Need?

Looking at the existing `ConcurrentUpdateListenerContainer` and `ContainerProperties`, the per-listener configurable values are:

| Container Property | Should it be per-listener? | Annotation Attribute? |
|---|---|---|
| `concurrency` | Yes — different handlers may need different parallelism | `concurrency` |
| `pollTimeout` | Usually global | No (use factory defaults) |
| `pollInterval` | Usually global | No (use factory defaults) |
| `shutdownTimeout` | Usually global | No (use factory defaults) |
| `errorHandler` | Could be per-listener | `errorHandler` (future) |

Plus infrastructure attributes:
- `id` — unique identifier for the container (for JMX, logging, manual start/stop)
- `containerFactory` — name of the factory bean to use
- `autoStartup` — whether this listener starts automatically

---

## 2.2 The @ZaloListener Annotation

```java
package dev.linhvu.zalobot.boot.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation that marks a method as a Zalo update listener.
 *
 * <p>The annotated method will be invoked when the ZaloBot receives updates
 * from the Zalo API. Each {@code @ZaloListener} method gets its own
 * {@link dev.linhvu.zalobot.listener.UpdateListenerContainer} with
 * independently configurable concurrency and lifecycle.
 *
 * <p>Processing is performed by a
 * {@link ZaloListenerAnnotationBeanPostProcessor} that is registered
 * via {@link EnableZaloBot @EnableZaloBot}.
 *
 * <p>Annotated methods may have flexible signatures:
 * <ul>
 *   <li>{@code void handle(GetUpdatesResult update)} — receives the full update</li>
 *   <li>{@code void handle(GetUpdatesResult.Message message)} — receives the message only</li>
 *   <li>{@code void handle(String text)} — receives the message text only</li>
 * </ul>
 *
 * @author Linh Vu
 * @since 0.1.0
 * @see EnableZaloBot
 * @see ZaloListenerAnnotationBeanPostProcessor
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ZaloListener {

    /**
     * The unique identifier of the container for this listener.
     * <p>If not specified, an auto-generated id is used.
     * <p>Supports property placeholder resolution: {@code "${my.listener.id}"}.
     * @return the container id
     */
    String id() default "";

    /**
     * The bean name of the {@link dev.linhvu.zalobot.boot.config.ZaloListenerContainerFactory}
     * to use to create the listener container.
     * <p>If not specified, the default factory ({@code "zaloListenerContainerFactory"}) is used.
     * <p>Supports property placeholder resolution.
     * @return the container factory bean name
     */
    String containerFactory() default "";

    /**
     * Override the container factory's default concurrency for this listener.
     * <p>Supports property placeholder resolution: {@code "${my.listener.concurrency}"}.
     * @return the concurrency (as a String to support property placeholders)
     */
    String concurrency() default "";

    /**
     * Whether this listener container should auto-start.
     * <p>If not specified, the container factory's default is used.
     * <p>Supports property placeholder resolution: {@code "${my.listener.auto-startup}"}.
     * @return whether to auto-start (as a String to support property placeholders)
     */
    String autoStartup() default "";
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Why are `concurrency` and `autoStartup` Strings, not `int` and `boolean`?** This is a deliberate pattern from Spring Kafka. By using `String`, we can support property placeholder resolution like `"${my.concurrency}"`. If we used `int`, the value must be a compile-time constant — no externalized configuration. The BPP resolves these strings to their actual types at runtime using Spring's `Environment`.
> - **Why `@Target(ElementType.METHOD)` only?** Spring Kafka supports both `ElementType.TYPE` (class-level, with `@KafkaHandler` methods) and `ElementType.METHOD`. We omit class-level to keep things simple. Class-level requires a `MultiMethodKafkaListenerEndpoint` that dispatches to the right `@KafkaHandler` based on payload type — significant complexity for little benefit in ZaloBot's use case.
> ─────────────────────────────────────────────────────

---

## 2.3 Why No `topics` Attribute?

The most obvious difference from `@KafkaListener` is the absence of `topics`. Spring Kafka's `@KafkaListener` MUST specify which topics to consume:

```java
// Spring Kafka — topics are required
@KafkaListener(topics = "orders")
public void handle(ConsumerRecord<String, String> record) { ... }
```

ZaloBot doesn't have topics. All updates come from a single long-polling endpoint (`/getUpdates`). Each `@ZaloListener` method receives ALL updates — filtering by event type (text, image, sticker) is the method's responsibility.

Future enhancement: we could add a `filter` attribute or event-type filtering, but that's beyond the initial implementation.

---

## 2.4 The Default Container Factory Name

Spring Kafka uses the convention:
```java
public static final String DEFAULT_KAFKA_LISTENER_CONTAINER_FACTORY_BEAN_NAME =
    "kafkaListenerContainerFactory";
```

We follow the same convention:
```java
// In a constants class or directly in the BPP
public static final String DEFAULT_ZALO_LISTENER_CONTAINER_FACTORY_BEAN_NAME =
    "zaloListenerContainerFactory";
```

When `@ZaloListener(containerFactory = "")` (the default), the BPP looks up this bean name. Users can override it to use a custom factory:

```java
@ZaloListener(containerFactory = "myCustomFactory")
public void handle(GetUpdatesResult update) { ... }
```

> ★ **Insight** ─────────────────────────────────────
> - **Convention over configuration** is the core Spring philosophy. By defining a well-known default bean name, 90% of users never need to specify `containerFactory`. But the 10% who need a custom factory (e.g., different `ContainerProperties` for different listeners) can override it. This is the same approach used across Spring: `transactionManager`, `entityManagerFactory`, `kafkaListenerContainerFactory`.
> ─────────────────────────────────────────────────────

---

## 2.5 Comparison with @KafkaListener

| Attribute | @KafkaListener | @ZaloListener | Reason for difference |
|---|---|---|---|
| `id` | Yes | Yes | Same purpose |
| `topics` | Yes (required) | **No** | ZaloBot has no topic concept |
| `topicPattern` | Yes | **No** | No topics |
| `topicPartitions` | Yes | **No** | No partitions |
| `groupId` | Yes | **No** | No consumer groups |
| `containerFactory` | Yes | Yes | Same purpose |
| `concurrency` | Yes | Yes | Same purpose |
| `autoStartup` | Yes | Yes | Same purpose |
| `errorHandler` | Yes | **No** (future) | Use `ContainerProperties.errorHandler` for now |
| `properties` | Yes | **No** | No Kafka consumer properties |
| `batch` | Yes | **No** | No batch semantics |
| `filter` | Yes | **No** (future) | Can be added later |
| `contentTypeConverter` | Yes | **No** | Fixed message format |
| `beanRef` | Yes | **No** | No SpEL support |

---

## 2.6 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `@ZaloListener` | `@KafkaListener` | `annotation/KafkaListener.java` |
| `id()` attribute | `id()` attribute | `KafkaListener.java` (id field) |
| `containerFactory()` attribute | `containerFactory()` attribute | `KafkaListener.java` (containerFactory field) |
| `concurrency()` as String | `concurrency()` as String | `KafkaListener.java` (concurrency field) |
| `autoStartup()` as String | `autoStartup()` as String | `KafkaListener.java` (autoStartup field) |
| No `@Repeatable` | `@Repeatable(KafkaListeners.class)` | `KafkaListener.java`, `KafkaListeners.java` |
| No `@Target(TYPE)` | `@Target({TYPE, METHOD, ANNOTATION_TYPE})` | `KafkaListener.java` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **@ZaloListener** | Method-level annotation that marks a method as an update handler |
| **String-typed attributes** | Enables property placeholder resolution (`${...}`) for externalized config |
| **containerFactory** | Lets users choose which factory creates the container for this listener |
| **Default factory name** | `"zaloListenerContainerFactory"` — convention over configuration |
| **No topics** | Unlike Kafka, ZaloBot has a single update stream per bot |

**Next: Chapter 3** — We build the `UpdateListenerAdapter`, the lowest-level bridge that converts an annotated method into an `UpdateListener` implementation.
