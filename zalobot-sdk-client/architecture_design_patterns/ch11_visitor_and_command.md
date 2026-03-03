# Chapter 11: Visitor & Command — Traversal and Deferred Execution

## What You'll Learn
- How the **Visitor** pattern traverses and modifies bean definition structures
- How the **Command** pattern encapsulates operations as objects for deferred execution
- How Spring's `BeanDefinitionVisitor` resolves placeholders in bean definitions
- When Command is better than direct method calls

---

## 11.1 The Problem: Processing Complex Structures

A bean definition contains nested values — properties, constructor arguments, references to other beans. Processing all of these requires traversing the entire structure:

```
BeanDefinition
├── propertyValues
│   ├── "dataSource" → BeanReference("myDataSource")
│   ├── "timeout" → "${db.timeout}"          ← placeholder!
│   └── "url" → "${db.url}"                  ← placeholder!
├── constructorArgs
│   └── [0] → "${service.name}"              ← placeholder!
└── initMethodName → "init"
```

We need to visit every value, check if it's a placeholder, and resolve it. Writing separate traversal code for each processing task leads to duplication.

## 11.2 Visitor Pattern: Separate Traversal From Processing

The Visitor separates the **structure traversal** from the **operation applied to each element**:

```
  Visitor                    Structure
    │                          │
    │  visit(definition)       │
    │─────────────────────────►│
    │                          │ for each property:
    │  resolveValue(value)     │
    │◄─────────────────────────│
    │  → resolved value        │
    │─────────────────────────►│  replace
    │                          │
    │                          │ for each constructor arg:
    │  resolveValue(value)     │
    │◄─────────────────────────│
    │  → resolved value        │
    │─────────────────────────►│  replace
```

> ★ **Insight** ─────────────────────────────────────
> - **Visitor vs Iterator:** Iterator traverses a flat collection. Visitor traverses a **nested, heterogeneous structure** (like a tree), applying type-specific operations. Bean definitions are exactly this — nested maps and lists with different value types.
> - In classic GoF Visitor, the visited structure has `accept(Visitor)` methods. Spring's `BeanDefinitionVisitor` is a **simplified Visitor** — it controls the traversal itself rather than using double dispatch. This works because the structure (BeanDefinition) is well-known and stable.
> - **When NOT to use Visitor:** When the structure changes frequently (adding new element types requires updating ALL visitors). Bean definitions are stable — the structure hasn't changed significantly in years.
> ─────────────────────────────────────────────────────

## 11.3 Building SimpleBeanDefinitionVisitor

```java
// === SimpleValueResolver.java ===
/**
 * Strategy for resolving values during visitation.
 * Different implementations resolve different things:
 * - PlaceholderResolver: ${...} → actual values
 * - SpEL Resolver: #{...} → expression results
 */
@FunctionalInterface
public interface SimpleValueResolver {
    String resolveStringValue(String value);
}
```

```java
// === PlaceholderResolver.java ===
import java.util.Map;

/**
 * Resolves ${...} placeholders against a property source.
 */
public class PlaceholderResolver implements SimpleValueResolver {

    private final Map<String, String> properties;

    public PlaceholderResolver(Map<String, String> properties) {
        this.properties = properties;
    }

    @Override
    public String resolveStringValue(String value) {
        if (value == null) return null;

        StringBuilder result = new StringBuilder(value);
        int startIdx;
        while ((startIdx = result.indexOf("${")) != -1) {
            int endIdx = result.indexOf("}", startIdx);
            if (endIdx == -1) break;

            String placeholder = result.substring(startIdx + 2, endIdx);

            // Support default values: ${key:default}
            String defaultValue = null;
            int colonIdx = placeholder.indexOf(':');
            if (colonIdx != -1) {
                defaultValue = placeholder.substring(colonIdx + 1);
                placeholder = placeholder.substring(0, colonIdx);
            }

            String resolved = properties.getOrDefault(placeholder, defaultValue);
            if (resolved == null) {
                throw new RuntimeException("Could not resolve placeholder: ${" + placeholder + "}");
            }

            result.replace(startIdx, endIdx + 1, resolved);
        }
        return result.toString();
    }
}
```

```java
// === SimpleBeanDefinitionVisitor.java ===
import java.util.Map;

/**
 * Visitor that traverses a BeanDefinition structure and resolves
 * all string values using a ValueResolver.
 *
 * Simplified version of Spring's BeanDefinitionVisitor.
 */
public class SimpleBeanDefinitionVisitor {

    private final SimpleValueResolver valueResolver;

    public SimpleBeanDefinitionVisitor(SimpleValueResolver valueResolver) {
        this.valueResolver = valueResolver;
    }

    /**
     * Visit a bean definition — traverse and resolve all values.
     */
    public void visitBeanDefinition(SimpleBeanDefinition definition) {
        // Visit property values
        visitPropertyValues(definition.getPropertyValues());

        // Visit constructor arguments
        visitConstructorArgs(definition);

        // Visit init/destroy method names
        if (definition.getInitMethodName() != null) {
            String resolved = resolveValue(definition.getInitMethodName());
            definition.setInitMethodName(resolved);
        }
        if (definition.getDestroyMethodName() != null) {
            String resolved = resolveValue(definition.getDestroyMethodName());
            definition.setDestroyMethodName(resolved);
        }
    }

    private void visitPropertyValues(Map<String, Object> properties) {
        for (Map.Entry<String, Object> entry : properties.entrySet()) {
            Object value = entry.getValue();
            Object resolved = resolveValueIfNecessary(value);
            if (resolved != value) {
                entry.setValue(resolved);
            }
        }
    }

    private void visitConstructorArgs(SimpleBeanDefinition definition) {
        java.util.List<Object> args = definition.getConstructorArgs();
        for (int i = 0; i < args.size(); i++) {
            Object value = args.get(i);
            Object resolved = resolveValueIfNecessary(value);
            if (resolved != value) {
                args.set(i, resolved);
            }
        }
    }

    private Object resolveValueIfNecessary(Object value) {
        if (value instanceof String strValue) {
            return resolveValue(strValue);
        }
        // Could handle List, Map, Set values recursively here
        return value;
    }

    private String resolveValue(String value) {
        return this.valueResolver.resolveStringValue(value);
    }
}
```

## 11.4 Command Pattern: Operations as Objects

The Command pattern encapsulates operations so they can be stored, queued, and executed later:

```java
// Chapter 3's callbacks ARE commands:
TransactionCallback<T> → encapsulates "what to do in a transaction"
StatementCallback<T>   → encapsulates "what to do with a Statement"
```

Beyond callbacks, Spring uses Command for lifecycle operations:

```java
// === SimpleLifecycleCallback.java ===
/**
 * Command pattern: encapsulates a lifecycle action.
 * Can be stored, ordered, and executed at the right time.
 */
@FunctionalInterface
public interface SimpleLifecycleCallback {
    void execute() throws Exception;
}
```

```java
// === SimpleDisposableBean.java ===
/**
 * Command for bean destruction.
 * Container stores these and executes on shutdown.
 */
public interface SimpleDisposableBean {
    void destroy() throws Exception;
}
```

```java
// === SimpleDestructionCallbackRegistry.java ===
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Stores destruction commands and executes them on shutdown.
 * Demonstrates Command pattern: operations stored and deferred.
 */
public class SimpleDestructionCallbackRegistry {

    // Bean name → list of destruction commands
    private final Map<String, List<SimpleLifecycleCallback>> callbacks = new LinkedHashMap<>();

    /**
     * Register a destruction callback for a bean.
     */
    public void registerDestructionCallback(String beanName, SimpleLifecycleCallback callback) {
        callbacks.computeIfAbsent(beanName, k -> new ArrayList<>()).add(callback);
    }

    /**
     * Register callbacks for a DisposableBean.
     */
    public void registerDisposableBean(String beanName, SimpleDisposableBean bean) {
        registerDestructionCallback(beanName, bean::destroy);
    }

    /**
     * Execute all destruction callbacks in reverse registration order.
     * (Last created → first destroyed)
     */
    public void destroyAll() {
        List<String> beanNames = new ArrayList<>(callbacks.keySet());
        // Reverse order — last registered destroyed first
        for (int i = beanNames.size() - 1; i >= 0; i--) {
            String name = beanNames.get(i);
            List<SimpleLifecycleCallback> beanCallbacks = callbacks.get(name);
            for (SimpleLifecycleCallback callback : beanCallbacks) {
                try {
                    callback.execute();
                    System.out.println("[DESTROY] Destroyed: " + name);
                } catch (Exception ex) {
                    System.err.println("[DESTROY] Error destroying " + name + ": " + ex.getMessage());
                }
            }
        }
        callbacks.clear();
    }

    /**
     * Destroy a single bean's callbacks.
     */
    public void destroyBean(String beanName) {
        List<SimpleLifecycleCallback> beanCallbacks = callbacks.remove(beanName);
        if (beanCallbacks != null) {
            for (SimpleLifecycleCallback callback : beanCallbacks) {
                try {
                    callback.execute();
                } catch (Exception ex) {
                    System.err.println("[DESTROY] Error: " + ex.getMessage());
                }
            }
        }
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - **Command vs Callback:** They're structurally identical (a functional interface wrapping an operation), but the INTENT differs. **Callback** means "call me when ready" (immediate). **Command** means "store this for later" (deferred). Spring's destruction callbacks are true Commands — they're stored at bean creation time but executed at shutdown.
> - The **reverse order destruction** is important: if bean B depends on bean A, B was created after A. Destroying B first ensures A is still available during B's cleanup.
> - **When to use Command:** When you need to: (1) queue operations for later, (2) support undo, (3) serialize operations, or (4) parameterize actions. Spring's `Scope.registerDestructionCallback()` is a perfect example — the scope stores cleanup commands for when the scope ends.
> ─────────────────────────────────────────────────────

## 11.5 Usage

```java
// === VisitorCommandDemo.java ===
import java.util.HashMap;
import java.util.Map;

public class VisitorCommandDemo {
    public static void main(String[] args) throws Exception {
        // ══════════════════════════════════════
        // Visitor Pattern: Resolve placeholders
        // ══════════════════════════════════════
        System.out.println("=== Visitor Pattern: Placeholder Resolution ===");

        // Properties source
        Map<String, String> properties = new HashMap<>();
        properties.put("db.url", "jdbc:h2:mem:production");
        properties.put("db.timeout", "30");
        properties.put("service.name", "UserService");

        // Build a bean definition with placeholders
        SimpleBeanDefinition def = SimpleBeanDefinitionBuilder
            .genericBeanDefinition(SimpleDataSource.class)
            .addPropertyValue("url", "${db.url}")
            .addPropertyValue("timeout", "${db.timeout}")
            .addPropertyValue("label", "Service: ${service.name}")
            .addConstructorArgValue("${db.url}")
            .setInitMethodName("init")
            .getBeanDefinition();

        System.out.println("Before: " + def.getPropertyValues());

        // Visit and resolve
        SimpleValueResolver resolver = new PlaceholderResolver(properties);
        SimpleBeanDefinitionVisitor visitor = new SimpleBeanDefinitionVisitor(resolver);
        visitor.visitBeanDefinition(def);

        System.out.println("After:  " + def.getPropertyValues());
        System.out.println("Constructor arg: " + def.getConstructorArgs().get(0));

        // ══════════════════════════════════════
        // Command Pattern: Deferred destruction
        // ══════════════════════════════════════
        System.out.println("\n=== Command Pattern: Deferred Destruction ===");

        SimpleDestructionCallbackRegistry registry = new SimpleDestructionCallbackRegistry();

        // Register destruction commands (stored, not executed yet)
        registry.registerDestructionCallback("dataSource",
            () -> System.out.println("  Closing connection pool"));

        registry.registerDestructionCallback("sessionFactory",
            () -> System.out.println("  Closing session factory"));

        registry.registerDestructionCallback("cacheManager",
            () -> System.out.println("  Flushing and closing cache"));

        // Commands stored for later...
        System.out.println("Commands registered. Simulating shutdown...");

        // Execute all destruction commands (reverse order)
        registry.destroyAll();
    }
}
```

**Output:**
```
=== Visitor Pattern: Placeholder Resolution ===
Before: {url=${db.url}, timeout=${db.timeout}, label=Service: ${service.name}}
After:  {url=jdbc:h2:mem:production, timeout=30, label=Service: UserService}
Constructor arg: jdbc:h2:mem:production

=== Command Pattern: Deferred Destruction ===
Commands registered. Simulating shutdown...
[DESTROY] Destroyed: cacheManager
  Flushing and closing cache
[DESTROY] Destroyed: sessionFactory
  Closing session factory
[DESTROY] Destroyed: dataSource
  Closing connection pool
```

## 11.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleBeanDefinitionVisitor` | `BeanDefinitionVisitor` | `spring-beans/.../config/BeanDefinitionVisitor.java` |
| `SimpleValueResolver` | `StringValueResolver` | `spring-beans` |
| `PlaceholderResolver` | `PropertyPlaceholderConfigurer` / `PropertySourcesPlaceholderConfigurer` | `spring-context` |
| `SimpleLifecycleCallback` | `Runnable` (used as destruction callback) | |
| `SimpleDisposableBean` | `DisposableBean` | `spring-beans` |
| `SimpleDestructionCallbackRegistry` | `DefaultSingletonBeanRegistry.destroySingletons()` | `spring-beans` |
| `Scope.registerDestructionCallback()` | `Scope.registerDestructionCallback(String, Runnable)` | `:124` |
| `TransactionCallback<T>` | `TransactionCallback<T>` | `spring-tx` |
| `StatementCallback<T>` | `StatementCallback<T>` | `spring-jdbc` |

**What real Spring adds:**
- `BeanDefinitionVisitor` handles `TypedStringValue`, `ManagedList`, `ManagedMap`, `BeanReference`, nested bean definitions
- `PlaceholderConfigurerSupport` implements both BeanFactoryPostProcessor and Visitor
- `DisposableBeanAdapter` adapts `@PreDestroy`, `DisposableBean`, and custom destroy-methods into a single callback
- `DefaultSingletonBeanRegistry.destroySingleton()` handles dependent bean destruction ordering
- `ContextClosedEvent` published before destruction begins

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Visitor** | Traverse and process a nested structure without modifying its classes |
| **Value Resolver** | Strategy plugged into the Visitor for actual resolution logic |
| **Command** | Encapsulate operation as object for deferred execution |
| **Destruction Callbacks** | Commands stored at creation time, executed at shutdown |
| **Reverse-Order Destruction** | Last-created beans destroyed first (LIFO) |

**Next: Chapter 12** — We integrate ALL patterns into a single working system to see how they compose.
