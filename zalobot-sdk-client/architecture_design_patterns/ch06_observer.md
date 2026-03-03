# Chapter 6: Observer — Decoupled Event Communication

## What You'll Learn
- How the **Observer** (Publish-Subscribe) pattern decouples event producers from consumers
- How Spring's `ApplicationEvent` system enables loose component coupling
- How type-safe generics filter events to the right listeners
- When to use synchronous vs asynchronous event delivery

---

## 6.1 The Problem: Direct Method Calls Create Tight Coupling

When `OrderService` needs to notify `EmailService`, `InventoryService`, and `AnalyticsService` after an order:

```java
// TIGHT COUPLING: OrderService knows about every consumer
public class OrderService {
    private final EmailService emailService;       // dependency
    private final InventoryService inventoryService; // dependency
    private final AnalyticsService analyticsService; // dependency

    public void placeOrder(Order order) {
        // Business logic
        saveOrder(order);

        // Notify everyone — OrderService shouldn't care about these!
        emailService.sendConfirmation(order);
        inventoryService.decrementStock(order);
        analyticsService.trackOrder(order);
    }
}
```

Adding a new consumer (e.g., `LoyaltyPointsService`) requires **modifying OrderService**. This violates the Open-Closed Principle.

## 6.2 Observer Pattern: Publish Without Knowing Who Listens

```
  Publisher                 Event Bus              Listeners
     │                        │                     │
     │  publish(OrderEvent)   │                     │
     │───────────────────────►│                     │
     │                        │  onEvent(e)         │
     │                        │────────────────────►│ EmailListener
     │                        │  onEvent(e)         │
     │                        │────────────────────►│ InventoryListener
     │                        │  onEvent(e)         │
     │                        │────────────────────►│ AnalyticsListener
     │                        │                     │
     │  Publisher doesn't know │  New listeners can  │
     │  who's listening!       │  be added without   │
     │                        │  changing publisher  │
```

> ★ **Insight** ─────────────────────────────────────
> - Observer vs Mediator: Both decouple components, but **Observer** is one-to-many (publish to all subscribers), while **Mediator** is many-to-many with intelligent routing. Spring's event system is Observer; Spring Integration is closer to Mediator.
> - **When to use Observer:** For notifications where the publisher genuinely doesn't care who responds. If the publisher needs a result from the listener, Observer is the wrong pattern — use direct calls or Strategy instead.
> - Spring's Observer is **synchronous by default**. This means listeners execute in the publisher's thread and exceptions propagate back. Set a `TaskExecutor` on the multicaster for async delivery.
> ─────────────────────────────────────────────────────

## 6.3 Building SimpleEventBus

### Step 1: The Event Base Class

```java
// === SimpleApplicationEvent.java ===
/**
 * Base class for all application events.
 * Carries a source object and timestamp.
 */
public abstract class SimpleApplicationEvent {
    private final Object source;
    private final long timestamp;

    public SimpleApplicationEvent(Object source) {
        this.source = source;
        this.timestamp = System.currentTimeMillis();
    }

    public Object getSource() { return source; }
    public long getTimestamp() { return timestamp; }
}
```

### Step 2: The Listener Interface

```java
// === SimpleApplicationListener.java ===
/**
 * Type-safe event listener.
 * Generic parameter E filters which events this listener receives.
 */
@FunctionalInterface
public interface SimpleApplicationListener<E extends SimpleApplicationEvent> {
    void onApplicationEvent(E event);
}
```

### Step 3: The Event Publisher Interface

```java
// === SimpleEventPublisher.java ===
/**
 * Interface for publishing events.
 * This is what beans use — they don't know about the multicaster.
 */
public interface SimpleEventPublisher {
    void publishEvent(SimpleApplicationEvent event);
}
```

### Step 4: The Event Multicaster (the actual Observer registry)

```java
// === SimpleEventMulticaster.java ===
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.Executor;

/**
 * Manages listener registration and event dispatch.
 * Simplified version of SimpleApplicationEventMulticaster.
 */
public class SimpleEventMulticaster implements SimpleEventPublisher {

    private final List<ListenerEntry> listeners = new CopyOnWriteArrayList<>();
    private Executor taskExecutor;  // null = synchronous

    /**
     * Register a listener. The generic type parameter determines
     * which events it receives.
     */
    public void addListener(SimpleApplicationListener<?> listener) {
        Class<?> eventType = resolveEventType(listener);
        listeners.add(new ListenerEntry(listener, eventType));
    }

    /**
     * Remove a listener.
     */
    public void removeListener(SimpleApplicationListener<?> listener) {
        listeners.removeIf(entry -> entry.listener == listener);
    }

    /**
     * Set executor for async event delivery.
     * null (default) means synchronous delivery.
     */
    public void setTaskExecutor(Executor taskExecutor) {
        this.taskExecutor = taskExecutor;
    }

    /**
     * Publish event to all matching listeners.
     */
    @Override
    @SuppressWarnings("unchecked")
    public void publishEvent(SimpleApplicationEvent event) {
        for (ListenerEntry entry : listeners) {
            // Type-safe filtering: only deliver if listener's type matches
            if (entry.eventType.isAssignableFrom(event.getClass())) {
                SimpleApplicationListener<SimpleApplicationEvent> typedListener =
                    (SimpleApplicationListener<SimpleApplicationEvent>) entry.listener;

                if (taskExecutor != null) {
                    // Async delivery
                    taskExecutor.execute(() -> invokeListener(typedListener, event));
                } else {
                    // Synchronous delivery
                    invokeListener(typedListener, event);
                }
            }
        }
    }

    private void invokeListener(SimpleApplicationListener<SimpleApplicationEvent> listener,
                                 SimpleApplicationEvent event) {
        try {
            listener.onApplicationEvent(event);
        } catch (Exception ex) {
            System.err.println("Error in event listener: " + ex.getMessage());
        }
    }

    /**
     * Resolve the event type from the listener's generic parameter.
     * e.g., SimpleApplicationListener<OrderCreatedEvent> → OrderCreatedEvent.class
     */
    private Class<?> resolveEventType(SimpleApplicationListener<?> listener) {
        for (Type iface : listener.getClass().getGenericInterfaces()) {
            if (iface instanceof ParameterizedType pt) {
                if (pt.getRawType() == SimpleApplicationListener.class) {
                    Type arg = pt.getActualTypeArguments()[0];
                    if (arg instanceof Class<?> cls) {
                        return cls;
                    }
                }
            }
        }
        // Fallback: accept all events (for lambdas where generics are erased)
        return SimpleApplicationEvent.class;
    }

    // Internal record to pair a listener with its event type
    private record ListenerEntry(SimpleApplicationListener<?> listener, Class<?> eventType) {}
}
```

> ★ **Insight** ─────────────────────────────────────
> - `CopyOnWriteArrayList` is crucial here — it's thread-safe for the common case where reads (event publishing) vastly outnumber writes (listener registration). Adding/removing listeners copies the entire list, but iterating is lock-free.
> - Type resolution via reflection is the weakest point. Lambda listeners lose their generic type information due to Java's type erasure. Real Spring handles this with `ResolvableType` which can inspect generic signatures more thoroughly than raw reflection.
> - The **synchronous default** is deliberate. It means events participate in the caller's transaction — if the listener throws, the transaction rolls back. Async listeners (`supportsAsyncExecution()` in Spring 6.1) opt out of this guarantee.
> ─────────────────────────────────────────────────────

## 6.4 Usage

### Custom Events

```java
// === OrderCreatedEvent.java ===
public class OrderCreatedEvent extends SimpleApplicationEvent {
    private final String orderId;
    private final double amount;

    public OrderCreatedEvent(Object source, String orderId, double amount) {
        super(source);
        this.orderId = orderId;
        this.amount = amount;
    }

    public String getOrderId() { return orderId; }
    public double getAmount() { return amount; }
}

// === UserRegisteredEvent.java ===
public class UserRegisteredEvent extends SimpleApplicationEvent {
    private final String username;

    public UserRegisteredEvent(Object source, String username) {
        super(source);
        this.username = username;
    }

    public String getUsername() { return username; }
}
```

### Typed Listeners

```java
// === EmailListener.java ===
public class EmailListener implements SimpleApplicationListener<OrderCreatedEvent> {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        System.out.println("[EMAIL] Sending confirmation for order: " + event.getOrderId());
    }
}

// === AnalyticsListener.java ===
public class AnalyticsListener implements SimpleApplicationListener<OrderCreatedEvent> {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        System.out.println("[ANALYTICS] Tracking order: " + event.getOrderId()
            + " amount: $" + event.getAmount());
    }
}

// === WelcomeListener.java ===
public class WelcomeListener implements SimpleApplicationListener<UserRegisteredEvent> {
    @Override
    public void onApplicationEvent(UserRegisteredEvent event) {
        System.out.println("[WELCOME] Hello, " + event.getUsername() + "!");
    }
}
```

### Bringing It Together

```java
// === ObserverDemo.java ===
public class ObserverDemo {
    public static void main(String[] args) {
        SimpleEventMulticaster multicaster = new SimpleEventMulticaster();

        // Register listeners (they don't know about each other)
        multicaster.addListener(new EmailListener());
        multicaster.addListener(new AnalyticsListener());
        multicaster.addListener(new WelcomeListener());

        // Publish events (publisher doesn't know who's listening)
        System.out.println("--- Order placed ---");
        multicaster.publishEvent(new OrderCreatedEvent("OrderService", "ORD-001", 99.99));

        System.out.println("\n--- User registered ---");
        multicaster.publishEvent(new UserRegisteredEvent("AuthService", "alice"));
    }
}
```

**Output:**
```
--- Order placed ---
[EMAIL] Sending confirmation for order: ORD-001
[ANALYTICS] Tracking order: ORD-001 amount: $99.99

--- User registered ---
[WELCOME] Hello, alice!
```

Note: `WelcomeListener` did NOT receive the `OrderCreatedEvent` — type-safe filtering works!

## 6.5 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleApplicationEvent` | `ApplicationEvent` | `:31-77` |
| `SimpleApplicationListener<E>` | `ApplicationListener<E>` | `:42-76` |
| `SimpleEventPublisher` | `ApplicationEventPublisher` | Interface in `spring-context` |
| `SimpleEventMulticaster` | `SimpleApplicationEventMulticaster` | `:75-202` |
| `multicaster.publishEvent()` | `SimpleApplicationEventMulticaster.multicastEvent()` | `:132-154` |
| `invokeListener()` | `SimpleApplicationEventMulticaster.invokeListener()` | `:162-175` |
| Type resolution | `ResolvableType.forInstance(event)` | Used in `multicastEvent()` |
| `setTaskExecutor()` | `SimpleApplicationEventMulticaster.setTaskExecutor()` | `:75` |

**What real Spring adds:**
- `@EventListener` annotation for declarative listeners (no interface needed)
- `PayloadApplicationEvent<T>` for publishing arbitrary objects as events
- `ApplicationListener.forPayload(Consumer)` factory method (`:69-71`)
- `supportsAsyncExecution()` per-listener async control (since 6.1)
- `@TransactionalEventListener` for transaction-phase-aware listeners
- `SmartApplicationListener` for ordering and event type filtering
- `ErrorHandler` strategy for handling listener exceptions (`:75`)

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Observer Pattern** | Publisher emits events; subscribers react without coupling |
| **Event Multicaster** | Registry that matches events to listeners by type and dispatches |
| **Type-Safe Filtering** | Generic parameter `E` ensures listener only receives matching events |
| **Synchronous Default** | Listeners run in publisher's thread; participate in transactions |
| **CopyOnWriteArrayList** | Thread-safe list optimized for frequent reads, rare writes |

**Next: Chapter 7** — We build `SimpleBeanPostProcessor` using the **Decorator** pattern to enhance beans during creation.
