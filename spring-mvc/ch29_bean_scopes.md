# Chapter 29: Bean Scopes (Prototype / Request / Session)

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The bean container (ch24) only supports singletons — every bean is created once during `refresh()` and shared for the container's lifetime. AOP (ch28) wraps beans with proxies, but every proxy still delegates to a single target instance. | A shopping cart, user preferences, or request-specific audit logger all need different lifecycles — per-request, per-session, or per-call. A singleton injected with a request-scoped bean holds a stale reference that never changes. | Add prototype scope (new instance per call), request scope (one per HTTP request), and session scope (one per HTTP session). Solve the lifecycle mismatch between singletons and shorter-lived scopes with scoped proxies that delegate to the current scope's instance on every method call. |

---

## 29.1 The Integration Point

The integration point is `SimpleBeanContainer.getBean()`. Currently it always returns a cached singleton or creates one on first access. We need the three-way scope branching that the real `AbstractBeanFactory.doGetBean()` uses (lines 329-383).

**Modifying:** `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java`
**Change:** Replace the single-path `getBean()` with scope-aware `getScopedBean()` that branches on the `BeanDefinition`'s scope.

```java
@Override
public Object getBean(String name) {
    // 1. Check singletons first (pre-registered or already created)
    Object bean = singletonMap.get(name);
    if (bean != null) {
        return bean;
    }

    // 2. Check bean definitions for scope-aware creation
    BeanDefinition definition = beanDefinitions.get(name);
    if (definition != null) {
        return getScopedBean(definition);
    }

    throw new BeanNotFoundException("No bean named '" + name + "' is registered");
}

private Object getScopedBean(BeanDefinition definition) {
    String beanName = definition.getBeanName();

    // Path 1: Singleton — the original behavior
    if (definition.isSingleton()) {
        Object existing = singletonMap.get(beanName);
        if (existing != null) {
            return existing;
        }
        return createBeanInstance(definition);
    }

    // Path 2: Prototype — new instance every time, no caching
    if (definition.isPrototype()) {
        return createPrototypeInstance(definition);
    }

    // Path 3: Custom scope (request, session, etc.)
    String scopeName = definition.getScope();
    Scope scope = scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException(
                "No Scope registered for scope name '" + scopeName + "'");
    }

    try {
        return scope.get(beanName, () -> createPrototypeInstance(definition));
    } catch (IllegalStateException ex) {
        throw new ScopeNotActiveException(beanName, scopeName, ex);
    }
}
```

Two key decisions here:

1. **Scope branching lives in `getBean()`, not `createBean()`** — because scoping is about *retrieval policy* (where to cache), not about *creation logic* (how to instantiate). The creation logic is shared via `createPrototypeInstance()`.
2. **Custom scopes receive an `ObjectFactory` lambda** — the scope decides *whether* to create, the container decides *how*. This callback pattern keeps scopes completely decoupled from the DI machinery.

This connects the **Scope SPI** to the **container's bean retrieval pipeline**. To make it work, we need to build:
- A `Scope` interface — the SPI that custom scopes implement
- `BeanDefinition` scope metadata — so the container knows each bean's scope
- `RequestAttributes` / `RequestContextHolder` — the thread-local bridge between HTTP requests and scopes
- `RequestScope` and `SessionScope` — concrete scope implementations
- `RequestContextFilter` — the Servlet filter that binds request context to the thread
- `@ScopeAnnotation` — for declaring scope on bean classes
- Scoped proxy creation — using AOP (ch28) to solve the singleton→scoped lifecycle mismatch

## 29.2 The Scope SPI

The `Scope` interface is the strategy pattern that lets the container delegate bean storage to external lifecycle managers (HTTP request, HTTP session, or any custom scope).

**New file:** `src/main/java/com/simplespringmvc/scope/Scope.java`

```java
package com.simplespringmvc.scope;

import java.util.function.Supplier;

public interface Scope {

    /**
     * Return the object from the underlying scope, creating it via the
     * objectFactory if not present. This is the central "get-or-create" operation.
     */
    Object get(String name, Supplier<?> objectFactory);

    /** Remove the object with the given name from the underlying scope. */
    Object remove(String name);

    /** Register a callback for when the scoped object is destroyed. */
    void registerDestructionCallback(String name, Runnable callback);

    /** Return an ID for the current underlying scope (e.g., session ID). */
    default String getConversationId() {
        return null;
    }
}
```

The key insight is `Scope.get(name, objectFactory)`: the scope checks its storage — if the bean exists, return it; if not, call `objectFactory.get()` to create it, store it, and return it. The container passes a lambda that calls its own `createPrototypeInstance()` as the object factory.

## 29.3 BeanDefinition Scope Metadata

The `BeanDefinition` needs to carry scope information so the container knows which path to take during `getBean()`.

**Modifying:** `src/main/java/com/simplespringmvc/injection/BeanDefinition.java`
**Change:** Add `scope`, `proxyMode`, and `autowireCandidate` fields with convenience methods.

```java
public static final String SCOPE_SINGLETON = "singleton";
public static final String SCOPE_PROTOTYPE = "prototype";

private String scope = SCOPE_SINGLETON;
private ScopedProxyMode proxyMode = ScopedProxyMode.DEFAULT;
private boolean autowireCandidate = true;

public boolean isSingleton() {
    return SCOPE_SINGLETON.equals(scope) || scope.isEmpty();
}

public boolean isPrototype() {
    return SCOPE_PROTOTYPE.equals(scope);
}
```

The `autowireCandidate` flag is essential for scoped proxies: when a scoped proxy is created, the original target bean is renamed to `scopedTarget.xxx` and marked as `autowireCandidate=false` so that only the proxy (under the original name) gets injected.

## 29.4 RequestContextHolder — The Thread-Local Bridge

Scopes need to access the current HTTP request/session. The `RequestContextHolder` uses a `ThreadLocal` to make the current `RequestAttributes` available anywhere in the call chain.

**New file:** `src/main/java/com/simplespringmvc/scope/RequestContextHolder.java`

```java
package com.simplespringmvc.scope;

public class RequestContextHolder {
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
            new ThreadLocal<>();

    public static void setRequestAttributes(RequestAttributes attributes) {
        requestAttributesHolder.set(attributes);
    }

    public static RequestAttributes getRequestAttributes() {
        return requestAttributesHolder.get();
    }

    public static RequestAttributes currentRequestAttributes() {
        RequestAttributes attributes = requestAttributesHolder.get();
        if (attributes == null) {
            throw new IllegalStateException(
                    "No thread-bound request found. Are you accessing request/session "
                            + "attributes outside of a web request?");
        }
        return attributes;
    }

    public static void resetRequestAttributes() {
        requestAttributesHolder.remove();
    }
}
```

## 29.5 RequestAttributes and ServletRequestAttributes

The `RequestAttributes` interface abstracts access to request- and session-scoped attributes, decoupling the scope implementations from the Servlet API.

**New file:** `src/main/java/com/simplespringmvc/scope/RequestAttributes.java`

```java
package com.simplespringmvc.scope;

public interface RequestAttributes {
    int SCOPE_REQUEST = 0;
    int SCOPE_SESSION = 1;

    Object getAttribute(String name, int scope);
    void setAttribute(String name, Object value, int scope);
    void removeAttribute(String name, int scope);
    void registerDestructionCallback(String name, Runnable callback, int scope);
    String getSessionId();
    Object getSessionMutex();
}
```

**New file:** `src/main/java/com/simplespringmvc/scope/ServletRequestAttributes.java`

The concrete implementation backed by `HttpServletRequest` and `HttpSession`:
- `SCOPE_REQUEST` → `request.getAttribute()` / `request.setAttribute()`
- `SCOPE_SESSION` → `session.getAttribute()` / `session.setAttribute()`
- `requestCompleted()` — executes request-scoped destruction callbacks

## 29.6 AbstractRequestAttributesScope, RequestScope, and SessionScope

The Template Method pattern makes `RequestScope` and `SessionScope` trivially simple — they only differ in which `SCOPE_*` constant they return.

**New file:** `src/main/java/com/simplespringmvc/scope/AbstractRequestAttributesScope.java`

```java
public abstract class AbstractRequestAttributesScope implements Scope {
    protected abstract int getScope();

    @Override
    public Object get(String name, Supplier<?> objectFactory) {
        RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();
        Object scopedObject = attributes.getAttribute(name, getScope());
        if (scopedObject == null) {
            scopedObject = objectFactory.get();
            attributes.setAttribute(name, scopedObject, getScope());
        }
        return scopedObject;
    }

    // remove() and registerDestructionCallback() delegate similarly...
}
```

**New file:** `src/main/java/com/simplespringmvc/scope/RequestScope.java` — returns `SCOPE_REQUEST`

**New file:** `src/main/java/com/simplespringmvc/scope/SessionScope.java` — returns `SCOPE_SESSION`, adds `synchronized(sessionMutex)` around `get()` and `remove()` for thread safety.

## 29.7 @ScopeAnnotation and ScopedProxyMode

**New file:** `src/main/java/com/simplespringmvc/scope/ScopeAnnotation.java`

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ScopeAnnotation {
    String value() default "singleton";
    ScopedProxyMode proxyMode() default ScopedProxyMode.DEFAULT;
}
```

**New file:** `src/main/java/com/simplespringmvc/scope/ScopedProxyMode.java`

```java
public enum ScopedProxyMode {
    DEFAULT,        // Typically equals NO
    NO,             // No proxy — direct injection
    INTERFACES,     // JDK dynamic proxy
    TARGET_CLASS    // CGLIB class-based proxy
}
```

## 29.8 Scoped Proxies — Solving the Lifecycle Mismatch

The scoped proxy is the most important concept in this chapter. When a singleton bean depends on a request-scoped bean, directly injecting the scoped bean would give the singleton a permanent reference to a single instance — but request scope means a *new* instance per request.

The solution: inject a proxy that delegates each method call to the scope's current instance.

**Modifying:** `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java`
**Change:** Add `registerScopedProxies()` called during `refresh()`, and `createScopedProxy()` using AOP from ch28.

```java
private void registerScopedProxies() {
    for (var entry : new ArrayList<>(beanDefinitions.entrySet())) {
        BeanDefinition def = entry.getValue();
        ScopedProxyMode proxyMode = def.getProxyMode();

        if (proxyMode == ScopedProxyMode.DEFAULT || proxyMode == ScopedProxyMode.NO) {
            continue;
        }

        String originalName = entry.getKey();
        String targetName = "scopedTarget." + originalName;

        // Rename target, mark as non-autowirable
        BeanDefinition targetDef = new BeanDefinition(targetName, def.getBeanClass());
        targetDef.setScope(def.getScope());
        targetDef.setAutowireCandidate(false);  // only the proxy gets injected

        // Create the proxy (singleton) under the original name
        Object proxy = createScopedProxy(targetName, def.getBeanClass(), proxyMode);
        singletonMap.put(originalName, proxy);
    }
}

private Object createScopedProxy(String targetBeanName, Class<?> targetClass,
                                  ScopedProxyMode proxyMode) {
    MethodInterceptor scopeInterceptor = invocation -> {
        Object target = getBean(targetBeanName);  // resolves from scope each time
        return invocation.getMethod().invoke(target, invocation.getArguments());
    };
    // ... creates JDK or CGLIB proxy using AOP infrastructure from ch28
}
```

## 29.9 RequestContextFilter — Binding the Request to the Thread

**New file:** `src/main/java/com/simplespringmvc/scope/RequestContextFilter.java`

```java
public class RequestContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        ServletRequestAttributes attributes = new ServletRequestAttributes(request);
        RequestContextHolder.setRequestAttributes(attributes);
        try {
            chain.doFilter(req, res);
        } finally {
            RequestContextHolder.resetRequestAttributes();
            attributes.requestCompleted();
        }
    }
}
```

**Modifying:** `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java`
**Change:** Register the `RequestContextFilter` using Tomcat's FilterDef/FilterMap API.

**Modifying:** `src/main/java/com/simplespringmvc/scan/ClasspathScanner.java`
**Change:** Detect `@ScopeAnnotation` and set scope metadata on the `BeanDefinition`.

## 29.10 Try It Yourself

<details>
<summary>Challenge 1: Implement a prototype-scoped bean</summary>

Create a container, register a `BeanDefinition` with scope "prototype", call `getBean()` three times, and verify you get three different instances.

```java
SimpleBeanContainer container = new SimpleBeanContainer();

BeanDefinition def = new BeanDefinition("counter", Counter.class);
def.setScope("prototype");
container.registerBeanDefinition(def);
container.refresh();

Counter c1 = (Counter) container.getBean("counter");
Counter c2 = (Counter) container.getBean("counter");
Counter c3 = (Counter) container.getBean("counter");

assertThat(c1).isNotSameAs(c2);
assertThat(c2).isNotSameAs(c3);
```

</details>

<details>
<summary>Challenge 2: Use a scoped proxy to inject a custom-scoped bean into a singleton</summary>

Register a scoped bean with `ScopedProxyMode.TARGET_CLASS`, inject it into a singleton via constructor injection, and verify that clearing the scope changes the delegate.

```java
SimpleBeanContainer container = new SimpleBeanContainer();
MapBackedScope scope = new MapBackedScope();
container.registerScope("custom", scope);

BeanDefinition scopedDef = new BeanDefinition("counter", Counter.class);
scopedDef.setScope("custom");
scopedDef.setProxyMode(ScopedProxyMode.TARGET_CLASS);
container.registerBeanDefinition(scopedDef);
container.registerBeanDefinition("service", SingletonService.class);
container.refresh();

SingletonService service = (SingletonService) container.getBean("service");
int id1 = service.getCounterId();
scope.clear();
int id2 = service.getCounterId();

assertThat(id1).isNotEqualTo(id2); // different instances after scope clear
```

</details>

## 29.11 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/scope/ScopeTest.java`

Tests include:
- `PrototypeScope`: `shouldCreateNewInstance_WhenPrototypeBeanRequested`, `shouldNotCachePrototype_InSingletonMap`, `shouldNotInstantiatePrototype_DuringRefresh`
- `CustomScope`: `shouldDelegateToScope_WhenCustomScopedBeanRequested`, `shouldCreateNewInstance_WhenScopeCleared`, `shouldThrowException_WhenScopeNotRegistered`, `shouldRejectRegistration_WhenScopeNameIsSingleton`
- `SingletonScope`: `shouldReturnSameInstance_WhenSingletonBeanRequested` (regression)
- `RequestContextHolderTest`: thread-local binding/retrieval
- `RequestScopeTest`: caching within request, new instance per request
- `SessionScopeTest`: caching across requests in same session, new instance per session
- `ScopedProxyTest`: CGLIB proxy, JDK proxy, scope delegation

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/scope/integration/ScopeIntegrationTest.java`

Tests include:
- Prototype + DI: multiple resolutions produce different instances
- Request scope + container: same instance within request, different across requests
- Session scope + container: shared instance across requests in same session
- Scoped proxy + DI: singleton service delegates to scope via proxy
- Component scanning + @ScopeAnnotation: scope detected during scanning

**Run:** `./gradlew test` — expected: all 1034 tests pass (including prior features' tests)

---

## 29.12 Why This Works

> ★ **Insight: The Scope SPI is a "storage policy" pattern** -------------------------------------------
> - The container knows *how* to create beans (DI, BPP, constructor resolution). The Scope knows *where* to store them (request attributes, session, a Map, a ThreadLocal — anything).
> - `Scope.get(name, objectFactory)` is a callback-driven lazy creation pattern: the scope decides whether to create, the container decides how. Neither knows about the other's internals.
> - This is why you can implement custom scopes (e.g., "conversation", "tenant", "websocket") without modifying the container — you just register a new `Scope` implementation.
> -----------------------------------------------------------

> ★ **Insight: Scoped proxies solve a fundamental object lifecycle problem** -------------------------------------------
> - A singleton bean lives for the entire application. A request-scoped bean lives for one HTTP request. If a singleton holds a direct reference to a request-scoped bean, it would see the same instance forever — violating the request scope contract.
> - The proxy indirection solves this: the singleton holds a proxy (which is itself a singleton), and each method call on the proxy resolves the *current* scoped instance via `getBean("scopedTarget.xxx")` → `scope.get()` → `RequestContextHolder.currentRequestAttributes()`.
> - The real framework uses `ScopedProxyFactoryBean` + `ScopedProxyUtils`, which is more sophisticated than our inline approach. But the core principle is identical: indirection through a proxy that resolves the target per-invocation.
> -----------------------------------------------------------

> ★ **Insight: Prototype scope is deliberately limited** -------------------------------------------
> - Prototypes are NOT cached, NOT stored in the singleton map, and NOT eligible for destruction callbacks. This means the container creates them but does not manage their lifecycle after creation.
> - This is intentional: if the container tracked prototypes, it would need to hold references to every instance ever created, which is a memory leak. The caller is responsible for the prototype's lifecycle.
> - The real framework documents this: "Spring does not manage the complete lifecycle of a prototype bean: the container instantiates, configures, and otherwise assembles a prototype object, and hands it to the client, with no further record of that prototype instance."
> -----------------------------------------------------------

## 29.13 What We Enhanced

| Aspect | Before (ch28) | Current (ch29) | Real Framework |
|--------|---------------|----------------|----------------|
| Bean scopes | Singleton only — every bean created once and cached forever | Three-way branching: singleton, prototype, custom scopes (request/session) | Full scope SPI with singleton, prototype, request, session, application scopes + `@Scope` annotation + `ScopedProxyFactoryBean` — `AbstractBeanFactory.doGetBean()` lines 329-383 |
| BeanDefinition | Name + class only | Name + class + scope + proxyMode + autowireCandidate | Rich metadata with scope, lazy-init, @Primary, @DependsOn, autowire mode, factory method — `BeanDefinition.java` |
| Refresh lifecycle | Creates all bean definitions during refresh | Skips non-singleton beans during refresh (created on demand) | `preInstantiateSingletons()` explicitly checks `!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()` |
| Dependency resolution | Always searches singletonMap | Filters by `autowireCandidate` flag, scope-aware creation | Complex autowire candidate filtering with `@Primary`, `@Fallback`, generics matching |
| Request context | No thread-local request binding | `RequestContextFilter` + `RequestContextHolder` for thread-local request attributes | `RequestContextFilter` + `FrameworkServlet.processRequest()` both set thread-locals, with `InheritableThreadLocal` support |

## 29.14 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Scope` interface | `o.s.beans.factory.config.Scope` | `Scope.java:43` | Real version uses `ObjectFactory<?>` instead of `Supplier<?>`, has `resolveContextualObject()` for SpEL |
| `getScopedBean()` 3-way branch | `AbstractBeanFactory.doGetBean()` | `AbstractBeanFactory.java:329-383` | Real version has parent factory delegation, FactoryBean unwrapping, `beforePrototypeCreation()` tracking |
| `RequestScope` | `o.s.web.context.request.RequestScope` | `RequestScope.java:33` | Identical pattern — one-liner returning `SCOPE_REQUEST` |
| `SessionScope` | `o.s.web.context.request.SessionScope` | `SessionScope.java:38` | Real version uses `WebUtils.getSessionMutex()` for cluster-safe locking |
| `AbstractRequestAttributesScope.get()` | Same | `AbstractRequestAttributesScope.java:44` | Real version re-retrieves after `setAttribute()` for implicit session decoration |
| `RequestContextHolder` | Same | `RequestContextHolder.java:47` | Real version has two ThreadLocals (normal + inheritable), JSF `FacesContext` fallback |
| `RequestContextFilter` | Same | `RequestContextFilter.java:71` | Real version extends `OncePerRequestFilter`, has `threadContextInheritable` property |
| `createScopedProxy()` | `ScopedProxyUtils.createScopedProxy()` | `ScopedProxyUtils.java:58` | Real version creates a `ScopedProxyFactoryBean` bean definition, copies `@Qualifier` metadata |
| `BeanDefinition.scope` | `AbstractBeanDefinition.scope` | `AbstractBeanDefinition.java:171` | Real version part of a rich metadata hierarchy with parent definitions |
| `ScopeAnnotation` | `@Scope` | `Scope.java` (context.annotation) | Real version uses `@AliasFor` for `value`/`scopeName`, lives in different package |
| `ScopedProxyMode` | Same | `ScopedProxyMode.java:30` | Identical enum values |
| `ServletRequestAttributes` | Same | `ServletRequestAttributes.java:52` | Real version tracks session attribute changes for cluster replication, uses `DestructionCallbackBindingListener` for session destruction |

## 29.15 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/scope/Scope.java` [NEW]

```java
package com.simplespringmvc.scope;

import java.util.function.Supplier;

/**
 * Strategy interface for custom bean scopes beyond singleton and prototype.
 * Each scope controls where bean instances are stored and how long they live.
 *
 * Maps to: {@code org.springframework.beans.factory.config.Scope}
 * at spring-beans/src/main/java/.../config/Scope.java
 *
 * <h3>The key insight: Scope.get() is a "get-or-create" operation</h3>
 * The container calls {@code scope.get(name, objectFactory)} when it needs a
 * scoped bean. The scope checks its storage (request attributes, session, etc.)
 * — if the bean exists, return it; if not, create it via the objectFactory
 * and store it. This callback-driven pattern lets the container stay scope-agnostic.
 *
 * <h3>Built-in scopes:</h3>
 * <pre>
 *   "singleton"  — one instance per container (handled internally, not via Scope SPI)
 *   "prototype"  — new instance every time (handled internally, not via Scope SPI)
 *   "request"    — one instance per HTTP request  (RequestScope)
 *   "session"    — one instance per HTTP session   (SessionScope)
 * </pre>
 *
 * Implementations must be thread-safe.
 */
public interface Scope {

    /**
     * Return the object from the underlying scope, creating it via the
     * {@code objectFactory} if not present.
     *
     * Maps to: {@code Scope.get(String, ObjectFactory<?>)}
     *
     * @param name          the bean name
     * @param objectFactory factory to create the bean if absent
     * @return the scoped object (never null)
     */
    Object get(String name, Supplier<?> objectFactory);

    /**
     * Remove the object with the given name from the underlying scope.
     *
     * @param name the bean name
     * @return the removed object, or {@code null} if not found
     */
    Object remove(String name);

    /**
     * Register a callback to be executed when the scoped object is destroyed
     * (e.g., when the request completes or session is invalidated).
     *
     * @param name     the bean name
     * @param callback the destruction callback
     */
    void registerDestructionCallback(String name, Runnable callback);

    /**
     * Return an ID for the current underlying scope, if any.
     * For example, the session ID for session scope.
     *
     * @return the conversation ID, or {@code null}
     */
    default String getConversationId() {
        return null;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/ScopeAnnotation.java` [NEW]

```java
package com.simplespringmvc.scope;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Declares the scope of a bean — controls how many instances are created
 * and where they are stored.
 *
 * Maps to: {@code org.springframework.context.annotation.Scope}
 * at spring-context/src/main/java/.../annotation/Scope.java
 *
 * <h3>Usage:</h3>
 * <pre>
 *   {@literal @}Component
 *   {@literal @}ScopeAnnotation("prototype")
 *   public class ShoppingCart { ... }
 *
 *   {@literal @}Component
 *   {@literal @}ScopeAnnotation(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
 *   public class UserPreferences { ... }
 * </pre>
 *
 * Named {@code ScopeAnnotation} instead of {@code Scope} to avoid collision
 * with the {@link Scope} interface in the same package.
 * The real framework has them in different packages (context.annotation vs beans.factory.config).
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ScopeAnnotation {

    /**
     * The scope name: "singleton" (default), "prototype", "request", or "session".
     */
    String value() default "singleton";

    /**
     * Whether to create a scoped proxy for this bean, and if so, which kind.
     * Scoped proxies are essential when injecting a shorter-lived scoped bean
     * (request/session) into a longer-lived bean (singleton).
     */
    ScopedProxyMode proxyMode() default ScopedProxyMode.DEFAULT;
}
```

#### File: `src/main/java/com/simplespringmvc/scope/ScopedProxyMode.java` [NEW]

```java
package com.simplespringmvc.scope;

/**
 * Controls whether and how a scoped proxy is created for a bean.
 *
 * Maps to: {@code org.springframework.context.annotation.ScopedProxyMode}
 * at spring-context/src/main/java/.../annotation/ScopedProxyMode.java
 *
 * <h3>Why scoped proxies exist:</h3>
 * A singleton bean injected with a request-scoped bean would hold a direct
 * reference to a single instance — but request scope means a NEW instance
 * per request. The proxy solves this: the singleton holds a proxy that
 * delegates every method call to the current scope's real instance.
 *
 * <pre>
 *   Singleton → Proxy → [on each method call] → RequestContextHolder
 *                                                → current request's instance
 * </pre>
 */
public enum ScopedProxyMode {

    /**
     * Typically equivalent to {@link #NO}, unless configured otherwise
     * at the component-scan level.
     */
    DEFAULT,

    /**
     * No scoped proxy. The bean itself is injected directly.
     * Not useful for request/session scoped beans used as dependencies.
     */
    NO,

    /**
     * Create a JDK dynamic proxy implementing all interfaces exposed
     * by the target bean.
     */
    INTERFACES,

    /**
     * Create a CGLIB class-based proxy (subclass of the target class).
     * Works even when the bean doesn't implement interfaces.
     */
    TARGET_CLASS
}
```

#### File: `src/main/java/com/simplespringmvc/scope/RequestAttributes.java` [NEW]

```java
package com.simplespringmvc.scope;

/**
 * Abstraction for accessing request-scoped and session-scoped attributes,
 * decoupling the scope implementation from the Servlet API.
 *
 * Maps to: {@code org.springframework.web.context.request.RequestAttributes}
 * at spring-web/src/main/java/.../request/RequestAttributes.java
 *
 * This interface lets {@link RequestScope} and {@link SessionScope} work
 * without directly depending on HttpServletRequest/HttpSession — they just
 * call getAttribute/setAttribute with the appropriate scope constant.
 */
public interface RequestAttributes {

    /** Constant for request scope. */
    int SCOPE_REQUEST = 0;

    /** Constant for session scope. */
    int SCOPE_SESSION = 1;

    /**
     * Return the value of the scoped attribute, or {@code null} if not found.
     *
     * @param name  the attribute name
     * @param scope SCOPE_REQUEST or SCOPE_SESSION
     */
    Object getAttribute(String name, int scope);

    /**
     * Set the value of the scoped attribute.
     *
     * @param name  the attribute name
     * @param value the attribute value
     * @param scope SCOPE_REQUEST or SCOPE_SESSION
     */
    void setAttribute(String name, Object value, int scope);

    /**
     * Remove the scoped attribute.
     *
     * @param name  the attribute name
     * @param scope SCOPE_REQUEST or SCOPE_SESSION
     */
    void removeAttribute(String name, int scope);

    /**
     * Register a destruction callback for the given attribute.
     *
     * @param name     the attribute name
     * @param callback the callback to execute on destruction
     * @param scope    SCOPE_REQUEST or SCOPE_SESSION
     */
    void registerDestructionCallback(String name, Runnable callback, int scope);

    /**
     * Return the session ID, if available.
     */
    String getSessionId();

    /**
     * Return the mutex object for the underlying session, for synchronization.
     */
    Object getSessionMutex();
}
```

#### File: `src/main/java/com/simplespringmvc/scope/ServletRequestAttributes.java` [NEW]

```java
package com.simplespringmvc.scope;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Servlet-based implementation of {@link RequestAttributes}, backed by
 * {@link HttpServletRequest} and {@link HttpSession}.
 *
 * Maps to: {@code org.springframework.web.context.request.ServletRequestAttributes}
 * at spring-web/src/main/java/.../request/ServletRequestAttributes.java
 *
 * <h3>Attribute storage:</h3>
 * <pre>
 *   SCOPE_REQUEST → request.getAttribute() / request.setAttribute()
 *   SCOPE_SESSION → session.getAttribute() / session.setAttribute()
 * </pre>
 *
 * <h3>Simplifications vs real Spring:</h3>
 * <ul>
 *   <li>No session attribute change tracking for cluster replication</li>
 *   <li>No DestructionCallbackBindingListener for session destruction</li>
 *   <li>No response reference</li>
 * </ul>
 */
public class ServletRequestAttributes implements RequestAttributes {

    private final HttpServletRequest request;

    /**
     * Request-scoped destruction callbacks, keyed by attribute name.
     * Executed when {@link #requestCompleted()} is called.
     */
    private final Map<String, Runnable> requestDestructionCallbacks = new ConcurrentHashMap<>();

    public ServletRequestAttributes(HttpServletRequest request) {
        this.request = request;
    }

    @Override
    public Object getAttribute(String name, int scope) {
        if (scope == SCOPE_REQUEST) {
            return request.getAttribute(name);
        } else {
            HttpSession session = request.getSession(false);
            return session != null ? session.getAttribute(name) : null;
        }
    }

    @Override
    public void setAttribute(String name, Object value, int scope) {
        if (scope == SCOPE_REQUEST) {
            request.setAttribute(name, value);
        } else {
            request.getSession(true).setAttribute(name, value);
        }
    }

    @Override
    public void removeAttribute(String name, int scope) {
        if (scope == SCOPE_REQUEST) {
            request.removeAttribute(name);
        } else {
            HttpSession session = request.getSession(false);
            if (session != null) {
                session.removeAttribute(name);
            }
        }
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback, int scope) {
        if (scope == SCOPE_REQUEST) {
            requestDestructionCallbacks.put(name, callback);
        }
        // Session-scoped destruction callbacks are more complex in real Spring —
        // they use HttpSessionBindingListener (DestructionCallbackBindingListener).
        // We skip session destruction callbacks for simplicity.
    }

    @Override
    public String getSessionId() {
        return request.getSession(true).getId();
    }

    @Override
    public Object getSessionMutex() {
        HttpSession session = request.getSession(true);
        // Real Spring: WebUtils.getSessionMutex(session)
        // checks for a "org.springframework.web.servlet.mvc.MUTEX" attribute.
        // We use the session itself as the mutex — simpler, works for non-clustered.
        return session;
    }

    /**
     * Called when the request completes — executes request-scoped destruction callbacks.
     *
     * Maps to: {@code ServletRequestAttributes.requestCompleted()} (line 232)
     * which also calls updateAccessedSessionAttributes() for cluster replication.
     */
    public void requestCompleted() {
        for (Runnable callback : requestDestructionCallbacks.values()) {
            callback.run();
        }
        requestDestructionCallbacks.clear();
    }

    /**
     * Return the underlying request.
     */
    public HttpServletRequest getRequest() {
        return request;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/RequestContextHolder.java` [NEW]

```java
package com.simplespringmvc.scope;

/**
 * Thread-local holder for the current web request's {@link RequestAttributes}.
 * This is the bridge between the web layer (filter/servlet) and the scope
 * implementations — scopes call {@link #currentRequestAttributes()} to find
 * where to store/retrieve beans.
 *
 * Maps to: {@code org.springframework.web.context.request.RequestContextHolder}
 * at spring-web/src/main/java/.../request/RequestContextHolder.java
 *
 * <h3>Lifecycle:</h3>
 * <pre>
 *   Request arrives
 *     → RequestContextFilter creates ServletRequestAttributes
 *     → RequestContextHolder.setRequestAttributes(attrs)
 *     → ... request processing ...
 *     → scoped beans call RequestContextHolder.currentRequestAttributes()
 *     → ... request completes ...
 *     → RequestContextHolder.resetRequestAttributes()
 * </pre>
 *
 * <h3>Simplification:</h3>
 * Real Spring has two ThreadLocals: one normal, one InheritableThreadLocal
 * for child thread propagation. We use just one.
 */
public class RequestContextHolder {

    private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
            new ThreadLocal<>();

    /**
     * Bind the given RequestAttributes to the current thread.
     */
    public static void setRequestAttributes(RequestAttributes attributes) {
        requestAttributesHolder.set(attributes);
    }

    /**
     * Return the RequestAttributes currently bound to the thread, or {@code null}.
     */
    public static RequestAttributes getRequestAttributes() {
        return requestAttributesHolder.get();
    }

    /**
     * Return the RequestAttributes currently bound to the thread.
     *
     * @throws IllegalStateException if no attributes are bound (i.e., we're not
     *                               in a web request context)
     */
    public static RequestAttributes currentRequestAttributes() {
        RequestAttributes attributes = requestAttributesHolder.get();
        if (attributes == null) {
            throw new IllegalStateException(
                    "No thread-bound request found. "
                            + "Are you referring to request/session attributes outside of a web request? "
                            + "If you are in a filter or DispatcherServlet, use RequestContextFilter "
                            + "to expose the current request.");
        }
        return attributes;
    }

    /**
     * Reset the RequestAttributes for the current thread.
     */
    public static void resetRequestAttributes() {
        requestAttributesHolder.remove();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/AbstractRequestAttributesScope.java` [NEW]

```java
package com.simplespringmvc.scope;

import java.util.function.Supplier;

/**
 * Template Method base class for {@link RequestScope} and {@link SessionScope}.
 * Implements {@link Scope} by delegating to {@link RequestAttributes} with
 * the appropriate scope constant.
 *
 * Maps to: {@code org.springframework.web.context.request.AbstractRequestAttributesScope}
 * at spring-web/src/main/java/.../request/AbstractRequestAttributesScope.java
 *
 * <h3>The pattern:</h3>
 * <pre>
 *   Scope.get("myBean", factory)
 *     → RequestContextHolder.currentRequestAttributes()     ← get thread-bound attrs
 *     → attributes.getAttribute("myBean", getScope())       ← check storage
 *     → if null: factory.get() → attributes.setAttribute()  ← create + store
 *     → return the bean
 * </pre>
 *
 * Subclasses only implement {@link #getScope()} to return the scope constant
 * ({@link RequestAttributes#SCOPE_REQUEST} or {@link RequestAttributes#SCOPE_SESSION}).
 */
public abstract class AbstractRequestAttributesScope implements Scope {

    /**
     * Return the scope constant: {@link RequestAttributes#SCOPE_REQUEST}
     * or {@link RequestAttributes#SCOPE_SESSION}.
     */
    protected abstract int getScope();

    /**
     * Get or create the scoped object.
     *
     * Maps to: {@code AbstractRequestAttributesScope.get()} (line 44-58)
     *
     * The real version re-retrieves after setAttribute to handle implicit
     * session attribute decoration. We skip that subtlety.
     */
    @Override
    public Object get(String name, Supplier<?> objectFactory) {
        RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();
        Object scopedObject = attributes.getAttribute(name, getScope());
        if (scopedObject == null) {
            scopedObject = objectFactory.get();
            attributes.setAttribute(name, scopedObject, getScope());
        }
        return scopedObject;
    }

    @Override
    public Object remove(String name) {
        RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();
        Object scopedObject = attributes.getAttribute(name, getScope());
        if (scopedObject != null) {
            attributes.removeAttribute(name, getScope());
        }
        return scopedObject;
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        RequestContextHolder.currentRequestAttributes()
                .registerDestructionCallback(name, callback, getScope());
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/RequestScope.java` [NEW]

```java
package com.simplespringmvc.scope;

/**
 * {@link Scope} implementation for the "request" scope — one bean instance
 * per HTTP request, stored as a request attribute.
 *
 * Maps to: {@code org.springframework.web.context.request.RequestScope}
 * at spring-web/src/main/java/.../request/RequestScope.java
 *
 * The real implementation is remarkably simple — just a one-liner returning
 * {@code RequestAttributes.SCOPE_REQUEST}. All logic lives in
 * {@link AbstractRequestAttributesScope}.
 *
 * <h3>Lifecycle:</h3>
 * <pre>
 *   Request 1:  scope.get("cart", factory)  → creates Cart@1, stored in request attrs
 *               scope.get("cart", factory)  → returns same Cart@1  (same request)
 *   Request 2:  scope.get("cart", factory)  → creates Cart@2, stored in NEW request attrs
 * </pre>
 */
public class RequestScope extends AbstractRequestAttributesScope {

    @Override
    protected int getScope() {
        return RequestAttributes.SCOPE_REQUEST;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/SessionScope.java` [NEW]

```java
package com.simplespringmvc.scope;

import java.util.function.Supplier;

/**
 * {@link Scope} implementation for the "session" scope — one bean instance
 * per HTTP session, stored as a session attribute.
 *
 * Maps to: {@code org.springframework.web.context.request.SessionScope}
 * at spring-web/src/main/java/.../request/SessionScope.java
 *
 * <h3>Key difference from RequestScope: synchronization</h3>
 * Session-scoped beans can be accessed concurrently — multiple requests from
 * the same user can hit the server simultaneously. The real framework wraps
 * {@code get()} and {@code remove()} in {@code synchronized(sessionMutex)}
 * to prevent concurrent modification. We do the same.
 *
 * <h3>Lifecycle:</h3>
 * <pre>
 *   User A, Request 1: scope.get("prefs", factory) → creates Prefs@1, stored in session
 *   User A, Request 2: scope.get("prefs", factory) → returns same Prefs@1 (same session)
 *   User B, Request 1: scope.get("prefs", factory) → creates Prefs@2 (different session)
 * </pre>
 */
public class SessionScope extends AbstractRequestAttributesScope {

    @Override
    protected int getScope() {
        return RequestAttributes.SCOPE_SESSION;
    }

    /**
     * Synchronized access to session-scoped beans.
     *
     * Maps to: {@code SessionScope.get()} (line 50-55)
     * which synchronizes on {@code RequestContextHolder.currentRequestAttributes().getSessionMutex()}.
     */
    @Override
    public Object get(String name, Supplier<?> objectFactory) {
        Object mutex = RequestContextHolder.currentRequestAttributes().getSessionMutex();
        synchronized (mutex) {
            return super.get(name, objectFactory);
        }
    }

    /**
     * Synchronized removal of session-scoped beans.
     */
    @Override
    public Object remove(String name) {
        Object mutex = RequestContextHolder.currentRequestAttributes().getSessionMutex();
        synchronized (mutex) {
            return super.remove(name);
        }
    }

    @Override
    public String getConversationId() {
        return RequestContextHolder.currentRequestAttributes().getSessionId();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/RequestContextFilter.java` [NEW]

```java
package com.simplespringmvc.scope;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.FilterConfig;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;

import java.io.IOException;

/**
 * Servlet Filter that binds the current HTTP request to the thread via
 * {@link RequestContextHolder}, enabling request/session scoped beans.
 *
 * Maps to: {@code org.springframework.web.filter.RequestContextFilter}
 * at spring-web/src/main/java/.../filter/RequestContextFilter.java
 *
 * <h3>The critical role of this filter:</h3>
 * Without it, {@link RequestContextHolder#currentRequestAttributes()} throws
 * an exception, and request/session scoped beans cannot be resolved. This
 * filter is the bridge between the Servlet container's per-request lifecycle
 * and Spring's scope abstraction.
 *
 * <h3>Lifecycle:</h3>
 * <pre>
 *   HTTP Request arrives
 *     → Filter.doFilter()
 *       → create ServletRequestAttributes(request)
 *       → RequestContextHolder.setRequestAttributes(attrs)
 *       → filterChain.doFilter(request, response)     ← process the request
 *       → finally:
 *           → attrs.requestCompleted()                 ← run destruction callbacks
 *           → RequestContextHolder.resetRequestAttributes()
 * </pre>
 */
public class RequestContextFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // No initialization needed
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        ServletRequestAttributes attributes = new ServletRequestAttributes(request);

        RequestContextHolder.setRequestAttributes(attributes);
        try {
            filterChain.doFilter(servletRequest, servletResponse);
        } finally {
            RequestContextHolder.resetRequestAttributes();
            attributes.requestCompleted();
        }
    }

    @Override
    public void destroy() {
        // No cleanup needed
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scope/ScopeNotActiveException.java` [NEW]

```java
package com.simplespringmvc.scope;

/**
 * Thrown when a scoped bean is requested but the scope is not currently active.
 * For example, accessing a request-scoped bean outside of an HTTP request.
 *
 * Maps to: {@code org.springframework.beans.factory.support.ScopeNotActiveException}
 */
public class ScopeNotActiveException extends RuntimeException {

    public ScopeNotActiveException(String beanName, String scopeName, Throwable cause) {
        super("Scope '" + scopeName + "' is not active for bean '" + beanName
                + "'. Are you accessing a " + scopeName + "-scoped bean outside of "
                + (scopeName.equals("request") ? "an HTTP request"
                : "an HTTP session") + "?", cause);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/BeanDefinition.java` [MODIFIED]

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.scope.ScopedProxyMode;

/**
 * Metadata describing a bean to be created by the container — the class to
 * instantiate and the name to register it under.
 *
 * Maps to: {@code org.springframework.beans.factory.config.BeanDefinition}
 * (interface) and {@code org.springframework.beans.factory.support.RootBeanDefinition}
 * (primary implementation)
 *
 * <h3>Real vs simplified:</h3>
 * The real BeanDefinition is a rich metadata object storing:
 * <ul>
 *   <li>Bean class name, scope (singleton/prototype/request/session)</li>
 *   <li>Constructor argument values and property values</li>
 *   <li>Factory method, init-method, destroy-method references</li>
 *   <li>Lazy-init flag, @Primary, @DependsOn, autowire mode</li>
 *   <li>Source (annotation metadata, XML element)</li>
 * </ul>
 *
 * Our BeanDefinition holds the bean name, class, and scope — enough to demonstrate
 * the key insight: <b>separating registration from instantiation</b> is what
 * enables dependency ordering, constructor injection, and post-processing.
 *
 * <h3>Ch29 upgrade: scope metadata</h3>
 * Added scope name and scoped proxy mode fields. The container reads these
 * during {@code getBean()} to determine whether to branch into singleton,
 * prototype, or custom scope paths — mirroring the three-way branch in
 * {@code AbstractBeanFactory.doGetBean()}.
 */
public class BeanDefinition {

    /** Scope constant for singletons. */
    public static final String SCOPE_SINGLETON = "singleton";
    /** Scope constant for prototypes. */
    public static final String SCOPE_PROTOTYPE = "prototype";

    private final String beanName;
    private final Class<?> beanClass;

    // ─── Ch29: Scope metadata ────────────────────────────────────────
    private String scope = SCOPE_SINGLETON;
    private ScopedProxyMode proxyMode = ScopedProxyMode.DEFAULT;
    /**
     * Whether this bean is a candidate for autowiring. Set to false for
     * scoped proxy targets — only the proxy should be injected.
     * Maps to: {@code AbstractBeanDefinition.autowireCandidate}
     */
    private boolean autowireCandidate = true;

    public BeanDefinition(String beanName, Class<?> beanClass) {
        this.beanName = beanName;
        this.beanClass = beanClass;
    }

    public String getBeanName() {
        return beanName;
    }

    public Class<?> getBeanClass() {
        return beanClass;
    }

    // ─── Ch29: Scope accessors ───────────────────────────────────────

    /**
     * Return the scope name. Defaults to "singleton".
     *
     * Maps to: {@code BeanDefinition.getScope()} — the real framework returns
     * null or empty for singletons and "prototype" for prototypes.
     */
    public String getScope() {
        return scope;
    }

    public void setScope(String scope) {
        this.scope = scope != null && !scope.isEmpty() ? scope : SCOPE_SINGLETON;
    }

    public boolean isSingleton() {
        return SCOPE_SINGLETON.equals(scope) || scope.isEmpty();
    }

    public boolean isPrototype() {
        return SCOPE_PROTOTYPE.equals(scope);
    }

    public ScopedProxyMode getProxyMode() {
        return proxyMode;
    }

    public void setProxyMode(ScopedProxyMode proxyMode) {
        this.proxyMode = proxyMode;
    }

    public boolean isAutowireCandidate() {
        return autowireCandidate;
    }

    public void setAutowireCandidate(boolean autowireCandidate) {
        this.autowireCandidate = autowireCandidate;
    }

    @Override
    public String toString() {
        return "BeanDefinition{name='" + beanName + "', class=" + beanClass.getName()
                + ", scope='" + scope + "'}";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java` [MODIFIED]

```java
package com.simplespringmvc.container;

import com.simplespringmvc.aop.AdvisedSupport;
import com.simplespringmvc.aop.CglibAopProxy;
import com.simplespringmvc.aop.JdkDynamicAopProxy;
import com.simplespringmvc.aop.MethodInterceptor;
import com.simplespringmvc.aop.aspectj.AspectJAutoProxyCreator;
import com.simplespringmvc.aop.aspectj.EnableAspectJAutoProxy;
import com.simplespringmvc.injection.AutowiredAnnotationBeanPostProcessor;
import com.simplespringmvc.injection.BeanDefinition;
import com.simplespringmvc.injection.BeanPostProcessor;
import com.simplespringmvc.injection.CircularDependencyException;
import com.simplespringmvc.injection.ConstructorResolver;
import com.simplespringmvc.injection.PropertySource;
import com.simplespringmvc.injection.UnsatisfiedDependencyException;
import com.simplespringmvc.scope.Scope;
import com.simplespringmvc.scope.ScopeNotActiveException;
import com.simplespringmvc.scope.ScopedProxyMode;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * A minimal IoC container that stores singleton beans in a Map and supports
 * retrieval by name or type (including interface lookup), plus dependency
 * injection via constructor autowiring, @Autowired fields/methods, and @Value.
 *
 * Maps to: {@code org.springframework.beans.factory.support.DefaultListableBeanFactory}
 *
 * <h3>Two-phase lifecycle (ch24 upgrade):</h3>
 * <pre>
 *   Phase 1 — Registration (no instantiation):
 *     registerBean("name", instance)          → pre-instantiated singleton (ch01)
 *     registerBeanDefinition("name", Class)   → deferred: class only, no instance yet
 *
 *   Phase 2 — Refresh (instantiation with DI):
 *     refresh()
 *       → register AutowiredAnnotationBeanPostProcessor
 *       → instantiate BeanPostProcessor definitions first
 *       → instantiate all remaining definitions in order
 *         → for each: ConstructorResolver.autowireConstructor()
 *                    → BPP.postProcessBeforeInitialization()  (@Autowired/@Value injection)
 *                    → BPP.postProcessAfterInitialization()   (proxy wrapping, future AOP)
 * </pre>
 *
 * <h3>Core data structures (ch24 upgrade):</h3>
 * <pre>
 *   singletonMap:     ConcurrentHashMap&lt;String, Object&gt;   ← fully created beans
 *   beanDefinitions:  LinkedHashMap&lt;String, BeanDefinition&gt; ← class metadata, not yet instantiated
 *   beanPostProcessors: List&lt;BeanPostProcessor&gt;            ← lifecycle hooks
 *   currentlyCreating: Set&lt;String&gt;                         ← circular dependency detection
 *   propertySource:   PropertySource                         ← externalized config for @Value
 * </pre>
 *
 * <h3>Ch29 upgrade: scope support</h3>
 * <pre>
 *   scopes:  Map&lt;String, Scope&gt;  ← registered custom scopes ("request", "session")
 *
 *   getBean() now branches on the BeanDefinition's scope:
 *     "singleton"  → existing path: singletonMap cache
 *     "prototype"  → create new instance every time, no caching
 *     custom scope → delegate to Scope.get(name, objectFactory)
 *
 *   Scoped proxies: for request/session beans injected into singletons,
 *   the container creates a proxy (via AOP from ch28) that delegates
 *   each method call to the scope's current instance.
 * </pre>
 *
 * Pre-instantiated beans (registerBean) skip the DI lifecycle — they're already
 * fully initialized. Only bean definitions (registerBeanDefinition) go through
 * constructor resolution and BeanPostProcessor processing.
 *
 * Every future feature connects here:
 * - DispatcherServlet (ch02) retrieves strategy beans from the container
 * - HandlerMapping (ch03) iterates all beans to find @Controller classes
 * - Component Scanning (ch17) auto-registers discovered classes
 * - Constructor Injection (ch24) resolves dependencies during instantiation
 * - AOP (ch28) wraps beans with proxies via postProcessAfterInitialization()
 * - Bean Scopes (ch29) adds prototype/request/session scope support
 */
public class SimpleBeanContainer implements BeanContainer {

    /**
     * The central data structure — maps bean name → singleton instance.
     *
     * Real Spring equivalent: DefaultSingletonBeanRegistry.singletonObjects
     * (ConcurrentHashMap<String, Object>) at DefaultSingletonBeanRegistry.java:128
     *
     * We use ConcurrentHashMap for thread-safety, matching what the real framework does.
     */
    private final Map<String, Object> singletonMap = new ConcurrentHashMap<>(64);

    /**
     * Preserves registration order, matching real Spring's beanDefinitionNames list.
     * This ensures getBeansOfType returns beans in a deterministic order.
     */
    private final List<String> beanNames = new ArrayList<>();

    // ─── Ch24: Dependency Injection support ─────────────────────────

    /**
     * Bean definitions awaiting instantiation — registered via registerBeanDefinition().
     * Instantiated during refresh() with full DI lifecycle.
     *
     * Real Spring equivalent: DefaultListableBeanFactory.beanDefinitionMap
     * (ConcurrentHashMap<String, BeanDefinition>)
     */
    private final Map<String, BeanDefinition> beanDefinitions = new LinkedHashMap<>();

    /**
     * Registered BeanPostProcessors, applied to every bean created during refresh().
     *
     * Real Spring equivalent: AbstractBeanFactory.beanPostProcessors (BeanPostProcessorCache)
     */
    private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();

    /**
     * Tracks beans currently being created — detects circular dependencies.
     * If we encounter a bean name already in this set during createBean(),
     * it means A → B → ... → A, which is a cycle.
     *
     * Real Spring equivalent: DefaultSingletonBeanRegistry.singletonsCurrentlyInCreation
     * (Collections.newSetFromMap(ConcurrentHashMap))
     */
    private final Set<String> currentlyCreating = new HashSet<>();

    /**
     * Externalized configuration properties for @Value resolution.
     * Default is empty — set via setPropertySource() before refresh().
     */
    private PropertySource propertySource = new PropertySource(Map.of());

    // ─── Ch29: Scope support ────────────────────────────────────────

    /**
     * Registered custom scopes, keyed by scope name (e.g., "request", "session").
     * Singleton and prototype scopes are built-in and not stored here.
     *
     * Maps to: {@code AbstractBeanFactory.scopes} (line 158)
     * (LinkedHashMap<String, Scope>)
     */
    private final Map<String, Scope> scopes = new LinkedHashMap<>(8);

    // ─── Bean registration (ch01, unchanged) ────────────────────────

    @Override
    public void registerBean(String name, Object bean) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Bean name must not be null or blank");
        }
        if (bean == null) {
            throw new IllegalArgumentException("Bean instance must not be null");
        }
        singletonMap.put(name, bean);
        if (!beanNames.contains(name)) {
            beanNames.add(name);
        }
    }

    @Override
    public void registerBean(Object bean) {
        String name = deriveBeanName(bean.getClass());
        registerBean(name, bean);
    }

    // ─── Bean retrieval (ch29: scope-aware getBean) ─────────────────

    /**
     * Retrieve a bean by name.
     *
     * <h3>Ch29 upgrade: three-way scope branching</h3>
     * This now mirrors {@code AbstractBeanFactory.doGetBean()} (lines 329-383):
     * <pre>
     *   if (singleton)  → check singletonMap cache, create if absent
     *   if (prototype)  → always create a new instance
     *   else (custom)   → delegate to Scope.get(name, objectFactory)
     * </pre>
     *
     * @throws BeanNotFoundException if no bean exists with this name
     */
    @Override
    public Object getBean(String name) {
        // 1. Check singletons first (pre-registered or already created)
        Object bean = singletonMap.get(name);
        if (bean != null) {
            return bean;
        }

        // 2. Check bean definitions for scope-aware creation
        BeanDefinition definition = beanDefinitions.get(name);
        if (definition != null) {
            return getScopedBean(definition);
        }

        throw new BeanNotFoundException("No bean named '" + name + "' is registered");
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> type) {
        Map<String, T> matches = getBeansOfType(type);

        if (matches.isEmpty()) {
            // Ch24: check uninstantiated definitions
            String defName = findBeanDefinitionByType(type);
            if (defName != null) {
                getScopedBean(beanDefinitions.get(defName));
                return getBean(type); // retry after creation
            }
            throw new BeanNotFoundException(
                    "No bean of type '" + type.getName() + "' is registered");
        }
        if (matches.size() > 1) {
            throw new NoUniqueBeanException(
                    "Expected single bean of type '" + type.getName()
                            + "' but found " + matches.size() + ": " + matches.keySet());
        }

        return matches.values().iterator().next();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> Map<String, T> getBeansOfType(Class<T> type) {
        Map<String, T> result = new LinkedHashMap<>();
        for (String name : beanNames) {
            Object bean = singletonMap.get(name);
            if (type.isInstance(bean)) {
                result.put(name, (T) bean);
            }
        }
        return result;
    }

    @Override
    public List<String> getBeanNames() {
        return List.copyOf(beanNames);
    }

    @Override
    public boolean containsBean(String name) {
        // Ch24: also check bean definitions (not yet instantiated)
        return singletonMap.containsKey(name) || beanDefinitions.containsKey(name);
    }

    // ─── Ch24: Bean definition registration ─────────────────────────

    /**
     * Register a class for deferred instantiation with dependency injection.
     * The bean will be created during {@link #refresh()} (or on first access).
     *
     * Maps to: {@code DefaultListableBeanFactory.registerBeanDefinition()}
     * (line 1186) which stores the BeanDefinition in beanDefinitionMap.
     *
     * @param name      the bean name
     * @param beanClass the class to instantiate
     */
    public void registerBeanDefinition(String name, Class<?> beanClass) {
        beanDefinitions.put(name, new BeanDefinition(name, beanClass));
        if (!beanNames.contains(name)) {
            beanNames.add(name);
        }
    }

    /**
     * Register a pre-built BeanDefinition (with scope metadata already set).
     * Used by ClasspathScanner when it detects @ScopeAnnotation.
     *
     * @param definition the bean definition to register
     */
    public void registerBeanDefinition(BeanDefinition definition) {
        beanDefinitions.put(definition.getBeanName(), definition);
        if (!beanNames.contains(definition.getBeanName())) {
            beanNames.add(definition.getBeanName());
        }
    }

    /**
     * Check whether a bean definition (not yet instantiated) exists with this name.
     */
    public boolean containsBeanDefinition(String name) {
        return beanDefinitions.containsKey(name);
    }

    /**
     * Return a bean definition by name, or {@code null} if not found.
     */
    public BeanDefinition getBeanDefinition(String name) {
        return beanDefinitions.get(name);
    }

    // ─── Ch24: Refresh — instantiate all definitions with DI ────────

    /**
     * Instantiate all registered bean definitions with full DI lifecycle.
     * This is the core lifecycle method — it transforms class metadata into
     * live, fully-wired singleton instances.
     *
     * Maps to: {@code AbstractApplicationContext.refresh()} (line 596)
     *  → {@code finishBeanFactoryInitialization()} (line 937)
     *    → {@code DefaultListableBeanFactory.preInstantiateSingletons()} (line 1008)
     *
     * <h3>Lifecycle order:</h3>
     * <ol>
     *   <li>Auto-register the AutowiredAnnotationBeanPostProcessor</li>
     *   <li>Instantiate BeanPostProcessor definitions first (they're infrastructure)</li>
     *   <li>Instantiate all remaining singleton bean definitions</li>
     * </ol>
     *
     * <h3>Ch29 change:</h3>
     * Non-singleton beans (prototype, request, session) are NOT instantiated during
     * refresh() — they are created on-demand when first requested. This matches
     * the real Spring behavior: preInstantiateSingletons() skips non-singleton defs.
     *
     * Idempotent for already-created beans — safe to call multiple times.
     */
    public void refresh() {
        // 1. Auto-register the @Autowired/@Value processor if not already present
        // Real Spring: AnnotationConfigUtils.registerAnnotationConfigProcessors()
        boolean hasAutowiredProcessor = beanPostProcessors.stream()
                .anyMatch(bpp -> bpp instanceof AutowiredAnnotationBeanPostProcessor);
        if (!hasAutowiredProcessor) {
            beanPostProcessors.add(new AutowiredAnnotationBeanPostProcessor(this));
        }

        // 1b. Ch28: Auto-register AspectJAutoProxyCreator if @EnableAspectJAutoProxy found
        autoRegisterAopProcessorIfNeeded();

        // 1c. Ch29: Register scoped proxy bean definitions for beans that need them.
        // This must happen BEFORE BPP and bean instantiation so that singletons
        // that depend on scoped beans get the proxy, not the raw bean.
        registerScopedProxies();

        // 2. Instantiate BeanPostProcessor definitions first
        // Real Spring: AbstractApplicationContext.registerBeanPostProcessors() (line 781)
        for (var entry : new ArrayList<>(beanDefinitions.entrySet())) {
            if (BeanPostProcessor.class.isAssignableFrom(entry.getValue().getBeanClass())) {
                Object bpp = createBeanInstance(entry.getValue());
                if (bpp instanceof BeanPostProcessor processor) {
                    addBeanPostProcessor(processor);
                }
            }
        }

        // 3. Instantiate all remaining singleton bean definitions
        // Ch29: skip non-singleton beans — they're created on demand
        for (var entry : new ArrayList<>(beanDefinitions.entrySet())) {
            BeanDefinition def = entry.getValue();
            if (def.isSingleton() && !singletonMap.containsKey(entry.getKey())) {
                createBeanInstance(def);
            }
        }
    }

    // ─── Ch29: Scope-aware bean retrieval ───────────────────────────

    /**
     * Get a bean respecting its scope. This is the three-way branch that mirrors
     * {@code AbstractBeanFactory.doGetBean()} (lines 329-383).
     *
     * <h3>The three paths:</h3>
     * <pre>
     *   Path 1 — Singleton:  check cache, create once if absent
     *   Path 2 — Prototype:  always create a new instance (no caching)
     *   Path 3 — Custom:     delegate to Scope.get(name, objectFactory)
     * </pre>
     */
    private Object getScopedBean(BeanDefinition definition) {
        String beanName = definition.getBeanName();

        // Path 1: Singleton — the original behavior
        if (definition.isSingleton()) {
            Object existing = singletonMap.get(beanName);
            if (existing != null) {
                return existing;
            }
            return createBeanInstance(definition);
        }

        // Path 2: Prototype — new instance every time, no caching
        // Real Spring: AbstractBeanFactory.doGetBean() lines 346-358
        // Prototypes are NOT stored in singletonMap and have NO destruction callbacks
        if (definition.isPrototype()) {
            return createPrototypeInstance(definition);
        }

        // Path 3: Custom scope (request, session, etc.)
        // Real Spring: AbstractBeanFactory.doGetBean() lines 359-383
        String scopeName = definition.getScope();
        Scope scope = scopes.get(scopeName);
        if (scope == null) {
            throw new IllegalStateException(
                    "No Scope registered for scope name '" + scopeName
                            + "' for bean '" + beanName + "'");
        }

        try {
            return scope.get(beanName, () -> createPrototypeInstance(definition));
        } catch (IllegalStateException ex) {
            throw new ScopeNotActiveException(beanName, scopeName, ex);
        }
    }

    /**
     * Create a new bean instance with full DI lifecycle but WITHOUT caching
     * in singletonMap. Used for prototype and custom-scoped beans.
     */
    private Object createPrototypeInstance(BeanDefinition definition) {
        String beanName = definition.getBeanName();

        if (currentlyCreating.contains(beanName)) {
            throw new CircularDependencyException(
                    "Circular dependency detected: bean '" + beanName + "' is currently being created. "
                            + "Dependency chain includes: " + currentlyCreating);
        }

        currentlyCreating.add(beanName);
        try {
            ConstructorResolver resolver = new ConstructorResolver(this);
            Object instance = resolver.autowireConstructor(definition.getBeanClass());

            for (BeanPostProcessor bpp : beanPostProcessors) {
                instance = bpp.postProcessBeforeInitialization(instance, beanName);
            }
            for (BeanPostProcessor bpp : beanPostProcessors) {
                instance = bpp.postProcessAfterInitialization(instance, beanName);
            }

            return instance;
        } finally {
            currentlyCreating.remove(beanName);
        }
    }

    // ─── Ch24: Bean creation lifecycle (singleton path) ─────────────

    /**
     * Create a single bean with the full DI lifecycle:
     * constructor resolution → instantiate → BeanPostProcessor processing.
     *
     * Maps to: {@code AbstractAutowireCapableBeanFactory.createBean()} (line 489)
     *  → {@code doCreateBean()} (line 553)
     *    → {@code createBeanInstance()} — constructor resolution
     *    → {@code populateBean()} — @Autowired/@Value injection via BPP
     *    → {@code initializeBean()} — BPP postProcess hooks
     *
     * @param definition the bean definition to instantiate
     * @return the fully initialized bean instance
     */
    private Object createBeanInstance(BeanDefinition definition) {
        String beanName = definition.getBeanName();

        // Already created? Return existing singleton
        Object existing = singletonMap.get(beanName);
        if (existing != null) {
            return existing;
        }

        // Circular dependency detection
        if (currentlyCreating.contains(beanName)) {
            throw new CircularDependencyException(
                    "Circular dependency detected: bean '" + beanName + "' is currently being created. "
                            + "Dependency chain includes: " + currentlyCreating);
        }

        currentlyCreating.add(beanName);
        try {
            // Phase 1: Constructor resolution + instantiation
            ConstructorResolver resolver = new ConstructorResolver(this);
            Object instance = resolver.autowireConstructor(definition.getBeanClass());

            // Register early — before BPP processing — so that dependencies
            // resolved during field injection can find this bean.
            singletonMap.put(beanName, instance);

            // Phase 2: BeanPostProcessor.postProcessBeforeInitialization()
            // This is where @Autowired field/method injection happens
            for (BeanPostProcessor bpp : beanPostProcessors) {
                instance = bpp.postProcessBeforeInitialization(instance, beanName);
            }

            // Phase 3: BeanPostProcessor.postProcessAfterInitialization()
            // This is where AOP proxies would be created (future ch28)
            for (BeanPostProcessor bpp : beanPostProcessors) {
                instance = bpp.postProcessAfterInitialization(instance, beanName);
            }

            // Update singleton map in case a BPP replaced the instance (e.g., proxy)
            singletonMap.put(beanName, instance);

            return instance;
        } finally {
            currentlyCreating.remove(beanName);
        }
    }

    // ─── Ch24: Dependency resolution ────────────────────────────────

    /**
     * Resolve a dependency by type (and optional qualifier name).
     * Called by ConstructorResolver and InjectionMetadata elements.
     *
     * Maps to: {@code DefaultListableBeanFactory.resolveDependency()} (line 1484)
     * which delegates to {@code doResolveDependency()} (line 1556).
     *
     * <h3>Resolution strategy:</h3>
     * <ol>
     *   <li>Find all existing singletons matching the type</li>
     *   <li>Find all uninstantiated definitions matching the type → create them</li>
     *   <li>If qualifier specified → filter by bean name</li>
     *   <li>If single match → return it</li>
     *   <li>If zero matches → throw UnsatisfiedDependencyException</li>
     *   <li>If multiple matches → throw NoUniqueBeanException</li>
     * </ol>
     *
     * @param type      the required type
     * @param qualifier optional bean name qualifier (from @Qualifier), may be null
     * @return the resolved bean
     */
    public Object resolveDependency(Class<?> type, String qualifier) {
        Map<String, Object> candidates = new LinkedHashMap<>();

        // 1. Check existing singletons
        for (String name : beanNames) {
            Object bean = singletonMap.get(name);
            if (bean != null && type.isInstance(bean)) {
                candidates.put(name, bean);
            }
        }

        // 2. Check uninstantiated definitions
        for (var entry : beanDefinitions.entrySet()) {
            String name = entry.getKey();
            BeanDefinition def = entry.getValue();
            // Ch29: skip non-autowire-candidate beans (e.g., scoped proxy targets)
            if (!def.isAutowireCandidate()) {
                continue;
            }
            if (!singletonMap.containsKey(name)
                    && type.isAssignableFrom(def.getBeanClass())) {
                // Create the dependency on-demand (recursive)
                Object bean = getScopedBean(def);
                candidates.put(name, bean);
            }
        }

        // 3. Apply qualifier filter
        if (qualifier != null && !qualifier.isEmpty()) {
            Object qualified = candidates.get(qualifier);
            if (qualified != null) {
                return qualified;
            }
            throw new UnsatisfiedDependencyException(
                    "No bean named '" + qualifier + "' of type " + type.getName()
                            + " found. Available beans: " + candidates.keySet());
        }

        // 4. Single match
        if (candidates.size() == 1) {
            return candidates.values().iterator().next();
        }

        // 5. No match
        if (candidates.isEmpty()) {
            throw new UnsatisfiedDependencyException(
                    "No bean of type '" + type.getName() + "' is registered");
        }

        // 6. Multiple matches — ambiguous
        throw new NoUniqueBeanException(
                "Expected single bean of type '" + type.getName()
                        + "' but found " + candidates.size() + ": " + candidates.keySet()
                        + ". Use @Qualifier to disambiguate.");
    }

    // ─── Ch24: BeanPostProcessor management ─────────────────────────

    /**
     * Register a BeanPostProcessor to participate in bean creation lifecycle.
     *
     * Maps to: {@code AbstractBeanFactory.addBeanPostProcessor()} (line 348)
     */
    public void addBeanPostProcessor(BeanPostProcessor processor) {
        // Remove existing instance to avoid duplicates, then add at end
        beanPostProcessors.remove(processor);
        beanPostProcessors.add(processor);
    }

    /**
     * Return all registered BeanPostProcessors.
     */
    public List<BeanPostProcessor> getBeanPostProcessors() {
        return List.copyOf(beanPostProcessors);
    }

    // ─── Ch24: PropertySource management ────────────────────────────

    /**
     * Set the property source for @Value resolution.
     */
    public void setPropertySource(PropertySource propertySource) {
        this.propertySource = propertySource;
    }

    /**
     * Get the property source for @Value resolution.
     */
    public PropertySource getPropertySource() {
        return propertySource;
    }

    // ─── Ch28: Bean type introspection ─────────────────────────────

    /**
     * Get the type of a bean without forcing its instantiation.
     * Returns the bean's class if already instantiated, or the bean definition's class.
     *
     * Maps to: {@code AbstractBeanFactory.getType(String beanName)} (line 425)
     *
     * This is critical for AOP: the {@code AspectJAutoProxyCreator} needs to check
     * whether beans are @Aspect or Advisor types WITHOUT triggering creation of
     * every bean in the container.
     *
     * @param beanName the name of the bean
     * @return the bean's class, or null if not found
     */
    public Class<?> getType(String beanName) {
        Object bean = singletonMap.get(beanName);
        if (bean != null) {
            return bean.getClass();
        }
        BeanDefinition definition = beanDefinitions.get(beanName);
        if (definition != null) {
            return definition.getBeanClass();
        }
        return null;
    }

    // ─── Ch29: Scope registration ───────────────────────────────────

    /**
     * Register a custom scope under the given name.
     *
     * Maps to: {@code AbstractBeanFactory.registerScope()} (line 1063)
     * which validates that "singleton" and "prototype" are not used
     * (those are built-in and cannot be replaced).
     *
     * @param scopeName the scope name (e.g., "request", "session")
     * @param scope     the Scope implementation
     */
    public void registerScope(String scopeName, Scope scope) {
        if (BeanDefinition.SCOPE_SINGLETON.equals(scopeName)
                || BeanDefinition.SCOPE_PROTOTYPE.equals(scopeName)) {
            throw new IllegalArgumentException(
                    "Cannot replace built-in scope: " + scopeName);
        }
        scopes.put(scopeName, scope);
    }

    /**
     * Return the Scope registered under the given name, or {@code null}.
     */
    public Scope getRegisteredScope(String scopeName) {
        return scopes.get(scopeName);
    }

    // ─── Ch29: Scoped proxy registration ────────────────────────────

    /**
     * For beans with scoped proxy mode set, create a proxy bean definition
     * under the original name and rename the target to "scopedTarget.xxx".
     *
     * Maps to: {@code ScopedProxyUtils.createScopedProxy()} in
     * spring-aop/src/main/java/.../aop/scope/ScopedProxyUtils.java (line 58)
     *
     * The proxy is a singleton that delegates each method call to the
     * scope's current instance. This solves the lifecycle mismatch between
     * a singleton holding a reference to a request/session-scoped bean.
     */
    private void registerScopedProxies() {
        for (var entry : new ArrayList<>(beanDefinitions.entrySet())) {
            BeanDefinition def = entry.getValue();
            ScopedProxyMode proxyMode = def.getProxyMode();

            if (proxyMode == ScopedProxyMode.DEFAULT || proxyMode == ScopedProxyMode.NO) {
                continue;
            }

            String originalName = entry.getKey();
            String targetName = "scopedTarget." + originalName;

            // Rename the target bean definition
            beanDefinitions.remove(originalName);
            BeanDefinition targetDef = new BeanDefinition(targetName, def.getBeanClass());
            targetDef.setScope(def.getScope());
            targetDef.setProxyMode(ScopedProxyMode.NO); // target itself needs no proxy
            targetDef.setAutowireCandidate(false);       // only the proxy gets injected
            beanDefinitions.put(targetName, targetDef);
            // Update beanNames: replace the original name with the target name
            int idx = beanNames.indexOf(originalName);
            if (idx >= 0) {
                beanNames.set(idx, targetName);
            }
            if (!beanNames.contains(originalName)) {
                beanNames.add(originalName);
            }

            // Create the proxy as a pre-registered singleton
            Object proxy = createScopedProxy(targetName, def.getBeanClass(), proxyMode);
            singletonMap.put(originalName, proxy);
        }
    }

    /**
     * Create a scoped proxy for the given target bean.
     *
     * The proxy intercepts every method call and delegates to the current
     * scope's instance by calling {@code getBean(targetBeanName)}.
     *
     * Uses the AOP infrastructure from ch28 (ProxyFactory/CglibAopProxy/JdkDynamicAopProxy).
     *
     * @param targetBeanName the renamed target bean (e.g., "scopedTarget.cart")
     * @param targetClass    the target's class
     * @param proxyMode      INTERFACES or TARGET_CLASS
     * @return the proxy instance
     */
    private Object createScopedProxy(String targetBeanName, Class<?> targetClass,
                                     ScopedProxyMode proxyMode) {
        // The interceptor that delegates to the scope on every method call
        MethodInterceptor scopeInterceptor = invocation -> {
            Object target = getBean(targetBeanName);
            return invocation.getMethod().invoke(target, invocation.getArguments());
        };

        AdvisedSupport config = new AdvisedSupport();
        // Create a dummy target for proxy generation — the actual target
        // is resolved from the scope at invocation time
        try {
            Object dummyTarget = targetClass.getDeclaredConstructor().newInstance();
            config.setTarget(dummyTarget);
        } catch (Exception e) {
            throw new RuntimeException(
                    "Cannot create scoped proxy for " + targetClass.getName()
                            + ": no default constructor", e);
        }
        config.addAdvice(scopeInterceptor);

        if (proxyMode == ScopedProxyMode.INTERFACES) {
            for (Class<?> iface : targetClass.getInterfaces()) {
                config.addInterface(iface);
            }
            config.setProxyTargetClass(false);
            return new JdkDynamicAopProxy(config).getProxy();
        } else {
            config.setProxyTargetClass(true);
            return new CglibAopProxy(config).getProxy();
        }
    }

    // ─── Internal helpers ───────────────────────────────────────────

    /**
     * Find a bean definition whose class matches the given type.
     * Used during getBean(Class) to trigger on-demand creation.
     */
    private String findBeanDefinitionByType(Class<?> type) {
        for (var entry : beanDefinitions.entrySet()) {
            if (!singletonMap.containsKey(entry.getKey())
                    && type.isAssignableFrom(entry.getValue().getBeanClass())) {
                return entry.getKey();
            }
        }
        return null;
    }

    /**
     * Derive a bean name from a class: simple name with first letter lowercased.
     * e.g., UserController → "userController", SSEEmitter → "sSEEmitter"
     *
     * Real Spring equivalent: AnnotationBeanNameGenerator.buildDefaultBeanName()
     * which calls Introspector.decapitalize().
     */
    private String deriveBeanName(Class<?> clazz) {
        String simpleName = clazz.getSimpleName();
        if (simpleName.isEmpty()) {
            return simpleName;
        }
        return Character.toLowerCase(simpleName.charAt(0)) + simpleName.substring(1);
    }

    // ─── Ch28: AOP auto-registration ────────────────────────────────

    /**
     * Check if any bean definition or singleton carries @EnableAspectJAutoProxy
     * and auto-register the AspectJAutoProxyCreator BeanPostProcessor.
     *
     * Real Spring: @EnableAspectJAutoProxy is meta-annotated with
     * @Import(AspectJAutoProxyRegistrar.class) → registers
     * AnnotationAwareAspectJAutoProxyCreator as a BeanDefinition.
     *
     * We detect it directly and register the BPP instance.
     */
    private void autoRegisterAopProcessorIfNeeded() {
        EnableAspectJAutoProxy enableAop = null;

        // Check bean definition classes
        for (var def : beanDefinitions.values()) {
            enableAop = def.getBeanClass().getAnnotation(EnableAspectJAutoProxy.class);
            if (enableAop != null) break;
        }

        // Check existing singletons
        if (enableAop == null) {
            for (Object bean : singletonMap.values()) {
                enableAop = bean.getClass().getAnnotation(EnableAspectJAutoProxy.class);
                if (enableAop != null) break;
            }
        }

        if (enableAop != null) {
            boolean hasAopProcessor = beanPostProcessors.stream()
                    .anyMatch(bpp -> bpp instanceof AspectJAutoProxyCreator);
            if (!hasAopProcessor) {
                AspectJAutoProxyCreator creator = new AspectJAutoProxyCreator(this);
                creator.setProxyTargetClass(enableAop.proxyTargetClass());
                beanPostProcessors.add(creator);
            }
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/scan/ClasspathScanner.java` [MODIFIED]

```java
package com.simplespringmvc.scan;

import com.simplespringmvc.annotation.Component;
import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.injection.BeanDefinition;
import com.simplespringmvc.scope.ScopeAnnotation;

import java.beans.Introspector;
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Modifier;
import java.net.JarURLConnection;
import java.net.URL;
import java.util.Enumeration;
import java.util.LinkedHashSet;
import java.util.Set;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

/**
 * Scans the classpath for classes annotated with {@link Component} (directly or via
 * meta-annotation) and registers them as bean definitions in the {@link SimpleBeanContainer}.
 *
 * Maps to: {@code org.springframework.context.annotation.ClassPathBeanDefinitionScanner}
 * + {@code ClassPathScanningCandidateComponentProvider}
 *
 * <h3>End-to-end scanning flow (real framework):</h3>
 * <pre>
 *   ClassPathBeanDefinitionScanner.doScan("com.example")
 *     → findCandidateComponents("com.example")                     [inherited from parent]
 *       → scanCandidateComponents("com.example")
 *         → builds pattern: "classpath*:com/example/&#42;&#42;/&#42;.class"
 *         → PathMatchingResourcePatternResolver.getResources(pattern)
 *           → for each Resource (.class file):
 *             → MetadataReader (ASM-based, NO class loading)
 *             → isCandidateComponent(MetadataReader)               [filter chain]
 *             → isCandidateComponent(AnnotatedBeanDefinition)      [structural check]
 *     → for each candidate:
 *       → generate bean name (AnnotationBeanNameGenerator)
 *       → process @Lazy, @Primary, @DependsOn
 *       → check for conflicts
 *       → register BeanDefinition in registry
 * </pre>
 *
 * <h3>Our simplified flow (ch24 upgrade):</h3>
 * <pre>
 *   ClasspathScanner.scan("com.example")
 *     → findCandidateComponents("com.example")
 *       → ClassLoader.getResources("com/example")
 *       → for each URL:
 *         → file: protocol → walk directory tree
 *         → jar: protocol  → iterate JarFile entries
 *       → for each .class file:
 *         → Class.forName() (loads the class — simpler than ASM)
 *         → isCandidateComponent(Class)
 *     → for each candidate:
 *       → generate bean name (Introspector.decapitalize)
 *       → register BeanDefinition in container     ← ch24: was direct instantiation
 *     → container.refresh()                         ← ch24: instantiate with DI
 * </pre>
 *
 * <h3>Ch24 change: from instantiation to registration</h3>
 * Previously, the scanner instantiated beans directly via no-arg constructor.
 * Now it registers BeanDefinitions (class metadata), and the container's refresh()
 * instantiates them with constructor injection and @Autowired/@Value processing.
 * This is how the real framework works — scanning produces BeanDefinitions,
 * not live instances.
 *
 * <h3>Key simplification: Class.forName() vs ASM MetadataReader</h3>
 * The real framework uses ASM to read class bytecode WITHOUT loading the class.
 * This avoids triggering static initializers and is much faster when scanning
 * thousands of classes. We use Class.forName() with {@code initialize=false}
 * to avoid static initializers while keeping the code simple.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>Uses Class.forName() instead of ASM MetadataReader</li>
 *   <li>No include/exclude TypeFilter chain</li>
 *   <li>No @Conditional evaluation</li>
 *   <li>No component index (META-INF/spring.components) optimization</li>
 *   <li>No @Lazy/@Primary/@DependsOn processing</li>
 *   <li>No conflict detection — skips already-registered beans</li>
 * </ul>
 */
public class ClasspathScanner {

    private final SimpleBeanContainer beanContainer;

    public ClasspathScanner(SimpleBeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Scan the given base packages for component classes, register them as
     * bean definitions, and trigger container refresh to instantiate with DI.
     *
     * Maps to: {@code ClassPathBeanDefinitionScanner.scan(String...)} (line 251)
     *
     * Ch24 change: now calls container.refresh() after registration to trigger
     * the full DI lifecycle (constructor injection, @Autowired, @Value).
     *
     * @param basePackages the packages to scan (e.g., "com.example.controllers")
     * @return the number of newly registered beans
     */
    public int scan(String... basePackages) {
        int count = 0;
        for (String basePackage : basePackages) {
            count += doScan(basePackage);
        }

        // Ch24: trigger container refresh to instantiate with DI
        beanContainer.refresh();

        return count;
    }

    /**
     * Perform the actual scan for a single base package.
     *
     * Maps to: {@code ClassPathBeanDefinitionScanner.doScan(String...)} (line 272)
     *
     * Ch24 change: registers BeanDefinitions instead of instantiating directly.
     * The real framework returns {@code Set<BeanDefinitionHolder>}. We register
     * definitions in the container.
     */
    private int doScan(String basePackage) {
        Set<Class<?>> candidates = findCandidateComponents(basePackage);
        int count = 0;

        for (Class<?> clazz : candidates) {
            String beanName = deriveBeanName(clazz);

            // Skip if already registered — prevents double-registration when
            // a bean was registered manually before scanning.
            // Real framework does conflict checking in checkCandidate() (line 335).
            if (!beanContainer.containsBean(beanName)) {
                // Ch29: detect @ScopeAnnotation and set scope metadata
                BeanDefinition definition = new BeanDefinition(beanName, clazz);
                ScopeAnnotation scopeAnn = clazz.getAnnotation(ScopeAnnotation.class);
                if (scopeAnn != null) {
                    definition.setScope(scopeAnn.value());
                    definition.setProxyMode(scopeAnn.proxyMode());
                }
                beanContainer.registerBeanDefinition(definition);
                count++;
            }
        }

        return count;
    }

    /**
     * Find all classes in the given package that are annotated with @Component
     * (directly or via meta-annotation).
     *
     * Maps to: {@code ClassPathScanningCandidateComponentProvider.findCandidateComponents()}
     * (line 312) → {@code scanCandidateComponents()} (line 446)
     *
     * The real version builds a resource pattern like "classpath*:com/example/&#42;&#42;/&#42;.class"
     * and uses PathMatchingResourcePatternResolver to find resources. We use
     * ClassLoader.getResources() which gives us the root directories/JARs for
     * the package, then walk them ourselves.
     */
    private Set<Class<?>> findCandidateComponents(String basePackage) {
        Set<Class<?>> candidates = new LinkedHashSet<>();
        String packagePath = basePackage.replace('.', '/');

        try {
            ClassLoader classLoader = getClassLoader();
            Enumeration<URL> resources = classLoader.getResources(packagePath);

            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                String protocol = resource.getProtocol();

                if ("file".equals(protocol)) {
                    // Filesystem: compiled classes in a directory (typical for development)
                    File directory = new File(resource.toURI());
                    scanDirectory(directory, basePackage, candidates);
                } else if ("jar".equals(protocol)) {
                    // JAR file: classes packaged in a jar (typical for dependencies)
                    scanJar(resource, packagePath, candidates);
                }
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to scan package: " + basePackage, e);
        }

        return candidates;
    }

    /**
     * Recursively scan a filesystem directory for .class files.
     *
     * Maps to: {@code PathMatchingResourcePatternResolver.doFindPathMatchingFileResources()}
     * (line 1011) which uses {@code Files.walk()} with FOLLOW_LINKS.
     *
     * We use simple recursion through File.listFiles() — more readable for the
     * educational purpose, and equivalent for small codebases.
     *
     * @param directory   the directory to scan
     * @param packageName the Java package corresponding to this directory
     * @param candidates  accumulator for discovered component classes
     */
    private void scanDirectory(File directory, String packageName, Set<Class<?>> candidates) {
        File[] files = directory.listFiles();
        if (files == null) {
            return;
        }

        for (File file : files) {
            if (file.isDirectory()) {
                // Recurse into subdirectories — each subdirectory is a sub-package
                scanDirectory(file, packageName + "." + file.getName(), candidates);
            } else if (file.getName().endsWith(".class")) {
                // Convert filename to fully-qualified class name
                // e.g., "UserController.class" → "com.example.UserController"
                String className = packageName + "."
                        + file.getName().substring(0, file.getName().length() - ".class".length());
                loadAndCheck(className, candidates);
            }
        }
    }

    /**
     * Scan a JAR file for .class files under the given package path.
     *
     * Maps to: {@code PathMatchingResourcePatternResolver.doFindPathMatchingJarResources()}
     * (line 864) which opens a JarURLConnection, iterates entries, and matches
     * against the sub-pattern using AntPathMatcher.
     *
     * We simplify by iterating all entries and checking the prefix.
     */
    private void scanJar(URL jarUrl, String packagePath, Set<Class<?>> candidates) throws IOException {
        JarURLConnection connection = (JarURLConnection) jarUrl.openConnection();
        try (JarFile jarFile = connection.getJarFile()) {
            Enumeration<JarEntry> entries = jarFile.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                String entryName = entry.getName();

                // Only process .class files under our target package
                if (entryName.startsWith(packagePath + "/") && entryName.endsWith(".class")) {
                    // Convert JAR entry path to class name
                    // e.g., "com/example/UserController.class" → "com.example.UserController"
                    String className = entryName
                            .substring(0, entryName.length() - ".class".length())
                            .replace('/', '.');
                    loadAndCheck(className, candidates);
                }
            }
        }
    }

    /**
     * Load a class by name and add it to the candidate set if it passes the
     * component check.
     *
     * Uses {@code Class.forName(name, false, classLoader)} — the {@code false}
     * parameter prevents static initializer execution, similar (in spirit) to
     * the real framework's ASM-based MetadataReader which never loads the class.
     */
    private void loadAndCheck(String className, Set<Class<?>> candidates) {
        try {
            Class<?> clazz = Class.forName(className, false, getClassLoader());
            if (isCandidateComponent(clazz)) {
                candidates.add(clazz);
            }
        } catch (ClassNotFoundException | NoClassDefFoundError e) {
            // Skip classes that can't be loaded — may have unsatisfied dependencies.
            // The real framework logs at TRACE level and continues.
        }
    }

    /**
     * Check if a class is eligible for component registration.
     *
     * Maps to two checks in the real framework:
     * <ol>
     *   <li>{@code isCandidateComponent(MetadataReader)} (line 533) — filter-based:
     *       checks exclude filters, then include filters (default: @Component)</li>
     *   <li>{@code isCandidateComponent(AnnotatedBeanDefinition)} (line 571) — structural:
     *       must be independent (top-level or static nested) and concrete</li>
     * </ol>
     *
     * We combine both checks into one method.
     */
    private boolean isCandidateComponent(Class<?> clazz) {
        // Filter check: must have @Component (directly or via meta-annotation)
        // Real framework uses AnnotationTypeFilter(Component.class) as the default include filter
        if (!MergedAnnotationUtils.hasAnnotation(clazz, Component.class)) {
            return false;
        }

        // Structural check 1: must not be an interface or abstract class
        // Real framework: AnnotatedBeanDefinition.getMetadata().isConcrete()
        if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
            return false;
        }

        // Structural check 2: must be independent — top-level or static nested
        // Real framework: AnnotatedBeanDefinition.getMetadata().isIndependent()
        // Inner (non-static) classes need an enclosing instance and can't be
        // independently instantiated.
        if (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers())) {
            return false;
        }

        return true;
    }

    /**
     * Derive a bean name from the class: short name with first letter decapitalized.
     *
     * Maps to: {@code AnnotationBeanNameGenerator.buildDefaultBeanName()} (line 150)
     * which calls {@code Introspector.decapitalize(shortClassName)}.
     *
     * Uses JDK's {@code Introspector.decapitalize()} to match the real framework's
     * behavior — notably, if the first TWO characters are uppercase (e.g., "URL"),
     * the name is kept as-is ("URL" not "uRL").
     *
     * Examples:
     * <ul>
     *   <li>UserController → "userController"</li>
     *   <li>SSEEmitter → "SSEEmitter" (first two chars uppercase — keep as-is)</li>
     *   <li>HTMLParser → "HTMLParser" (first two chars uppercase — keep as-is)</li>
     * </ul>
     */
    private String deriveBeanName(Class<?> clazz) {
        String shortName = clazz.getSimpleName();
        return Introspector.decapitalize(shortName);
    }

    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return cl != null ? cl : getClass().getClassLoader();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java` [MODIFIED]

```java
package com.simplespringmvc.server;

import com.simplespringmvc.scope.RequestContextFilter;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import com.simplespringmvc.websocket.SimpleWebSocketHandlerRegistry;
import com.simplespringmvc.websocket.WebSocketConfigurer;
import jakarta.servlet.MultipartConfigElement;
import jakarta.websocket.DeploymentException;
import jakarta.websocket.server.ServerContainer;
import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;

import java.io.File;

/**
 * Embeds Apache Tomcat to receive HTTP requests and route them through
 * the {@link SimpleDispatcherServlet}. Optionally initializes the WebSocket
 * container for persistent bidirectional connections.
 *
 * Maps to: Spring Boot's {@code TomcatServletWebServerFactory} which
 * programmatically creates a Tomcat instance, configures a context,
 * and registers the DispatcherServlet.
 *
 * The real Spring Boot registration chain:
 * <pre>
 *   TomcatServletWebServerFactory.getWebServer()
 *     → Tomcat tomcat = new Tomcat()
 *     → prepareContext(tomcat.getHost(), initializers)
 *     → new TomcatWebServer(tomcat)
 *
 *   ServletWebServerApplicationContext.onRefresh()
 *     → createWebServer()
 *     → factory.getWebServer(getSelfInitializer())
 *     → initializers register DispatcherServlet via servletContext.addServlet()
 * </pre>
 *
 * We simplify all of that into a single class that:
 * <ol>
 *   <li>Creates a Tomcat instance</li>
 *   <li>Adds a context at the root path</li>
 *   <li>Registers SimpleDispatcherServlet mapped to "/"</li>
 *   <li>Optionally configures multipart support for file uploads</li>
 *   <li>Optionally initializes WebSocket support via {@link WebSocketConfigurer}</li>
 *   <li>Starts the server</li>
 * </ol>
 *
 * Simplifications vs real Spring Boot:
 * <ul>
 *   <li>No auto-configuration</li>
 *   <li>No customizer callbacks</li>
 *   <li>No graceful shutdown</li>
 *   <li>No SSL/HTTP2 configuration</li>
 *   <li>No compression, access logs, etc.</li>
 * </ul>
 */
public class EmbeddedTomcat {

    private final Tomcat tomcat;
    private final SimpleDispatcherServlet dispatcherServlet;
    private final Context context;
    private final org.apache.catalina.Wrapper servletWrapper;

    /**
     * Creates an embedded Tomcat that routes all requests through the given servlet.
     *
     * @param port              the port to listen on (use 0 for a random available port)
     * @param dispatcherServlet the front-controller servlet that handles all requests
     */
    public EmbeddedTomcat(int port, SimpleDispatcherServlet dispatcherServlet) {
        this.dispatcherServlet = dispatcherServlet;
        this.tomcat = new Tomcat();
        tomcat.setPort(port);

        // Tomcat needs a base directory for temp files. Use a temp dir so we
        // don't pollute the project directory.
        tomcat.setBaseDir(new File(System.getProperty("java.io.tmpdir"), "tomcat-simple")
                .getAbsolutePath());

        // Create the root context — equivalent to a web application at "/"
        // The empty docBase means no static files directory.
        this.context = tomcat.addContext("", null);

        // Register our DispatcherServlet.
        // Tomcat.addServlet() is the programmatic equivalent of a <servlet> entry
        // in web.xml. We map it to "/" so it receives all requests (the default servlet).
        this.servletWrapper = Tomcat.addServlet(context, "dispatcher", dispatcherServlet);
        context.addServletMappingDecoded("/", "dispatcher");

        // ch22: Enable async support so SSE (and any future async features) can
        // call request.startAsync(). Without this, startAsync() throws
        // IllegalStateException. Maps to: Spring Boot's
        // ServletRegistrationBean.setAsyncSupported(true) which is enabled by
        // default for the DispatcherServlet registration.
        servletWrapper.setAsyncSupported(true);

        // ch29: Register RequestContextFilter to bind request/session attributes
        // to the current thread, enabling request/session scoped beans.
        // Maps to: Spring Boot's RequestContextFilter auto-configuration.
        // Uses Tomcat's FilterDef/FilterMap API since the ServletContext isn't
        // available until after start().
        org.apache.tomcat.util.descriptor.web.FilterDef filterDef =
                new org.apache.tomcat.util.descriptor.web.FilterDef();
        filterDef.setFilterName("requestContextFilter");
        filterDef.setFilterClass(RequestContextFilter.class.getName());
        filterDef.setAsyncSupported("true");
        context.addFilterDef(filterDef);

        org.apache.tomcat.util.descriptor.web.FilterMap filterMap =
                new org.apache.tomcat.util.descriptor.web.FilterMap();
        filterMap.setFilterName("requestContextFilter");
        filterMap.addURLPattern("/*");
        context.addFilterMap(filterMap);

        // Enable the connector so Tomcat will actually listen on the port.
        tomcat.getConnector();
    }

    /**
     * Enable multipart/form-data support for file uploads.
     *
     * This configures the Servlet container to parse multipart requests via
     * the Servlet 3.0 {@code Part} API. Without this configuration,
     * {@code request.getParts()} will throw an exception.
     *
     * Maps to: Spring Boot's {@code MultipartAutoConfiguration} which creates
     * a {@code MultipartConfigElement} and applies it to the DispatcherServlet
     * registration via {@code ServletRegistrationBean.setMultipartConfig()}.
     *
     * <h3>ch25 Enhancement:</h3>
     * Must be called BEFORE {@link #start()} for the configuration to take effect.
     *
     * @param location     the directory for temporary file storage during upload
     * @param maxFileSize  the maximum size allowed for a single uploaded file (-1 for unlimited)
     * @param maxRequestSize the maximum size of the entire multipart request (-1 for unlimited)
     * @param fileSizeThreshold the size threshold for writing to disk (0 writes immediately)
     */
    public void enableMultipart(String location, long maxFileSize,
                                 long maxRequestSize, int fileSizeThreshold) {
        servletWrapper.setMultipartConfigElement(
                new MultipartConfigElement(location, maxFileSize, maxRequestSize, fileSizeThreshold));
    }

    /**
     * Enable multipart support with sensible defaults.
     *
     * <ul>
     *   <li>Location: system temp directory</li>
     *   <li>Max file size: 10MB</li>
     *   <li>Max request size: 50MB</li>
     *   <li>File size threshold: 0 (write to disk immediately)</li>
     * </ul>
     */
    public void enableMultipart() {
        String tempDir = System.getProperty("java.io.tmpdir");
        enableMultipart(tempDir, 10 * 1024 * 1024, 50 * 1024 * 1024, 0);
    }

    /**
     * Registers WebSocket handlers via the given configurer.
     *
     * This initializes Tomcat's WebSocket container (WsSci) and deploys
     * all handlers registered through the configurer's registry.
     *
     * Must be called BEFORE {@link #start()}.
     *
     * Maps to: Spring's {@code WebSocketConfigurationSupport} which creates a
     * {@code ServletWebSocketHandlerRegistry}, passes it to all {@code WebSocketConfigurer}
     * beans, and produces a {@code HandlerMapping} for WebSocket upgrade requests.
     *
     * We simplify by registering directly with the Jakarta ServerContainer
     * instead of creating a HandlerMapping.
     *
     * @param configurer the callback that registers WebSocket handlers
     */
    public void registerWebSocketHandlers(WebSocketConfigurer configurer) {
        // Initialize Tomcat's WebSocket support by adding the WsSci
        // (WebSocket Server Container Initializer) to the context.
        // This makes the ServerContainer available via the ServletContext attribute.
        context.addServletContainerInitializer(
                new org.apache.tomcat.websocket.server.WsSci(), null);

        // Create a registry and let the configurer populate it
        SimpleWebSocketHandlerRegistry registry = new SimpleWebSocketHandlerRegistry();
        configurer.registerWebSocketHandlers(registry);

        // Store for deployment after Tomcat starts (WsSci needs to run first)
        context.addLifecycleListener(event -> {
            if ("after_start".equals(event.getType())) {
                try {
                    ServerContainer serverContainer = (ServerContainer) context.getServletContext()
                            .getAttribute(ServerContainer.class.getName());
                    if (serverContainer != null) {
                        registry.deploy(serverContainer);
                    }
                } catch (DeploymentException ex) {
                    throw new RuntimeException("Failed to deploy WebSocket endpoints", ex);
                }
            }
        });
    }

    /**
     * Starts the embedded Tomcat server.
     *
     * After this call, the server is accepting HTTP connections on {@link #getPort()}.
     */
    public void start() throws Exception {
        tomcat.start();
    }

    /**
     * Stops the embedded Tomcat server and releases all resources.
     */
    public void stop() throws Exception {
        tomcat.stop();
        tomcat.destroy();
    }

    /**
     * Returns the port the server is listening on.
     *
     * If the server was created with port 0 (random), this returns the actual
     * allocated port after {@link #start()} has been called.
     */
    public int getPort() {
        return tomcat.getConnector().getLocalPort();
    }

    /**
     * Returns the DispatcherServlet this server is routing requests to.
     */
    public SimpleDispatcherServlet getDispatcherServlet() {
        return dispatcherServlet;
    }

    /**
     * Blocks the calling thread until the server shuts down.
     * Useful for running the server as a standalone application.
     */
    public void await() {
        tomcat.getServer().await();
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/scope/ScopeTest.java` [NEW]

```java
package com.simplespringmvc.scope;

import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.injection.BeanDefinition;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for bean scope support: prototype, request, and session scopes.
 *
 * Tests the three-way scope branching in SimpleBeanContainer.getBean(),
 * the Scope SPI, and the RequestContextHolder thread-local mechanism.
 */
class ScopeTest {

    // ─── Test helpers ────────────────────────────────────────────────

    /** A simple bean class for testing. Must have a no-arg constructor. */
    public static class Counter {
        private static final AtomicInteger instances = new AtomicInteger(0);
        private final int id;

        public Counter() {
            this.id = instances.incrementAndGet();
        }

        public int getId() {
            return id;
        }

        public static void resetCount() {
            instances.set(0);
        }
    }

    /** An interface for testing JDK proxy-based scoped proxies. */
    public interface Greeter {
        String greet();
    }

    /** Implementation of Greeter for proxy testing. */
    public static class SimpleGreeter implements Greeter {
        private static final AtomicInteger instances = new AtomicInteger(0);
        private final int id;

        public SimpleGreeter() {
            this.id = instances.incrementAndGet();
        }

        @Override
        public String greet() {
            return "Hello from greeter #" + id;
        }

        public static void resetCount() {
            instances.set(0);
        }
    }

    /** A simple in-memory Scope implementation for unit testing. */
    public static class MapBackedScope implements Scope {
        private final Map<String, Object> store = new ConcurrentHashMap<>();
        private final Map<String, Runnable> destructionCallbacks = new ConcurrentHashMap<>();

        @Override
        public Object get(String name, Supplier<?> objectFactory) {
            return store.computeIfAbsent(name, k -> objectFactory.get());
        }

        @Override
        public Object remove(String name) {
            destructionCallbacks.remove(name);
            return store.remove(name);
        }

        @Override
        public void registerDestructionCallback(String name, Runnable callback) {
            destructionCallbacks.put(name, callback);
        }

        public void clear() {
            store.clear();
            destructionCallbacks.clear();
        }

        public int size() {
            return store.size();
        }
    }

    @BeforeEach
    void resetCounters() {
        Counter.resetCount();
        SimpleGreeter.resetCount();
    }

    // ─── Prototype scope tests ──────────────────────────────────────

    @Nested
    class PrototypeScope {

        @Test
        void shouldCreateNewInstance_WhenPrototypeBeanRequested() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("prototype");
            container.registerBeanDefinition(def);
            container.refresh();

            Counter c1 = (Counter) container.getBean("counter");
            Counter c2 = (Counter) container.getBean("counter");

            assertThat(c1).isNotSameAs(c2);
            assertThat(c1.getId()).isNotEqualTo(c2.getId());
        }

        @Test
        void shouldNotCachePrototype_InSingletonMap() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("prototype");
            container.registerBeanDefinition(def);
            container.refresh();

            container.getBean("counter");
            container.getBean("counter");
            container.getBean("counter");

            // The prototype should not appear in getBeansOfType (which reads singletonMap)
            assertThat(container.getBeansOfType(Counter.class)).isEmpty();
        }

        @Test
        void shouldNotInstantiatePrototype_DuringRefresh() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("prototype");
            container.registerBeanDefinition(def);
            container.refresh();

            // No instance should have been created yet
            assertThat(Counter.instances.get()).isZero();
        }
    }

    // ─── Custom scope tests ─────────────────────────────────────────

    @Nested
    class CustomScope {

        @Test
        void shouldDelegateToScope_WhenCustomScopedBeanRequested() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            MapBackedScope scope = new MapBackedScope();
            container.registerScope("custom", scope);

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("custom");
            container.registerBeanDefinition(def);
            container.refresh();

            Counter c1 = (Counter) container.getBean("counter");
            Counter c2 = (Counter) container.getBean("counter");

            // Same instance from the scope (scope caches it)
            assertThat(c1).isSameAs(c2);
            assertThat(scope.size()).isEqualTo(1);
        }

        @Test
        void shouldCreateNewInstance_WhenScopeCleared() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            MapBackedScope scope = new MapBackedScope();
            container.registerScope("custom", scope);

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("custom");
            container.registerBeanDefinition(def);
            container.refresh();

            Counter c1 = (Counter) container.getBean("counter");
            scope.clear();  // Simulate scope expiration (e.g., request complete)
            Counter c2 = (Counter) container.getBean("counter");

            assertThat(c1).isNotSameAs(c2);
            assertThat(c1.getId()).isNotEqualTo(c2.getId());
        }

        @Test
        void shouldThrowException_WhenScopeNotRegistered() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("nonexistent");
            container.registerBeanDefinition(def);
            container.refresh();

            assertThatThrownBy(() -> container.getBean("counter"))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No Scope registered")
                    .hasMessageContaining("nonexistent");
        }

        @Test
        void shouldRejectRegistration_WhenScopeNameIsSingleton() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            assertThatThrownBy(() -> container.registerScope("singleton", new MapBackedScope()))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Cannot replace built-in scope");
        }

        @Test
        void shouldRejectRegistration_WhenScopeNameIsPrototype() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            assertThatThrownBy(() -> container.registerScope("prototype", new MapBackedScope()))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Cannot replace built-in scope");
        }
    }

    // ─── Singleton scope tests (regression) ─────────────────────────

    @Nested
    class SingletonScope {

        @Test
        void shouldReturnSameInstance_WhenSingletonBeanRequested() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            // default scope is singleton
            container.registerBeanDefinition(def);
            container.refresh();

            Counter c1 = (Counter) container.getBean("counter");
            Counter c2 = (Counter) container.getBean("counter");

            assertThat(c1).isSameAs(c2);
        }

        @Test
        void shouldInstantiateSingleton_DuringRefresh() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            container.registerBeanDefinition("counter", Counter.class);
            container.refresh();

            assertThat(Counter.instances.get()).isEqualTo(1);
        }
    }

    // ─── BeanDefinition scope metadata tests ────────────────────────

    @Nested
    class BeanDefinitionScopeMetadata {

        @Test
        void shouldDefaultToSingleton_WhenNoScopeSet() {
            BeanDefinition def = new BeanDefinition("test", Counter.class);
            assertThat(def.isSingleton()).isTrue();
            assertThat(def.isPrototype()).isFalse();
            assertThat(def.getScope()).isEqualTo("singleton");
        }

        @Test
        void shouldDetectPrototype_WhenScopeSetToPrototype() {
            BeanDefinition def = new BeanDefinition("test", Counter.class);
            def.setScope("prototype");
            assertThat(def.isPrototype()).isTrue();
            assertThat(def.isSingleton()).isFalse();
        }

        @Test
        void shouldDefaultToSingleton_WhenScopeSetToNull() {
            BeanDefinition def = new BeanDefinition("test", Counter.class);
            def.setScope(null);
            assertThat(def.isSingleton()).isTrue();
        }
    }

    // ─── RequestContextHolder tests ─────────────────────────────────

    @Nested
    class RequestContextHolderTest {

        @Test
        void shouldReturnNull_WhenNoAttributesBound() {
            RequestContextHolder.resetRequestAttributes();
            assertThat(RequestContextHolder.getRequestAttributes()).isNull();
        }

        @Test
        void shouldThrowException_WhenCurrentRequestAttributesCalledOutsideRequest() {
            RequestContextHolder.resetRequestAttributes();
            assertThatThrownBy(RequestContextHolder::currentRequestAttributes)
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No thread-bound request found");
        }

        @Test
        void shouldBindAndRetrieve_WhenRequestAttributesSet() {
            RequestAttributes attrs = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs);
            try {
                assertThat(RequestContextHolder.getRequestAttributes()).isSameAs(attrs);
                assertThat(RequestContextHolder.currentRequestAttributes()).isSameAs(attrs);
            } finally {
                RequestContextHolder.resetRequestAttributes();
            }
        }
    }

    // ─── RequestScope tests ─────────────────────────────────────────

    @Nested
    class RequestScopeTest {

        @Test
        void shouldCacheWithinSameRequest_WhenRequestScopeUsed() {
            RequestScope scope = new RequestScope();
            MockRequestAttributes attrs = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs);
            try {
                AtomicInteger count = new AtomicInteger(0);
                Object obj1 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
                Object obj2 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());

                assertThat(obj1).isSameAs(obj2);
                assertThat(count.get()).isEqualTo(1); // factory called only once
            } finally {
                RequestContextHolder.resetRequestAttributes();
            }
        }

        @Test
        void shouldCreateNewInstance_WhenDifferentRequest() {
            RequestScope scope = new RequestScope();
            AtomicInteger count = new AtomicInteger(0);

            // Request 1
            MockRequestAttributes attrs1 = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs1);
            Object obj1 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
            RequestContextHolder.resetRequestAttributes();

            // Request 2
            MockRequestAttributes attrs2 = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs2);
            Object obj2 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
            RequestContextHolder.resetRequestAttributes();

            assertThat(obj1).isNotSameAs(obj2);
            assertThat(count.get()).isEqualTo(2);
        }

        @Test
        void shouldRemoveBean_WhenRemoveCalled() {
            RequestScope scope = new RequestScope();
            MockRequestAttributes attrs = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs);
            try {
                scope.get("bean1", () -> "original");
                Object removed = scope.remove("bean1");
                assertThat(removed).isEqualTo("original");

                // After removal, a new instance is created
                Object fresh = scope.get("bean1", () -> "replacement");
                assertThat(fresh).isEqualTo("replacement");
            } finally {
                RequestContextHolder.resetRequestAttributes();
            }
        }
    }

    // ─── SessionScope tests ─────────────────────────────────────────

    @Nested
    class SessionScopeTest {

        @Test
        void shouldCacheAcrossRequests_WhenSameSession() {
            SessionScope scope = new SessionScope();
            AtomicInteger count = new AtomicInteger(0);

            // Shared session store
            Map<String, Object> sessionStore = new ConcurrentHashMap<>();
            Object sessionMutex = new Object();

            // Request 1 (same session)
            MockRequestAttributes attrs1 = new MockRequestAttributes(sessionStore, sessionMutex);
            RequestContextHolder.setRequestAttributes(attrs1);
            Object obj1 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
            RequestContextHolder.resetRequestAttributes();

            // Request 2 (same session)
            MockRequestAttributes attrs2 = new MockRequestAttributes(sessionStore, sessionMutex);
            RequestContextHolder.setRequestAttributes(attrs2);
            Object obj2 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
            RequestContextHolder.resetRequestAttributes();

            assertThat(obj1).isSameAs(obj2);
            assertThat(count.get()).isEqualTo(1);
        }

        @Test
        void shouldCreateNewInstance_WhenDifferentSession() {
            SessionScope scope = new SessionScope();
            AtomicInteger count = new AtomicInteger(0);

            // Session 1
            MockRequestAttributes attrs1 = new MockRequestAttributes(
                    new ConcurrentHashMap<>(), new Object());
            RequestContextHolder.setRequestAttributes(attrs1);
            Object obj1 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
            RequestContextHolder.resetRequestAttributes();

            // Session 2 (different session store)
            MockRequestAttributes attrs2 = new MockRequestAttributes(
                    new ConcurrentHashMap<>(), new Object());
            RequestContextHolder.setRequestAttributes(attrs2);
            Object obj2 = scope.get("bean1", () -> "instance-" + count.incrementAndGet());
            RequestContextHolder.resetRequestAttributes();

            assertThat(obj1).isNotSameAs(obj2);
            assertThat(count.get()).isEqualTo(2);
        }

        @Test
        void shouldReturnSessionId_AsConversationId() {
            SessionScope scope = new SessionScope();
            MockRequestAttributes attrs = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs);
            try {
                assertThat(scope.getConversationId()).isEqualTo("mock-session-id");
            } finally {
                RequestContextHolder.resetRequestAttributes();
            }
        }
    }

    // ─── Scoped proxy tests ─────────────────────────────────────────

    @Nested
    class ScopedProxyTest {

        @Test
        void shouldCreateScopedProxy_WhenProxyModeTargetClass() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            MapBackedScope scope = new MapBackedScope();
            container.registerScope("custom", scope);

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("custom");
            def.setProxyMode(ScopedProxyMode.TARGET_CLASS);
            container.registerBeanDefinition(def);
            container.refresh();

            // The proxy is registered as "counter" (singleton)
            Object proxy = container.getBean("counter");
            assertThat(proxy).isNotNull();
            assertThat(proxy).isInstanceOf(Counter.class);

            // The target bean definition should be renamed
            assertThat(container.containsBeanDefinition("scopedTarget.counter")).isTrue();
        }

        @Test
        void shouldDelegateToScope_WhenScopedProxyMethodCalled() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            MapBackedScope scope = new MapBackedScope();
            container.registerScope("custom", scope);

            BeanDefinition def = new BeanDefinition("counter", Counter.class);
            def.setScope("custom");
            def.setProxyMode(ScopedProxyMode.TARGET_CLASS);
            container.registerBeanDefinition(def);
            container.refresh();

            Counter proxy = (Counter) container.getBean("counter");
            int id1 = proxy.getId();

            // Clear scope — new instance created on next call
            scope.clear();
            int id2 = proxy.getId();

            assertThat(id1).isNotEqualTo(id2);
        }

        @Test
        void shouldCreateJdkProxy_WhenProxyModeInterfaces() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            MapBackedScope scope = new MapBackedScope();
            container.registerScope("custom", scope);

            BeanDefinition def = new BeanDefinition("greeter", SimpleGreeter.class);
            def.setScope("custom");
            def.setProxyMode(ScopedProxyMode.INTERFACES);
            container.registerBeanDefinition(def);
            container.refresh();

            Object proxy = container.getBean("greeter");
            assertThat(proxy).isInstanceOf(Greeter.class);

            Greeter greeter = (Greeter) proxy;
            String greeting1 = greeter.greet();
            assertThat(greeting1).startsWith("Hello from greeter #");

            // Clear scope
            scope.clear();
            String greeting2 = greeter.greet();
            assertThat(greeting1).isNotEqualTo(greeting2);
        }
    }

    // ─── Mock RequestAttributes ─────────────────────────────────────

    /**
     * Simple mock RequestAttributes for unit testing scopes without Servlet API.
     */
    public static class MockRequestAttributes implements RequestAttributes {
        private final Map<String, Object> requestAttrs = new ConcurrentHashMap<>();
        private final Map<String, Object> sessionAttrs;
        private final Object sessionMutex;

        MockRequestAttributes() {
            this(new ConcurrentHashMap<>(), new Object());
        }

        MockRequestAttributes(Map<String, Object> sessionStore, Object sessionMutex) {
            this.sessionAttrs = sessionStore;
            this.sessionMutex = sessionMutex;
        }

        @Override
        public Object getAttribute(String name, int scope) {
            return (scope == SCOPE_REQUEST) ? requestAttrs.get(name) : sessionAttrs.get(name);
        }

        @Override
        public void setAttribute(String name, Object value, int scope) {
            if (scope == SCOPE_REQUEST) {
                requestAttrs.put(name, value);
            } else {
                sessionAttrs.put(name, value);
            }
        }

        @Override
        public void removeAttribute(String name, int scope) {
            if (scope == SCOPE_REQUEST) {
                requestAttrs.remove(name);
            } else {
                sessionAttrs.remove(name);
            }
        }

        @Override
        public void registerDestructionCallback(String name, Runnable callback, int scope) {
            // No-op for testing
        }

        @Override
        public String getSessionId() {
            return "mock-session-id";
        }

        @Override
        public Object getSessionMutex() {
            return sessionMutex;
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/scope/integration/ScopeIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.scope.integration;

import com.simplespringmvc.annotation.Component;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.injection.BeanDefinition;
import com.simplespringmvc.scope.RequestAttributes;
import com.simplespringmvc.scope.RequestContextHolder;
import com.simplespringmvc.scope.RequestScope;
import com.simplespringmvc.scope.Scope;
import com.simplespringmvc.scope.ScopeAnnotation;
import com.simplespringmvc.scope.ScopedProxyMode;
import com.simplespringmvc.scope.SessionScope;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying that bean scopes work with:
 * - Dependency injection (ch24) — prototype and scoped beans as dependencies
 * - Component scanning (ch17) — @ScopeAnnotation detected during scanning
 * - AOP proxies (ch28) — scoped proxies used as injected dependencies
 * - Container refresh — prototype/scoped beans skipped during preInstantiateSingletons
 */
class ScopeIntegrationTest {

    // ─── Test helpers ────────────────────────────────────────────────

    public static class MockRequestAttributes implements RequestAttributes {
        private final Map<String, Object> requestAttrs = new ConcurrentHashMap<>();
        private final Map<String, Object> sessionAttrs;
        private final Object sessionMutex;

        public MockRequestAttributes() {
            this(new ConcurrentHashMap<>(), new Object());
        }

        public MockRequestAttributes(Map<String, Object> sessionStore, Object sessionMutex) {
            this.sessionAttrs = sessionStore;
            this.sessionMutex = sessionMutex;
        }

        @Override
        public Object getAttribute(String name, int scope) {
            return (scope == SCOPE_REQUEST) ? requestAttrs.get(name) : sessionAttrs.get(name);
        }

        @Override
        public void setAttribute(String name, Object value, int scope) {
            if (scope == SCOPE_REQUEST) {
                requestAttrs.put(name, value);
            } else {
                sessionAttrs.put(name, value);
            }
        }

        @Override
        public void removeAttribute(String name, int scope) {
            if (scope == SCOPE_REQUEST) {
                requestAttrs.remove(name);
            } else {
                sessionAttrs.remove(name);
            }
        }

        @Override
        public void registerDestructionCallback(String name, Runnable callback, int scope) {
        }

        @Override
        public String getSessionId() {
            return "mock-session-id";
        }

        @Override
        public Object getSessionMutex() {
            return sessionMutex;
        }
    }

    public static class MapBackedScope implements Scope {
        private final Map<String, Object> store = new ConcurrentHashMap<>();

        @Override
        public Object get(String name, Supplier<?> objectFactory) {
            return store.computeIfAbsent(name, k -> objectFactory.get());
        }

        @Override
        public Object remove(String name) {
            return store.remove(name);
        }

        @Override
        public void registerDestructionCallback(String name, Runnable callback) {
        }

        public void clear() {
            store.clear();
        }
    }

    @AfterEach
    void cleanup() {
        RequestContextHolder.resetRequestAttributes();
    }

    // ─── Prototype + DI integration ─────────────────────────────────

    public static class PrototypeCounter {
        private static final AtomicInteger instances = new AtomicInteger(0);
        private final int id;

        public PrototypeCounter() {
            this.id = instances.incrementAndGet();
        }

        public int getId() {
            return id;
        }

        public static void reset() {
            instances.set(0);
        }
    }

    @Nested
    class PrototypeWithDI {

        @BeforeEach
        void reset() {
            PrototypeCounter.reset();
        }

        @Test
        void shouldInjectDifferentPrototypeInstances_WhenResolvedMultipleTimes() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            BeanDefinition def = new BeanDefinition("protoCounter", PrototypeCounter.class);
            def.setScope("prototype");
            container.registerBeanDefinition(def);
            container.refresh();

            PrototypeCounter c1 = (PrototypeCounter) container.getBean("protoCounter");
            PrototypeCounter c2 = (PrototypeCounter) container.getBean("protoCounter");
            PrototypeCounter c3 = (PrototypeCounter) container.getBean("protoCounter");

            assertThat(c1.getId()).isEqualTo(1);
            assertThat(c2.getId()).isEqualTo(2);
            assertThat(c3.getId()).isEqualTo(3);
        }
    }

    // ─── Request scope + container integration ──────────────────────

    public static class RequestScopedBean {
        private static final AtomicInteger instances = new AtomicInteger(0);
        private final int id;

        public RequestScopedBean() {
            this.id = instances.incrementAndGet();
        }

        public int getId() {
            return id;
        }

        public static void reset() {
            instances.set(0);
        }
    }

    @Nested
    class RequestScopeIntegration {

        @BeforeEach
        void reset() {
            RequestScopedBean.reset();
        }

        @Test
        void shouldReturnSameInstance_WithinSingleRequest() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            container.registerScope("request", new RequestScope());

            BeanDefinition def = new BeanDefinition("reqBean", RequestScopedBean.class);
            def.setScope("request");
            container.registerBeanDefinition(def);
            container.refresh();

            MockRequestAttributes attrs = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs);
            try {
                RequestScopedBean b1 = (RequestScopedBean) container.getBean("reqBean");
                RequestScopedBean b2 = (RequestScopedBean) container.getBean("reqBean");
                assertThat(b1).isSameAs(b2);
            } finally {
                RequestContextHolder.resetRequestAttributes();
            }
        }

        @Test
        void shouldReturnNewInstance_ForDifferentRequests() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            container.registerScope("request", new RequestScope());

            BeanDefinition def = new BeanDefinition("reqBean", RequestScopedBean.class);
            def.setScope("request");
            container.registerBeanDefinition(def);
            container.refresh();

            // Request 1
            MockRequestAttributes attrs1 = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs1);
            RequestScopedBean b1 = (RequestScopedBean) container.getBean("reqBean");
            RequestContextHolder.resetRequestAttributes();

            // Request 2
            MockRequestAttributes attrs2 = new MockRequestAttributes();
            RequestContextHolder.setRequestAttributes(attrs2);
            RequestScopedBean b2 = (RequestScopedBean) container.getBean("reqBean");
            RequestContextHolder.resetRequestAttributes();

            assertThat(b1).isNotSameAs(b2);
            assertThat(b1.getId()).isNotEqualTo(b2.getId());
        }
    }

    // ─── Session scope + container integration ──────────────────────

    @Nested
    class SessionScopeIntegration {

        @BeforeEach
        void reset() {
            RequestScopedBean.reset();
        }

        @Test
        void shouldShareInstance_AcrossRequestsInSameSession() {
            SimpleBeanContainer container = new SimpleBeanContainer();
            container.registerScope("session", new SessionScope());

            BeanDefinition def = new BeanDefinition("sessBean", RequestScopedBean.class);
            def.setScope("session");
            container.registerBeanDefinition(def);
            container.refresh();

            // Shared session store
            Map<String, Object> sessionStore = new ConcurrentHashMap<>();
            Object mutex = new Object();

            // Request 1 in session
            MockRequestAttributes attrs1 = new MockRequestAttributes(sessionStore, mutex);
            RequestContextHolder.setRequestAttributes(attrs1);
            RequestScopedBean b1 = (RequestScopedBean) container.getBean("sessBean");
            RequestContextHolder.resetRequestAttributes();

            // Request 2 in same session
            MockRequestAttributes attrs2 = new MockRequestAttributes(sessionStore, mutex);
            RequestContextHolder.setRequestAttributes(attrs2);
            RequestScopedBean b2 = (RequestScopedBean) container.getBean("sessBean");
            RequestContextHolder.resetRequestAttributes();

            assertThat(b1).isSameAs(b2);
        }
    }

    // ─── Scoped proxy + DI integration ──────────────────────────────

    public static class SingletonService {
        private final PrototypeCounter counter;

        public SingletonService(PrototypeCounter counter) {
            this.counter = counter;
        }

        public int getCounterId() {
            return counter.getId();
        }
    }

    @Nested
    class ScopedProxyWithDI {

        @BeforeEach
        void reset() {
            PrototypeCounter.reset();
        }

        @Test
        void shouldResolveViaProxy_WhenScopedBeanInjectedIntoSingleton() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            MapBackedScope scope = new MapBackedScope();
            container.registerScope("custom", scope);

            // Register scoped bean with proxy
            BeanDefinition scopedDef = new BeanDefinition("protoCounter", PrototypeCounter.class);
            scopedDef.setScope("custom");
            scopedDef.setProxyMode(ScopedProxyMode.TARGET_CLASS);
            container.registerBeanDefinition(scopedDef);

            // Register singleton that depends on the scoped bean
            container.registerBeanDefinition("singletonService", SingletonService.class);

            container.refresh();

            SingletonService service = (SingletonService) container.getBean("singletonService");
            int id1 = service.getCounterId();

            // Clear scope — next call should get a new instance
            scope.clear();
            int id2 = service.getCounterId();

            assertThat(id1).isNotEqualTo(id2);
        }
    }

    // ─── Component scanning + @ScopeAnnotation integration ──────────

    @Component
    @ScopeAnnotation("prototype")
    public static class ScannedPrototype {
        private static final AtomicInteger instances = new AtomicInteger(0);
        private final int id;

        public ScannedPrototype() {
            this.id = instances.incrementAndGet();
        }

        public int getId() {
            return id;
        }

        public static void reset() {
            instances.set(0);
        }
    }

    @Nested
    class ScanningWithScope {

        @BeforeEach
        void reset() {
            ScannedPrototype.reset();
        }

        @Test
        void shouldDetectScopeAnnotation_WhenClasspathScanned() {
            SimpleBeanContainer container = new SimpleBeanContainer();

            // Manually register to simulate scanning behavior
            BeanDefinition def = new BeanDefinition("scannedPrototype", ScannedPrototype.class);
            ScopeAnnotation scopeAnn = ScannedPrototype.class.getAnnotation(ScopeAnnotation.class);
            if (scopeAnn != null) {
                def.setScope(scopeAnn.value());
                def.setProxyMode(scopeAnn.proxyMode());
            }
            container.registerBeanDefinition(def);
            container.refresh();

            // Should produce new instances each time (prototype)
            ScannedPrototype s1 = (ScannedPrototype) container.getBean("scannedPrototype");
            ScannedPrototype s2 = (ScannedPrototype) container.getBean("scannedPrototype");

            assertThat(s1).isNotSameAs(s2);
        }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Scope SPI** | Strategy interface (`get`/`remove`/`registerDestructionCallback`) that controls where bean instances are stored and how long they live |
| **Prototype scope** | New instance per `getBean()` call — no caching, no destruction callbacks, caller owns the lifecycle |
| **Request scope** | One instance per HTTP request, stored as a request attribute via `RequestContextHolder` |
| **Session scope** | One instance per HTTP session, with synchronized access to prevent concurrent modification |
| **Scoped proxy** | A singleton proxy (created via AOP from ch28) that delegates each method call to the scope's current instance, solving the lifecycle mismatch between long-lived and short-lived beans |
| **RequestContextHolder** | Thread-local holder that bridges the Servlet container's per-request lifecycle with Spring's scope abstraction |
| **autowireCandidate** | Flag on BeanDefinition that excludes scoped proxy targets from dependency injection — only the proxy gets injected |

**Next: Chapter 30 — API Versioning** — Route requests to different handler methods based on API version extracted from headers, URL paths, or query parameters, using Spring 7.0's new `ApiVersionStrategy` SPI.
