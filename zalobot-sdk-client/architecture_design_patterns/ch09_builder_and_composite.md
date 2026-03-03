# Chapter 9: Builder & Composite — Complex Construction and Aggregation

## What You'll Learn
- How the **Builder** pattern handles complex object construction with many optional parameters
- How the **Composite** pattern treats collections of processors as single processors
- How Spring's `BeanDefinitionBuilder` creates bean definitions fluently
- When to use Composite vs simple iteration

---

## 9.1 The Problem: Too Many Constructor Parameters

A bean definition has many optional properties:

```java
// Telescoping constructor anti-pattern
BeanDefinition def = new BeanDefinition(
    "userService",          // name
    UserServiceImpl.class,  // beanClass
    "singleton",            // scope
    true,                   // lazyInit
    "init",                 // initMethodName
    "cleanup",              // destroyMethodName
    null,                   // factoryMethodName
    null,                   // factoryBeanName
    // ... 10 more parameters
);
```

Most callers only need 2-3 of these. The constructor signature is unreadable and fragile.

## 9.2 Builder Pattern: Step-by-Step Construction

```java
// With Builder: read like a sentence
BeanDefinition def = BeanDefinitionBuilder
    .genericBeanDefinition(UserServiceImpl.class)
    .setScope("singleton")
    .setLazyInit(true)
    .setInitMethodName("init")
    .addPropertyValue("timeout", 30)
    .addConstructorArgValue("arg1")
    .getBeanDefinition();
```

> ★ **Insight** ─────────────────────────────────────
> - Builder is the right pattern when: (1) the object has many optional parameters, (2) the object is immutable once built, and (3) the construction process has validation rules. BeanDefinition has 20+ optional fields — a perfect Builder candidate.
> - Spring's `BeanDefinitionBuilder` uses **static factory methods** (`genericBeanDefinition()`, `rootBeanDefinition()`) to create different builder types. This is a common variation — the builder itself is created via a factory method.
> - **When NOT to use Builder:** For simple objects with 2-3 required fields and no optional fields. A plain constructor or a factory method is clearer.
> ─────────────────────────────────────────────────────

## 9.3 Building SimpleBeanDefinition and Builder

### Step 1: The Definition (what we're building)

```java
// === SimpleBeanDefinition.java ===
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Supplier;

/**
 * Describes how to create a bean — the "recipe".
 * Has many optional fields → perfect for Builder pattern.
 */
public class SimpleBeanDefinition {
    private Class<?> beanClass;
    private String scope = "singleton";           // default
    private boolean lazyInit = false;             // default
    private String initMethodName;
    private String destroyMethodName;
    private Supplier<?> instanceSupplier;
    private final List<Object> constructorArgs = new ArrayList<>();
    private final Map<String, Object> propertyValues = new LinkedHashMap<>();

    // -- Getters --
    public Class<?> getBeanClass() { return beanClass; }
    public String getScope() { return scope; }
    public boolean isLazyInit() { return lazyInit; }
    public String getInitMethodName() { return initMethodName; }
    public String getDestroyMethodName() { return destroyMethodName; }
    public Supplier<?> getInstanceSupplier() { return instanceSupplier; }
    public List<Object> getConstructorArgs() { return constructorArgs; }
    public Map<String, Object> getPropertyValues() { return propertyValues; }

    // -- Package-private setters (only Builder should use these) --
    void setBeanClass(Class<?> beanClass) { this.beanClass = beanClass; }
    void setScope(String scope) { this.scope = scope; }
    void setLazyInit(boolean lazyInit) { this.lazyInit = lazyInit; }
    void setInitMethodName(String name) { this.initMethodName = name; }
    void setDestroyMethodName(String name) { this.destroyMethodName = name; }
    void setInstanceSupplier(Supplier<?> supplier) { this.instanceSupplier = supplier; }

    @Override
    public String toString() {
        return "BeanDef{class=" + (beanClass != null ? beanClass.getSimpleName() : "null")
            + ", scope=" + scope
            + ", lazy=" + lazyInit
            + ", props=" + propertyValues.keySet()
            + ", args=" + constructorArgs.size()
            + "}";
    }
}
```

### Step 2: The Builder

```java
// === SimpleBeanDefinitionBuilder.java ===
import java.util.function.Supplier;

/**
 * Builder pattern for constructing SimpleBeanDefinition instances.
 * Fluent API with method chaining.
 */
public class SimpleBeanDefinitionBuilder {

    private final SimpleBeanDefinition definition;

    // Private constructor — use factory methods
    private SimpleBeanDefinitionBuilder(SimpleBeanDefinition definition) {
        this.definition = definition;
    }

    // ── Static factory methods (entry points) ──

    public static SimpleBeanDefinitionBuilder genericBeanDefinition() {
        return new SimpleBeanDefinitionBuilder(new SimpleBeanDefinition());
    }

    public static SimpleBeanDefinitionBuilder genericBeanDefinition(Class<?> beanClass) {
        SimpleBeanDefinitionBuilder builder = new SimpleBeanDefinitionBuilder(new SimpleBeanDefinition());
        builder.definition.setBeanClass(beanClass);
        return builder;
    }

    public static <T> SimpleBeanDefinitionBuilder genericBeanDefinition(
            Class<T> beanClass, Supplier<T> instanceSupplier) {
        SimpleBeanDefinitionBuilder builder = genericBeanDefinition(beanClass);
        builder.definition.setInstanceSupplier(instanceSupplier);
        return builder;
    }

    // ── Fluent setters (each returns 'this' for chaining) ──

    public SimpleBeanDefinitionBuilder setScope(String scope) {
        this.definition.setScope(scope);
        return this;
    }

    public SimpleBeanDefinitionBuilder setLazyInit(boolean lazyInit) {
        this.definition.setLazyInit(lazyInit);
        return this;
    }

    public SimpleBeanDefinitionBuilder setInitMethodName(String methodName) {
        this.definition.setInitMethodName(methodName);
        return this;
    }

    public SimpleBeanDefinitionBuilder setDestroyMethodName(String methodName) {
        this.definition.setDestroyMethodName(methodName);
        return this;
    }

    public SimpleBeanDefinitionBuilder addConstructorArgValue(Object value) {
        this.definition.getConstructorArgs().add(value);
        return this;
    }

    public SimpleBeanDefinitionBuilder addPropertyValue(String name, Object value) {
        this.definition.getPropertyValues().put(name, value);
        return this;
    }

    // ── Terminal operation: build and return ──

    public SimpleBeanDefinition getBeanDefinition() {
        // Validate
        if (definition.getBeanClass() == null && definition.getInstanceSupplier() == null) {
            throw new IllegalStateException("Either beanClass or instanceSupplier must be set");
        }
        return this.definition;
    }
}
```

## 9.4 Composite Pattern: Treat Many as One

The Composite pattern lets you treat a collection of processors as a single processor:

```
   Client
     │
     │  resolve(param)
     ▼
  ┌──────────────────────────────────────┐
  │   ArgumentResolverComposite          │  ◄── looks like ONE resolver
  │                                      │
  │   ┌─────────┐ ┌─────────┐ ┌───────┐│
  │   │Resolver1│ │Resolver2│ │Resolver3│
  │   └─────────┘ └─────────┘ └───────┘│
  │                                      │
  │   Iterates until one supports()      │
  │   Then delegates resolve() to it     │
  └──────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - Composite vs simple `List.forEach()`: Both iterate, but Composite **implements the same interface** as its children. This means a Composite can be nested inside another Composite, and clients treat single items and collections identically.
> - Spring's `HandlerMethodArgumentResolverComposite` is a "first match" composite — it finds the first resolver that `supportsParameter()` and delegates to it. This is different from "aggregate" composites (like `CompositeDatabasePopulator`) where ALL children execute.
> - **When to use Composite:** When clients shouldn't care whether they're talking to one object or many. Classic examples: file systems (file vs directory), UI components (widget vs container), expression trees (literal vs compound).
> ─────────────────────────────────────────────────────

### Building SimpleArgumentResolver (Composite)

```java
// === SimpleArgumentResolver.java ===
/**
 * Strategy for resolving method arguments from a request.
 */
public interface SimpleArgumentResolver {
    boolean supports(String parameterType);
    Object resolve(String parameterType, SimpleHttpRequest request);
}
```

```java
// === PathVariableResolver.java ===
public class PathVariableResolver implements SimpleArgumentResolver {
    @Override
    public boolean supports(String parameterType) {
        return "pathVariable".equals(parameterType);
    }

    @Override
    public Object resolve(String parameterType, SimpleHttpRequest request) {
        return request.getParam("id");  // simplified: always resolves "id"
    }
}
```

```java
// === RequestParamResolver.java ===
public class RequestParamResolver implements SimpleArgumentResolver {
    @Override
    public boolean supports(String parameterType) {
        return "requestParam".equals(parameterType);
    }

    @Override
    public Object resolve(String parameterType, SimpleHttpRequest request) {
        return request.getParam("name");  // simplified
    }
}
```

```java
// === SimpleArgumentResolverComposite.java ===
import java.util.ArrayList;
import java.util.List;

/**
 * Composite pattern: treats multiple resolvers as ONE resolver.
 * Implements the SAME interface as its children.
 */
public class SimpleArgumentResolverComposite implements SimpleArgumentResolver {

    private final List<SimpleArgumentResolver> resolvers = new ArrayList<>();

    public SimpleArgumentResolverComposite addResolver(SimpleArgumentResolver resolver) {
        this.resolvers.add(resolver);
        return this;
    }

    @Override
    public boolean supports(String parameterType) {
        for (SimpleArgumentResolver resolver : resolvers) {
            if (resolver.supports(parameterType)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public Object resolve(String parameterType, SimpleHttpRequest request) {
        for (SimpleArgumentResolver resolver : resolvers) {
            if (resolver.supports(parameterType)) {
                return resolver.resolve(parameterType, request);
            }
        }
        throw new RuntimeException("No resolver for: " + parameterType);
    }
}
```

## 9.5 Usage

```java
// === BuilderCompositeDemo.java ===
public class BuilderCompositeDemo {
    public static void main(String[] args) {
        // ── Builder Pattern Demo ──
        System.out.println("=== Builder Pattern ===");

        SimpleBeanDefinition def1 = SimpleBeanDefinitionBuilder
            .genericBeanDefinition(UserServiceImpl.class)
            .setScope("singleton")
            .setLazyInit(true)
            .setInitMethodName("init")
            .addPropertyValue("timeout", 30)
            .addPropertyValue("maxRetries", 3)
            .addConstructorArgValue("primary-db")
            .getBeanDefinition();

        System.out.println("Built: " + def1);

        // Minimal definition — most fields use defaults
        SimpleBeanDefinition def2 = SimpleBeanDefinitionBuilder
            .genericBeanDefinition(SimpleDataSource.class)
            .getBeanDefinition();

        System.out.println("Minimal: " + def2);

        // With supplier
        SimpleBeanDefinition def3 = SimpleBeanDefinitionBuilder
            .genericBeanDefinition(SimpleDataSource.class,
                () -> new SimpleDataSource("jdbc:h2:mem:test"))
            .setScope("prototype")
            .getBeanDefinition();

        System.out.println("With supplier: " + def3);

        // ── Composite Pattern Demo ──
        System.out.println("\n=== Composite Pattern ===");

        // Build composite resolver (looks like ONE resolver)
        SimpleArgumentResolver composite = new SimpleArgumentResolverComposite()
            .addResolver(new PathVariableResolver())
            .addResolver(new RequestParamResolver());

        // Client doesn't know it's talking to a composite
        SimpleHttpRequest request = new SimpleHttpRequest("GET", "/users/42");
        request.setParam("id", "42");
        request.setParam("name", "Alice");

        System.out.println("Path variable: " + composite.resolve("pathVariable", request));
        System.out.println("Request param: " + composite.resolve("requestParam", request));
        System.out.println("Supports 'header': " + composite.supports("header"));
    }
}
```

**Output:**
```
=== Builder Pattern ===
Built: BeanDef{class=UserServiceImpl, scope=singleton, lazy=true, props=[timeout, maxRetries], args=1}
Minimal: BeanDef{class=SimpleDataSource, scope=singleton, lazy=false, props=[], args=0}
With supplier: BeanDef{class=SimpleDataSource, scope=prototype, lazy=false, props=[], args=0}

=== Composite Pattern ===
Path variable: 42
Request param: Alice
Supports 'header': false
```

## 9.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleBeanDefinitionBuilder` | `BeanDefinitionBuilder` | `:46-384` |
| `genericBeanDefinition()` | `BeanDefinitionBuilder.genericBeanDefinition()` | `:46-81` |
| `addPropertyValue()` | `BeanDefinitionBuilder.addPropertyValue()` | `:244-247` |
| `addConstructorArgValue()` | `BeanDefinitionBuilder.addConstructorArgValue()` | `:225-229` |
| `getBeanDefinition()` | `BeanDefinitionBuilder.getBeanDefinition()` | `:185` |
| `SimpleBeanDefinition` | `GenericBeanDefinition` / `AbstractBeanDefinition` | `spring-beans` |
| `SimpleArgumentResolverComposite` | `HandlerMethodArgumentResolverComposite` | `spring-web` |
| `SimpleArgumentResolver` | `HandlerMethodArgumentResolver` | `spring-web` |
| `PathVariableResolver` | `PathVariableMethodArgumentResolver` | `spring-webmvc` |

**What real Spring adds:**
- `BeanDefinitionBuilder.rootBeanDefinition()` for parent-child bean definitions
- `BeanDefinitionBuilder.childBeanDefinition()` for inheriting parent definitions
- `BeanDefinitionBuilder.applyCustomizers()` for post-build modifications (since 5.0)
- `HandlerMethodReturnValueHandlerComposite` — composite for return value handling
- `ViewResolverComposite` — composite for view resolution
- `CompositeDatabasePopulator` — "aggregate" composite (all children execute)
- Caching in composites for previously resolved parameter→resolver mappings

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Builder** | Step-by-step fluent construction of complex objects |
| **Static Factory Methods** | Create builders for different product variants |
| **Composite** | Treat a collection of processors as a single processor |
| **First-Match Composite** | Delegates to the first child that supports the request |
| **Aggregate Composite** | All children execute (e.g., `CompositeDatabasePopulator`) |

**Next: Chapter 10** — We build `SimpleApplicationContext` using the **Facade** and **Aware** patterns to create a unified API.
