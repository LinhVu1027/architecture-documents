# Chapter 3: The Message Adapter — Bridging Methods to UpdateListener

## What You'll Learn
- Why we need an adapter between annotated methods and `UpdateListener`
- How `UpdateListenerAdapter` uses reflection to invoke target methods
- How parameter resolution works for different method signatures
- The Adapter pattern and why it's the right choice here

---

## 3.1 The Problem: Methods Aren't UpdateListeners

The existing `ZaloUpdateListenerContainer` expects an `UpdateListener`:

```java
// UpdateListener.java (existing)
@FunctionalInterface
public interface UpdateListener {
    void onUpdate(GetUpdatesResult update);
}
```

But the user writes methods like:

```java
@ZaloListener
public void handleUpdate(GetUpdatesResult update) { ... }

@ZaloListener
public void handleMessage(GetUpdatesResult.Message message) { ... }

@ZaloListener
public void handleText(String text) { ... }
```

We need something that:
1. Implements `UpdateListener` (so the container can call it)
2. Invokes the user's method (with the right parameter types)

This is the classic **Adapter pattern** — adapting one interface (`UpdateListener`) to another (the user's method signature).

---

## 3.2 UpdateListenerAdapter Implementation

```java
package dev.linhvu.zalobot.boot.listener.adapter;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

import dev.linhvu.zalobot.core.model.GetUpdatesResult;
import dev.linhvu.zalobot.listener.UpdateListener;

/**
 * Adapter that implements {@link UpdateListener} by delegating to a
 * reflective method invocation on a target bean.
 *
 * <p>Supports flexible method signatures:
 * <ul>
 *   <li>{@code void handle(GetUpdatesResult update)} — full update</li>
 *   <li>{@code void handle(GetUpdatesResult.Message message)} — message only</li>
 *   <li>{@code void handle(String text)} — message text only</li>
 * </ul>
 *
 * @author Linh Vu
 * @since 0.1.0
 */
public class UpdateListenerAdapter implements UpdateListener {

    private final Object bean;
    private final Method method;

    public UpdateListenerAdapter(Object bean, Method method) {
        if (bean == null) {
            throw new IllegalArgumentException("'bean' cannot be null");
        }
        if (method == null) {
            throw new IllegalArgumentException("'method' cannot be null");
        }
        this.bean = bean;
        this.method = method;
        this.method.setAccessible(true);
    }

    @Override
    public void onUpdate(GetUpdatesResult update) {
        Object[] args = resolveArguments(update);
        try {
            this.method.invoke(this.bean, args);
        }
        catch (InvocationTargetException ex) {
            Throwable target = ex.getTargetException();
            if (target instanceof RuntimeException rte) {
                throw rte;
            }
            throw new UpdateListenerExecutionException(
                    "Listener method '" + this.method.getName() + "' threw exception", target);
        }
        catch (IllegalAccessException ex) {
            throw new UpdateListenerExecutionException(
                    "Could not access listener method '" + this.method.getName() + "'", ex);
        }
    }

    /**
     * Resolves the arguments to pass to the target method based on
     * the method's parameter types.
     */
    private Object[] resolveArguments(GetUpdatesResult update) {
        Class<?>[] paramTypes = this.method.getParameterTypes();
        Object[] args = new Object[paramTypes.length];

        for (int i = 0; i < paramTypes.length; i++) {
            args[i] = resolveArgument(paramTypes[i], update);
        }
        return args;
    }

    private Object resolveArgument(Class<?> paramType, GetUpdatesResult update) {
        if (GetUpdatesResult.class.isAssignableFrom(paramType)) {
            return update;
        }
        if (GetUpdatesResult.Message.class.isAssignableFrom(paramType)) {
            return update != null ? update.message() : null;
        }
        if (String.class.isAssignableFrom(paramType)) {
            return (update != null && update.message() != null)
                    ? update.message().text() : null;
        }
        throw new IllegalStateException(
                "Unsupported parameter type '" + paramType.getName()
                + "' in listener method '" + this.method.getName()
                + "'. Supported types: GetUpdatesResult, GetUpdatesResult.Message, String");
    }

    public Object getBean() {
        return this.bean;
    }

    public Method getMethod() {
        return this.method;
    }
}
```

And the custom exception:

```java
package dev.linhvu.zalobot.boot.listener.adapter;

/**
 * Exception thrown when a listener method invocation fails.
 */
public class UpdateListenerExecutionException extends RuntimeException {

    public UpdateListenerExecutionException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## 3.3 How Parameter Resolution Works

The adapter inspects the method's parameter types at invocation time and extracts the appropriate data from `GetUpdatesResult`:

```
GetUpdatesResult update
├── message(): Message
│   ├── text(): String
│   ├── messageId(): String
│   ├── chat(): Chat
│   └── from(): From
└── eventName(): String

Method parameter type        →  Resolved argument
────────────────────────────────────────────────────
GetUpdatesResult             →  update (the full object)
GetUpdatesResult.Message     →  update.message()
String                       →  update.message().text()
```

**Why type-based, not annotation-based?**

Spring Kafka uses `@Payload`, `@Header`, and Spring Messaging's `HandlerMethodArgumentResolver` infrastructure for parameter resolution. That's powerful but complex — it requires `spring-messaging` dependency, `InvocableHandlerMethod`, and a `MessageHandlerMethodFactory`.

For ZaloBot, we have only 3 parameter types. Type-based resolution is simpler and sufficient:

```
Spring Kafka:  @Payload String message, @Header("kafka_offset") long offset
                    ↓                         ↓
              PayloadArgumentResolver    HeaderArgumentResolver
                    ↓                         ↓
              HandlerMethodArgumentResolver infrastructure

ZaloBot:       GetUpdatesResult update
                    ↓
              if (paramType == GetUpdatesResult.class) return update;
```

> ★ **Insight** ─────────────────────────────────────
> - **YAGNI (You Ain't Gonna Need It):** Spring Kafka's `InvocableHandlerMethod` infrastructure supports dozens of parameter types, SpEL expressions, validation, and async replies. But ZaloBot has 3 parameter types and no headers. Using the full Spring Messaging infrastructure would add 2+ dependencies and hundreds of lines of configuration for zero benefit. Start simple; add complexity when a real need arises.
> - **The Adapter pattern is the linchpin.** Without this class, every other component in the architecture would need to know about reflection, method signatures, and parameter resolution. By isolating this concern in one class, the rest of the system (endpoint, factory, registry) only deals with `UpdateListener` — a clean interface.
> ─────────────────────────────────────────────────────

---

## 3.4 Error Handling in the Adapter

The adapter wraps reflection exceptions into meaningful error types:

```
Method.invoke(bean, args)
    ├── throws InvocationTargetException
    │       └── getTargetException()
    │            ├── RuntimeException → rethrown as-is
    │            └── checked Exception → wrapped in UpdateListenerExecutionException
    └── throws IllegalAccessException
            └── wrapped in UpdateListenerExecutionException
```

The container's `ErrorHandler` then handles these exceptions:

```
Container (ListenerConsumer)
    ├── calls listener.onUpdate(update)
    │       └── UpdateListenerAdapter.onUpdate()
    │               └── may throw RuntimeException
    ├── catches Exception
    └── calls errorHandler.handleError(exception, container)
```

> ★ **Insight** ─────────────────────────────────────
> - **Unwrap `InvocationTargetException` before rethrowing.** A common mistake with reflection-based invocation is rethrowing the `InvocationTargetException` directly. The REAL exception is inside `getTargetException()`. Spring Kafka's `MessagingMessageListenerAdapter` (line ~330) does the same unwrapping. If you skip this, stack traces become confusing because the actual exception is buried inside a wrapper.
> ─────────────────────────────────────────────────────

---

## 3.5 Connection to Real Spring Kafka

| Simplified (ZaloBot) | Real Spring Kafka | File Reference |
|---|---|---|
| `UpdateListenerAdapter` | `RecordMessagingMessageListenerAdapter` | `listener/adapter/RecordMessagingMessageListenerAdapter.java` |
| `onUpdate(GetUpdatesResult)` | `onMessage(ConsumerRecord, Acknowledgment, Consumer)` | `RecordMessagingMessageListenerAdapter.java` |
| Type-based parameter resolution | `InvocableHandlerMethod` + argument resolvers | `adapter/MessagingMessageListenerAdapter.java` |
| `resolveArguments()` | `HandlerAdapter.invoke(Message)` → `InvocableHandlerMethod.invoke()` | `MessagingMessageListenerAdapter.java` |
| `UpdateListenerExecutionException` | `ListenerExecutionFailedException` | `listener/ListenerExecutionFailedException.java` |
| Direct `Method.invoke()` | Spring Messaging `InvocableHandlerMethod` | `messaging/handler/invocation/InvocableHandlerMethod.java` (in spring-messaging) |

**Key simplification:** Spring Kafka's adapter extends `MessagingMessageListenerAdapter` which implements both `MessageListener` and `AcknowledgingConsumerAwareMessageListener`. It converts `ConsumerRecord` to Spring `Message`, runs it through `HandlerMethodArgumentResolver` chain, then invokes via `InvocableHandlerMethod`. Our adapter directly invokes `Method.invoke()` with type-matched arguments — same result, 90% less code.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **UpdateListenerAdapter** | Implements `UpdateListener` by invoking an annotated method via reflection |
| **Type-based resolution** | Matches method parameter types to extract the right data from `GetUpdatesResult` |
| **Adapter pattern** | Bridges the gap between `UpdateListener.onUpdate()` and user's method signature |
| **Exception unwrapping** | Extracts the real exception from `InvocationTargetException.getTargetException()` |
| **Supported parameter types** | `GetUpdatesResult`, `GetUpdatesResult.Message`, `String` |

**Next: Chapter 4** — We build the `ZaloListenerEndpoint` abstraction that bundles a method reference, its configuration, and the logic to wire everything into a container.
