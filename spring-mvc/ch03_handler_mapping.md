# Chapter 3: Handler Mapping

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `SimpleDispatcherServlet` receives every HTTP request and writes a hardcoded `"Hello from DispatcherServlet"` response regardless of URL or HTTP method | No routing — every request gets the same response. There's no way to map `/users` to one method and `/orders` to another | Build a handler mapping that scans `@Controller` beans for `@RequestMapping` methods, registers them in a URL+method registry, and wires it into `doDispatch()` so requests are routed to the correct controller method |

---

## 3.1 The Integration Point

The integration point is `SimpleDispatcherServlet.doDispatch()` — the central dispatch method that currently writes a hardcoded response. This is where handler mapping plugs in to replace that placeholder with real request routing.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Replace the hardcoded "Hello from DispatcherServlet" response with handler lookup + reflection-based invocation

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    // Step 1: Look up the handler for this request (ch03)
    HandlerMethod handler = getHandler(request);

    if (handler == null) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND,
                "No handler found for " + request.getMethod() + " " + request.getRequestURI());
        return;
    }

    // Step 2: Invoke the handler (ch04 will extract this into HandlerAdapter)
    Object result = invokeHandler(handler);

    // Step 3: Write the result to the response
    response.setContentType("text/plain");
    response.setCharacterEncoding("UTF-8");
    if (result != null) {
        response.getWriter().write(result.toString());
    }
    response.getWriter().flush();
}
```

This code references `HandlerMethod`, `getHandler()`, and `invokeHandler()` — none of which exist yet. That's intentional. The integration point tells us what we need to build.

Two key decisions here:

1. **Handler lookup returns `null` for "not found" instead of throwing.** This follows the real `DispatcherServlet.getHandler()` pattern — the caller checks for null and sends a 404. This keeps the handler mapping decoupled from HTTP error handling.

2. **Reflection invocation lives inside `doDispatch()` temporarily.** In ch04, we'll extract this into a `HandlerAdapter` — but for now, keeping it here makes the routing feature self-contained and easy to understand.

This connects **Handler Mapping** (URL routing) to **DispatcherServlet** (request lifecycle). To make it work, we need to build:
- **`@Controller` annotation** — marks a class as a web controller
- **`@RequestMapping` annotation** — maps URL path + HTTP method to a handler method
- **`HandlerMethod`** — wraps a bean instance + `java.lang.reflect.Method`
- **`RouteKey`** — composite key of (path + HTTP method) for the registry
- **`SimpleHandlerMapping`** — scans beans, discovers mappings, looks up handlers

We also need to modify `initStrategies()` to initialize the handler mapping on startup.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Add `initHandlerMapping()` call in `initStrategies()`, add `handlerMapping` field

```java
private SimpleHandlerMapping handlerMapping;

protected void initStrategies() {
    initHandlerMapping();
    // Future features will initialize:
    // - HandlerAdapters (ch04)
    // - ExceptionResolvers (ch12)
    // - ViewResolvers (ch13)
}

private void initHandlerMapping() {
    handlerMapping = new SimpleHandlerMapping();
    handlerMapping.init(beanContainer);
}
```

## 3.2 The Annotations: @Controller and @RequestMapping

Before we can scan for handlers, we need annotations to mark them. These are the two fundamental annotations that connect Java classes to HTTP routing.

**New file:** `src/main/java/com/simplespringmvc/annotation/Controller.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Controller {
}
```

**New file:** `src/main/java/com/simplespringmvc/annotation/RequestMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestMapping {

    String path() default "";

    String method() default "";
}
```

Key design: `@RequestMapping` targets both `TYPE` and `METHOD`. When placed on a class, it defines a base path prefix. When on a method, it defines the specific endpoint. The real annotation has 10 attributes including `params`, `headers`, `consumes`, `produces`, and `version` — we start with just `path` and `method`.

## 3.3 HandlerMethod: Wrapping Bean + Method

`HandlerMethod` pairs a controller bean with a specific method on that bean. It's the "thing that handles a request" — the output of handler mapping and the input to handler invocation.

**New file:** `src/main/java/com/simplespringmvc/mapping/HandlerMethod.java`

```java
package com.simplespringmvc.mapping;

import java.lang.reflect.Method;

public class HandlerMethod {

    private final Object bean;
    private final Method method;

    public HandlerMethod(Object bean, Method method) {
        if (bean == null) {
            throw new IllegalArgumentException("Bean must not be null");
        }
        if (method == null) {
            throw new IllegalArgumentException("Method must not be null");
        }
        this.bean = bean;
        this.method = method;
    }

    public Object getBean() {
        return bean;
    }

    public Method getMethod() {
        return method;
    }

    public Class<?> getBeanType() {
        return bean.getClass();
    }

    @Override
    public String toString() {
        return bean.getClass().getSimpleName() + "#" + method.getName();
    }
}
```

The real `HandlerMethod` is 400+ lines: it supports lazy bean resolution by name, `MethodParameter[]` introspection, `@ResponseStatus` evaluation, and validation flags. Ours is just the two essential fields.

## 3.4 RouteKey: The Registry Key

We need a composite key for the mapping registry. The real Spring uses `RequestMappingInfo` — a composite of 8 request conditions (path patterns, HTTP methods, params, headers, consumes, produces, version, custom). We simplify to just path + HTTP method.

**New file:** `src/main/java/com/simplespringmvc/mapping/RouteKey.java`

```java
package com.simplespringmvc.mapping;

import java.util.Objects;

public final class RouteKey {

    private final String path;
    private final String httpMethod;

    public RouteKey(String path, String httpMethod) {
        this.path = normalizePath(path);
        this.httpMethod = httpMethod != null ? httpMethod.toUpperCase() : "";
    }

    public boolean matches(String requestPath, String requestMethod) {
        String normalizedRequestPath = normalizePath(requestPath);
        if (!this.path.equals(normalizedRequestPath)) {
            return false;
        }
        if (this.httpMethod.isEmpty()) {
            return true;
        }
        return this.httpMethod.equalsIgnoreCase(requestMethod);
    }

    private static String normalizePath(String path) {
        if (path == null || path.isEmpty()) return "/";
        if (!path.startsWith("/")) path = "/" + path;
        if (path.length() > 1 && path.endsWith("/")) path = path.substring(0, path.length() - 1);
        return path;
    }

    // equals, hashCode, toString omitted for brevity — see complete code section
}
```

Two important design choices:
- **Path normalization** — always start with `/`, never end with `/`. This prevents `/users` and `/users/` from being different routes.
- **Empty HTTP method means "match any"** — if `@RequestMapping(path = "/health")` omits the method, it responds to GET, POST, PUT, etc.

## 3.5 SimpleHandlerMapping: The Routing Engine

This is the core class that ties everything together. At startup, it scans beans for `@Controller`, discovers `@RequestMapping` methods, and builds the registry. At request time, it looks up the matching handler.

**New file:** `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java`

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;

import java.lang.reflect.Method;
import java.util.LinkedHashMap;
import java.util.Map;

public class SimpleHandlerMapping {

    private final Map<RouteKey, HandlerMethod> registry = new LinkedHashMap<>();

    public void init(BeanContainer beanContainer) {
        for (String beanName : beanContainer.getBeanNames()) {
            Object bean = beanContainer.getBean(beanName);
            if (isHandler(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    private boolean isHandler(Class<?> beanType) {
        return beanType.isAnnotationPresent(Controller.class);
    }

    private void detectHandlerMethods(Object bean) {
        Class<?> beanType = bean.getClass();

        String basePath = "";
        RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
        if (typeMapping != null) {
            basePath = typeMapping.path();
        }

        for (Method method : beanType.getDeclaredMethods()) {
            RequestMapping methodMapping = method.getAnnotation(RequestMapping.class);
            if (methodMapping == null) continue;

            String fullPath = combinePaths(basePath, methodMapping.path());
            String httpMethod = methodMapping.method();

            method.setAccessible(true);  // like Spring's ReflectionUtils.makeAccessible()

            RouteKey routeKey = new RouteKey(fullPath, httpMethod);
            HandlerMethod handlerMethod = new HandlerMethod(bean, method);

            if (registry.containsKey(routeKey)) {
                throw new IllegalStateException(
                        "Ambiguous mapping: " + routeKey + " is already mapped to "
                                + registry.get(routeKey) + ". Cannot map " + handlerMethod);
            }

            registry.put(routeKey, handlerMethod);
        }
    }

    public HandlerMethod lookupHandler(String requestPath, String requestMethod) {
        for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
            if (entry.getKey().matches(requestPath, requestMethod)) {
                return entry.getValue();
            }
        }
        return null;
    }
}
```

The flow mirrors the real framework's `AbstractHandlerMethodMapping`:

| Our Code | Real Framework |
|----------|----------------|
| `init(BeanContainer)` | `initHandlerMethods()` from `afterPropertiesSet()` |
| `isHandler()` | `RequestMappingHandlerMapping.isHandler()` |
| `detectHandlerMethods()` | `AbstractHandlerMethodMapping.detectHandlerMethods()` |
| `lookupHandler()` | `AbstractHandlerMethodMapping.lookupHandlerMethod()` |
| `RouteKey` | `RequestMappingInfo` (8 conditions) |
| `LinkedHashMap` | `MappingRegistry` with `ReentrantReadWriteLock` |

## 3.6 Try It Yourself

Delete the files in `src/main/java/com/simplespringmvc/mapping/` and `src/main/java/com/simplespringmvc/annotation/` and try to rebuild them using only the test files as a specification.

<details>
<summary>Challenge 1: Implement the annotation-scanning loop in detectHandlerMethods()</summary>

Given a bean with `@Controller` and several `@RequestMapping` methods, write the logic that:
1. Checks for a type-level `@RequestMapping` to get the base path
2. Iterates methods looking for `@RequestMapping`
3. Combines base path + method path
4. Registers each as a `RouteKey → HandlerMethod` pair

Hint: Use `beanType.getAnnotation(RequestMapping.class)` for the class level and `method.getAnnotation(RequestMapping.class)` for methods.

```java
private void detectHandlerMethods(Object bean) {
    Class<?> beanType = bean.getClass();

    String basePath = "";
    RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
    if (typeMapping != null) {
        basePath = typeMapping.path();
    }

    for (Method method : beanType.getDeclaredMethods()) {
        RequestMapping methodMapping = method.getAnnotation(RequestMapping.class);
        if (methodMapping == null) continue;

        String fullPath = combinePaths(basePath, methodMapping.path());
        method.setAccessible(true);

        RouteKey routeKey = new RouteKey(fullPath, methodMapping.method());
        HandlerMethod handlerMethod = new HandlerMethod(bean, method);

        if (registry.containsKey(routeKey)) {
            throw new IllegalStateException("Ambiguous mapping: " + routeKey);
        }
        registry.put(routeKey, handlerMethod);
    }
}
```

</details>

<details>
<summary>Challenge 2: Wire handler mapping into doDispatch()</summary>

Replace the hardcoded response in `doDispatch()` with:
1. Look up the handler using `handlerMapping.lookupHandler()`
2. Return 404 if not found
3. Invoke via `handler.getMethod().invoke(handler.getBean())`
4. Write the result as text/plain

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HandlerMethod handler = handlerMapping.lookupHandler(
            request.getRequestURI(), request.getMethod());

    if (handler == null) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND,
                "No handler found for " + request.getMethod() + " " + request.getRequestURI());
        return;
    }

    Object result = handler.getMethod().invoke(handler.getBean());

    response.setContentType("text/plain");
    response.setCharacterEncoding("UTF-8");
    if (result != null) {
        response.getWriter().write(result.toString());
    }
    response.getWriter().flush();
}
```

</details>

## 3.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/mapping/HandlerMethodTest.java` — Tests the wrapper class (5 tests):
- `shouldStoreBeanAndMethod` — constructor stores both references
- `shouldThrowOnNullBean` / `shouldThrowOnNullMethod` — validates inputs
- `shouldReturnBeansClass` — `getBeanType()` works
- `shouldReturnReadableString` — `toString()` format

**New file:** `src/test/java/com/simplespringmvc/mapping/RouteKeyTest.java` — Tests path normalization, matching, and equality (15 tests):
- Path normalization: leading slash, trailing slash, root, null, empty
- HTTP method normalization: uppercase, null handling
- Matching: exact match, case-insensitive, different path/method, wildcard
- Equality: same key, different path, different method

**New file:** `src/test/java/com/simplespringmvc/mapping/SimpleHandlerMappingTest.java` — Tests the routing engine (10 tests):
- Handler detection: finds `@Controller`, ignores plain beans
- Path combination: type-level + method-level, empty-method wildcard
- Lookup: exact match, multiple methods on same path, not found, wrong HTTP method
- Duplicate detection: throws on ambiguous mapping
- Multiple controllers: registers from several controllers

**Modified file:** `src/test/java/com/simplespringmvc/servlet/SimpleDispatcherServletTest.java` — Updated for new dispatch behavior:
- Tests now register a `@Controller` bean and verify routing
- New test: `shouldSend404_WhenNoHandlerFound`
- New test: `shouldInitializeHandlerMapping` verifying `init()` creates the mapping

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/HandlerMappingIntegrationTest.java` — Full HTTP stack tests (7 tests):

```java
@Test
@DisplayName("should route GET /hello to GreetingController#hello")
void shouldRouteGetHello() throws Exception {
    HttpResponse<String> response = httpClient.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("Hello, World!");
}

@Test
@DisplayName("should return 404 for unmatched path")
void shouldReturn404_WhenPathNotFound() throws Exception {
    HttpResponse<String> response = httpClient.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/nonexistent")).GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(404);
}
```

**Modified file:** `src/test/java/com/simplespringmvc/integration/EmbeddedTomcatIntegrationTest.java` — Updated to register controllers (previously expected hardcoded response for all paths).

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 3.8 Why This Works

> ★ **Insight: The Template Method pattern powers extensible scanning** -------------------------------------------
> The real `AbstractHandlerMethodMapping` defines the **algorithm skeleton** — iterate beans, check `isHandler()`, scan methods with `getMappingForMethod()`, register results — while leaving the specifics to subclasses. `RequestMappingHandlerMapping` fills in the template with `@Controller` detection and `@RequestMapping` parsing. Our `SimpleHandlerMapping` collapses both into one class, but the method names (`isHandler`, `detectHandlerMethods`) mirror the real structure. When you add `@GetMapping` support in ch10, you'll modify `detectHandlerMethods()` without touching the scan loop — just like the real framework.
>
> **Trade-off:** We sacrificed the Template Method abstraction for simplicity. A real application might have multiple mapping strategies (URL-based, bean-name-based, router-function-based) — the abstract base class lets them coexist.
> -----------------------------------------------------------

> ★ **Insight: Startup vs. request-time work — front-load the cost** -------------------------------------------
> Handler mapping does its expensive work (reflection, annotation scanning) **once at startup** and stores the results in a fast-lookup registry. At request time, `lookupHandler()` just iterates the registry — no reflection, no annotation scanning. The real framework takes this further: `MappingRegistry` maintains a `pathLookup` MultiValueMap indexed by direct paths for O(1) lookup, falling back to full scan only when patterns are involved.
>
> **When this pattern breaks down:** If you need to add/remove handlers at runtime (hot-reload, dynamic routing), the startup-only scan becomes a limitation. The real `MappingRegistry` uses a `ReentrantReadWriteLock` to support concurrent reads with occasional writes — that's the escape hatch.
> -----------------------------------------------------------

> ★ **Insight: Why `RouteKey` will evolve into something much richer** -------------------------------------------
> Our `RouteKey` captures just two dimensions: path and HTTP method. The real `RequestMappingInfo` is a **Composite pattern** — 8 `RequestCondition` objects, each with `combine()`, `getMatchingCondition()`, and `compareTo()` methods. This design means adding a new condition (like API versioning in Spring 7.0) doesn't change any existing code — you just add a new condition to the composite. We'll evolve toward this in ch07 (path patterns), ch10 (composed annotations), and ch14 (content negotiation).
> -----------------------------------------------------------

## 3.9 What We Enhanced

| Aspect | Before (ch02) | Current (ch03) | Real Framework |
|--------|---------------|----------------|----------------|
| **Request routing** | Hardcoded response for every URL | URL + HTTP method routing to specific controller methods | `RequestMappingInfo` with 8 conditions, pattern matching, content negotiation |
| **doDispatch()** | Writes "Hello from DispatcherServlet" | Looks up handler → invokes → writes result | 7-step pipeline with interceptors, adapters, exception handling, view rendering |
| **Handler discovery** | None | Scans `@Controller` beans for `@RequestMapping` at startup | Template Method pattern with `AbstractHandlerMethodMapping`, `MergedAnnotations`, `MethodIntrospector` |
| **Handler invocation** | None | Direct `Method.invoke()` in `doDispatch()` | Separated into `HandlerAdapter` → `InvocableHandlerMethod` with argument resolvers |
| **Registry** | None | `LinkedHashMap<RouteKey, HandlerMethod>` | `MappingRegistry` with `ReentrantReadWriteLock`, direct-path index, name index, CORS index |
| **initStrategies()** | Empty | Initializes `SimpleHandlerMapping` | Initializes `HandlerMapping`s, `HandlerAdapter`s, `ViewResolver`s, `ExceptionResolver`s, etc. |

## 3.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@Controller` | `@Controller` | `Controller.java:37` | Real version is meta-annotated with `@Component` for classpath scanning |
| `@RequestMapping` | `@RequestMapping` | `RequestMapping.java:82` | Real version has `method()` as `RequestMethod[]`, plus `params`, `headers`, `consumes`, `produces`, `version` |
| `HandlerMethod` | `HandlerMethod` | `HandlerMethod.java:55` | Real version supports lazy bean resolution, `MethodParameter[]`, `@ResponseStatus`, validation flags |
| `RouteKey` | `RequestMappingInfo` | `RequestMappingInfo.java:66` | Real version is a composite of 8 `RequestCondition` objects with `combine()`, `getMatchingCondition()`, `compareTo()` |
| `SimpleHandlerMapping.init()` | `AbstractHandlerMethodMapping.initHandlerMethods()` | `AbstractHandlerMethodMapping.java:217` | Real version uses `InitializingBean.afterPropertiesSet()`, handles scoped proxies, uses `MethodIntrospector` |
| `SimpleHandlerMapping.isHandler()` | `RequestMappingHandlerMapping.isHandler()` | `RequestMappingHandlerMapping.java:181` | Real version uses `AnnotatedElementUtils.hasAnnotation()` for meta-annotation support |
| `SimpleHandlerMapping.detectHandlerMethods()` | `AbstractHandlerMethodMapping.detectHandlerMethods()` | `AbstractHandlerMethodMapping.java:270` | Real version uses `MethodIntrospector.selectMethods()` and `ClassUtils.getUserClass()` for CGLIB proxy unwrapping |
| `SimpleHandlerMapping.lookupHandler()` | `AbstractHandlerMethodMapping.lookupHandlerMethod()` | `AbstractHandlerMethodMapping.java:393` | Real version has direct-path fast lookup, match sorting, ambiguity detection, CORS preflight handling |
| `doDispatch() getHandler()` | `DispatcherServlet.getHandler()` | `DispatcherServlet.java:1103` | Real version iterates a list of `HandlerMapping` beans, returns `HandlerExecutionChain` with interceptors |
| `doDispatch() invokeHandler()` | `HandlerAdapter.handle()` | `RequestMappingHandlerAdapter.java:798` | Real version uses `InvocableHandlerMethod` with argument resolvers, return value handlers |

## 3.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/Controller.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as a web controller whose methods can handle HTTP requests.
 *
 * Maps to: {@code org.springframework.stereotype.Controller}
 *
 * In real Spring, @Controller is meta-annotated with @Component so it gets
 * picked up by component scanning. Our simplified version is just a marker
 * that SimpleHandlerMapping looks for when scanning beans.
 *
 * Simplifications:
 * <ul>
 *   <li>No @Component meta-annotation (no classpath scanning yet — ch17)</li>
 *   <li>No value() attribute for bean naming</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Controller {
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/RequestMapping.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Maps an HTTP request to a handler method based on URL path and HTTP method.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.RequestMapping}
 *
 * Can be placed on both type (class) and method level:
 * <ul>
 *   <li>Type-level: defines a base path prefix for all methods in the controller</li>
 *   <li>Method-level: defines the specific path and HTTP method for this handler</li>
 * </ul>
 *
 * Example:
 * <pre>
 * {@literal @}Controller
 * {@literal @}RequestMapping("/api")
 * public class UserController {
 *
 *     {@literal @}RequestMapping(path = "/users", method = "GET")
 *     public String listUsers() { ... }
 * }
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No params, headers, consumes, produces attributes (ch14)</li>
 *   <li>No pattern matching with {variables} (ch07)</li>
 *   <li>method is a single String, not RequestMethod[] (enough for exact match)</li>
 *   <li>path is a single String, not String[] (one path per annotation)</li>
 * </ul>
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestMapping {

    /**
     * The URL path this mapping applies to.
     * Example: "/users" or "/api/orders"
     */
    String path() default "";

    /**
     * The HTTP method to match. Case-insensitive.
     * Example: "GET", "POST", "PUT", "DELETE"
     * Empty string means match any HTTP method.
     */
    String method() default "";
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/HandlerMethod.java` [NEW]

```java
package com.simplespringmvc.mapping;

import java.lang.reflect.Method;

/**
 * Wraps a controller bean instance and a specific method on that bean,
 * representing a single endpoint that can handle HTTP requests.
 *
 * Maps to: {@code org.springframework.web.method.HandlerMethod}
 *
 * The real HandlerMethod is much richer: it supports lazy bean resolution
 * (by bean name through BeanFactory), method parameter introspection via
 * MethodParameter[], return type analysis, @ResponseStatus evaluation,
 * and validation flags. It extends AnnotatedMethod which itself wraps
 * a SynthesizingMethodParameter.
 *
 * Simplifications:
 * <ul>
 *   <li>Always holds the bean instance directly (no lazy resolution by name)</li>
 *   <li>No MethodParameter[] — we'll add that in ch05</li>
 *   <li>No @ResponseStatus evaluation</li>
 *   <li>No validation flag support</li>
 * </ul>
 */
public class HandlerMethod {

    private final Object bean;
    private final Method method;

    public HandlerMethod(Object bean, Method method) {
        if (bean == null) {
            throw new IllegalArgumentException("Bean must not be null");
        }
        if (method == null) {
            throw new IllegalArgumentException("Method must not be null");
        }
        this.bean = bean;
        this.method = method;
    }

    /**
     * The controller bean instance that owns this method.
     */
    public Object getBean() {
        return bean;
    }

    /**
     * The method to invoke on the bean.
     */
    public Method getMethod() {
        return method;
    }

    /**
     * The type of the controller bean (convenience for annotation checks).
     */
    public Class<?> getBeanType() {
        return bean.getClass();
    }

    @Override
    public String toString() {
        return bean.getClass().getSimpleName() + "#" + method.getName();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/RouteKey.java` [NEW]

```java
package com.simplespringmvc.mapping;

import java.util.Objects;

/**
 * A composite key combining a URL path and HTTP method for handler lookup.
 *
 * This has no direct equivalent in real Spring — the real framework uses
 * {@code RequestMappingInfo} (a composite of 8 request conditions) as the
 * registry key. We simplify to just path + method, which is enough for
 * exact-match routing.
 *
 * The path is normalized to always start with "/" and never end with "/".
 * The HTTP method is stored in uppercase for case-insensitive comparison.
 *
 * RouteKey is immutable and suitable for use as a HashMap key.
 */
public final class RouteKey {

    private final String path;
    private final String httpMethod;

    public RouteKey(String path, String httpMethod) {
        this.path = normalizePath(path);
        this.httpMethod = httpMethod != null ? httpMethod.toUpperCase() : "";
    }

    public String getPath() {
        return path;
    }

    public String getHttpMethod() {
        return httpMethod;
    }

    /**
     * Checks if this RouteKey matches a request path and HTTP method.
     * A RouteKey with an empty httpMethod matches any HTTP method.
     */
    public boolean matches(String requestPath, String requestMethod) {
        String normalizedRequestPath = normalizePath(requestPath);
        if (!this.path.equals(normalizedRequestPath)) {
            return false;
        }
        // Empty httpMethod means "match any"
        if (this.httpMethod.isEmpty()) {
            return true;
        }
        return this.httpMethod.equalsIgnoreCase(requestMethod);
    }

    private static String normalizePath(String path) {
        if (path == null || path.isEmpty()) {
            return "/";
        }
        // Ensure starts with /
        if (!path.startsWith("/")) {
            path = "/" + path;
        }
        // Remove trailing slash (except for root "/")
        if (path.length() > 1 && path.endsWith("/")) {
            path = path.substring(0, path.length() - 1);
        }
        return path;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof RouteKey that)) return false;
        return path.equals(that.path) && httpMethod.equals(that.httpMethod);
    }

    @Override
    public int hashCode() {
        return Objects.hash(path, httpMethod);
    }

    @Override
    public String toString() {
        if (httpMethod.isEmpty()) {
            return "ALL " + path;
        }
        return httpMethod + " " + path;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java` [NEW]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;

import java.lang.reflect.Method;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Scans all beans in the container for @Controller classes, discovers methods
 * annotated with @RequestMapping, and builds a registry that maps
 * (URL path + HTTP method) → HandlerMethod.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping}
 * which extends {@code AbstractHandlerMethodMapping<RequestMappingInfo>}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. afterPropertiesSet() → initHandlerMethods()
 *   2. initHandlerMethods() iterates all beans, calls isHandler() on each
 *   3. isHandler() checks for @Controller annotation (via MergedAnnotations)
 *   4. detectHandlerMethods() introspects each handler class for @RequestMapping methods
 *   5. getMappingForMethod() builds a RequestMappingInfo from the annotation
 *   6. registerHandlerMethod() stores (RequestMappingInfo → HandlerMethod) in MappingRegistry
 *   7. At request time, lookupHandlerMethod() matches the request against all registrations
 * </pre>
 *
 * We collapse steps 1-7 into a simpler flow:
 * <pre>
 *   1. init(BeanContainer) iterates all beans
 *   2. Checks each bean class for @Controller
 *   3. For each @Controller, scans methods for @RequestMapping
 *   4. Combines type-level + method-level paths
 *   5. Stores (RouteKey → HandlerMethod) in a LinkedHashMap
 *   6. lookupHandler() matches request path + method against RouteKeys
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No MappingRegistry inner class with ReadWriteLock</li>
 *   <li>No RequestMappingInfo composite — just RouteKey (path + method)</li>
 *   <li>No direct-path optimization (we scan all entries on every request)</li>
 *   <li>No HandlerExecutionChain wrapping (added in ch11)</li>
 *   <li>No meta-annotation detection (added in ch10)</li>
 *   <li>No pattern matching with {variables} (added in ch07)</li>
 * </ul>
 */
public class SimpleHandlerMapping {

    private final Map<RouteKey, HandlerMethod> registry = new LinkedHashMap<>();

    /**
     * Scan all beans in the container and register handler methods.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.initHandlerMethods()} (line 217)
     * which is called from afterPropertiesSet().
     */
    public void init(BeanContainer beanContainer) {
        for (String beanName : beanContainer.getBeanNames()) {
            Object bean = beanContainer.getBean(beanName);
            if (isHandler(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    /**
     * Check if a bean type is a handler (i.e., annotated with @Controller).
     *
     * Maps to: {@code RequestMappingHandlerMapping.isHandler()} (line 181)
     * Real version uses: AnnotatedElementUtils.hasAnnotation(beanType, Controller.class)
     * which walks the meta-annotation hierarchy. We do a direct check for now.
     */
    private boolean isHandler(Class<?> beanType) {
        return beanType.isAnnotationPresent(Controller.class);
    }

    /**
     * Discover all @RequestMapping methods on a handler bean and register them.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.detectHandlerMethods()} (line 270)
     * Real version uses MethodIntrospector.selectMethods() to scan the class,
     * then calls getMappingForMethod() for each method.
     */
    private void detectHandlerMethods(Object bean) {
        Class<?> beanType = bean.getClass();

        // Check for type-level @RequestMapping (base path prefix)
        String basePath = "";
        RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
        if (typeMapping != null) {
            basePath = typeMapping.path();
        }

        // Scan all declared methods for @RequestMapping
        for (Method method : beanType.getDeclaredMethods()) {
            RequestMapping methodMapping = method.getAnnotation(RequestMapping.class);
            if (methodMapping == null) {
                continue;
            }

            // Combine type-level base path + method-level path
            String fullPath = combinePaths(basePath, methodMapping.path());
            String httpMethod = methodMapping.method();

            // Ensure the method is accessible via reflection.
            // Maps to: ReflectionUtils.makeAccessible() used in Spring's
            // InvocableHandlerMethod. Needed when the declaring class
            // is package-private (common in tests, inner classes, etc.)
            method.setAccessible(true);

            RouteKey routeKey = new RouteKey(fullPath, httpMethod);
            HandlerMethod handlerMethod = new HandlerMethod(bean, method);

            // Check for duplicate mappings
            if (registry.containsKey(routeKey)) {
                HandlerMethod existing = registry.get(routeKey);
                throw new IllegalStateException(
                        "Ambiguous mapping: " + routeKey + " is already mapped to " + existing
                                + ". Cannot map " + handlerMethod);
            }

            registry.put(routeKey, handlerMethod);
        }
    }

    /**
     * Look up the handler method for a given request.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.lookupHandlerMethod()} (line 393)
     *
     * The real version first tries a fast direct-path lookup, then falls back
     * to scanning all mappings. We always scan (simple but O(n)).
     *
     * @return the matching HandlerMethod, or null if no handler found
     */
    public HandlerMethod lookupHandler(String requestPath, String requestMethod) {
        // First pass: try exact match (path + method)
        for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
            RouteKey key = entry.getKey();
            if (key.matches(requestPath, requestMethod)) {
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Returns an unmodifiable view of all registered mappings.
     * Useful for debugging and testing.
     */
    public Map<RouteKey, HandlerMethod> getRegisteredMappings() {
        return Map.copyOf(registry);
    }

    /**
     * Combine a base path and method path.
     * Handles cases like "/" + "/users" → "/users", "/api" + "/users" → "/api/users".
     */
    private String combinePaths(String basePath, String methodPath) {
        if (basePath == null || basePath.isEmpty()) {
            return methodPath;
        }
        if (methodPath == null || methodPath.isEmpty()) {
            return basePath;
        }
        // Remove trailing slash from base, ensure method starts with /
        String base = basePath.endsWith("/") ? basePath.substring(0, basePath.length() - 1) : basePath;
        String method = methodPath.startsWith("/") ? methodPath : "/" + methodPath;
        return base + method;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.lang.reflect.InvocationTargetException;

/**
 * The front controller that receives ALL HTTP requests and dispatches them
 * to the appropriate handler.
 *
 * Maps to: {@code org.springframework.web.servlet.DispatcherServlet}
 *
 * The real DispatcherServlet sits at the bottom of a three-class hierarchy:
 * <pre>
 *   HttpServletBean          → maps init-params to bean properties
 *     └── FrameworkServlet   → manages WebApplicationContext, request lifecycle
 *           └── DispatcherServlet → the actual dispatch logic
 * </pre>
 *
 * We collapse all three layers into one class. The key method is {@link #doDispatch}
 * which is the central routing point — every future feature adds a step here:
 * <ul>
 *   <li>ch03: handler mapping lookup</li>
 *   <li>ch04: handler adapter invocation</li>
 *   <li>ch11: interceptor pre/post processing</li>
 *   <li>ch12: exception handler resolution</li>
 *   <li>ch13: view rendering</li>
 * </ul>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No locale/theme resolution (added in ch20)</li>
 *   <li>No flash map management (added in ch18)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Called by Tomcat once the Servlet is registered. Initializes strategy
     * components (handler mappings, adapters, etc.) from the bean container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * which is called from {@code onRefresh()}, which is triggered by
     * {@code FrameworkServlet.initWebApplicationContext()}.
     *
     * Currently empty — future features will add:
     * <ul>
     *   <li>ch03: initHandlerMappings()</li>
     *   <li>ch04: initHandlerAdapters()</li>
     *   <li>ch11: interceptor registration</li>
     *   <li>ch12: initExceptionResolvers()</li>
     *   <li>ch13: initViewResolvers()</li>
     * </ul>
     */
    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    /**
     * Initialize all strategy beans from the container.
     * Subfeatures will populate this method.
     */
    protected void initStrategies() {
        initHandlerMapping();
        // Future features will initialize:
        // - HandlerAdapters (ch04)
        // - ExceptionResolvers (ch12)
        // - ViewResolvers (ch13)
    }

    /**
     * Create the handler mapping and scan all beans for @Controller methods.
     *
     * Maps to: {@code DispatcherServlet.initHandlerMappings()} (line 505)
     * Real version looks for HandlerMapping beans in the context. We create
     * one directly and initialize it with our bean container.
     */
    private void initHandlerMapping() {
        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(beanContainer);
    }

    /**
     * Override service() to route ALL HTTP methods through doDispatch().
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870) which overrides
     * HttpServlet.service() and routes GET/POST/PUT/DELETE/PATCH/OPTIONS/TRACE
     * all through {@code processRequest()}, which calls {@code doService()},
     * which calls {@code doDispatch()}.
     *
     * We collapse that chain: service() → doDispatch() directly.
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        }
    }

    /**
     * The central dispatch method — the heart of the framework.
     *
     * Maps to: {@code DispatcherServlet.doDispatch()} (line 935)
     *
     * The real doDispatch() flow:
     * <pre>
     *   1. checkMultipart(request)
     *   2. mappedHandler = getHandler(request)         ← ch03
     *   3. mappedHandler.applyPreHandle()              ← ch11
     *   4. ha = getHandlerAdapter(handler)             ← ch04
     *   5. mv = ha.handle(request, response, handler)  ← ch04
     *   6. mappedHandler.applyPostHandle()             ← ch11
     *   7. processDispatchResult(mv, exception)        ← ch12, ch13
     * </pre>
     *
     * For now, we just write a simple response to prove the pipeline works.
     * Each future feature will replace a piece of this placeholder.
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        // Step 1: Look up the handler for this request (ch03)
        HandlerMethod handler = getHandler(request);

        if (handler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    "No handler found for " + request.getMethod() + " " + request.getRequestURI());
            return;
        }

        // Step 2: Invoke the handler (ch04 will extract this into HandlerAdapter)
        Object result = invokeHandler(handler);

        // Step 3: Write the result to the response
        // For now, we write String results directly as text/plain.
        // ch04 will move this into HandlerAdapter, ch06 will add @ResponseBody + JSON.
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        if (result != null) {
            response.getWriter().write(result.toString());
        }
        response.getWriter().flush();
    }

    /**
     * Find the HandlerMethod that matches this request.
     *
     * Maps to: {@code DispatcherServlet.getHandler()} (line 1103)
     * Real version iterates a list of HandlerMapping beans and returns the
     * first non-null HandlerExecutionChain. We have just one mapping.
     */
    private HandlerMethod getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        return handlerMapping.lookupHandler(request.getRequestURI(), request.getMethod());
    }

    /**
     * Invoke the handler method via reflection and return the result.
     *
     * Maps to: {@code HandlerAdapter.handle()} → {@code InvocableHandlerMethod.invokeForRequest()}
     * This is a temporary implementation — ch04 will extract it into a proper HandlerAdapter.
     * For now, only supports no-arg methods returning String.
     */
    private Object invokeHandler(HandlerMethod handler) throws Exception {
        try {
            return handler.getMethod().invoke(handler.getBean());
        } catch (InvocationTargetException ex) {
            // Unwrap the real exception from the reflection wrapper
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    /**
     * Provides access to the bean container for strategy initialization.
     * Future features use this to look up HandlerMappings, HandlerAdapters, etc.
     */
    public BeanContainer getBeanContainer() {
        return beanContainer;
    }

    /**
     * Provides access to the handler mapping for testing and inspection.
     */
    public SimpleHandlerMapping getHandlerMapping() {
        return handlerMapping;
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/mapping/HandlerMethodTest.java` [NEW]

```java
package com.simplespringmvc.mapping;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link HandlerMethod}.
 */
class HandlerMethodTest {

    static class SampleController {
        public String hello() {
            return "hello";
        }
    }

    // ─── Construction ────────────────────────────────────────────────

    @Nested
    @DisplayName("Construction")
    class Construction {

        @Test
        @DisplayName("should store bean and method")
        void shouldStoreBeanAndMethod() throws Exception {
            SampleController bean = new SampleController();
            Method method = SampleController.class.getMethod("hello");

            HandlerMethod hm = new HandlerMethod(bean, method);

            assertThat(hm.getBean()).isSameAs(bean);
            assertThat(hm.getMethod()).isSameAs(method);
        }

        @Test
        @DisplayName("should throw on null bean")
        void shouldThrowOnNullBean() throws Exception {
            Method method = SampleController.class.getMethod("hello");

            assertThatThrownBy(() -> new HandlerMethod(null, method))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Bean");
        }

        @Test
        @DisplayName("should throw on null method")
        void shouldThrowOnNullMethod() {
            assertThatThrownBy(() -> new HandlerMethod(new SampleController(), null))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Method");
        }
    }

    // ─── getBeanType ─────────────────────────────────────────────────

    @Nested
    @DisplayName("getBeanType")
    class GetBeanType {

        @Test
        @DisplayName("should return the bean's class")
        void shouldReturnBeansClass() throws Exception {
            SampleController bean = new SampleController();
            Method method = SampleController.class.getMethod("hello");

            HandlerMethod hm = new HandlerMethod(bean, method);

            assertThat(hm.getBeanType()).isEqualTo(SampleController.class);
        }
    }

    // ─── toString ────────────────────────────────────────────────────

    @Nested
    @DisplayName("toString")
    class ToString {

        @Test
        @DisplayName("should return ClassName#methodName")
        void shouldReturnReadableString() throws Exception {
            SampleController bean = new SampleController();
            Method method = SampleController.class.getMethod("hello");

            HandlerMethod hm = new HandlerMethod(bean, method);

            assertThat(hm.toString()).isEqualTo("SampleController#hello");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/mapping/RouteKeyTest.java` [NEW]

```java
package com.simplespringmvc.mapping;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link RouteKey}.
 */
class RouteKeyTest {

    // ─── Path normalization ──────────────────────────────────────────

    @Nested
    @DisplayName("Path normalization")
    class PathNormalization {

        @Test
        @DisplayName("should add leading slash if missing")
        void shouldAddLeadingSlash() {
            RouteKey key = new RouteKey("users", "GET");
            assertThat(key.getPath()).isEqualTo("/users");
        }

        @Test
        @DisplayName("should remove trailing slash")
        void shouldRemoveTrailingSlash() {
            RouteKey key = new RouteKey("/users/", "GET");
            assertThat(key.getPath()).isEqualTo("/users");
        }

        @Test
        @DisplayName("should preserve root path")
        void shouldPreserveRootPath() {
            RouteKey key = new RouteKey("/", "GET");
            assertThat(key.getPath()).isEqualTo("/");
        }

        @Test
        @DisplayName("should handle null path as root")
        void shouldHandleNullPathAsRoot() {
            RouteKey key = new RouteKey(null, "GET");
            assertThat(key.getPath()).isEqualTo("/");
        }

        @Test
        @DisplayName("should handle empty path as root")
        void shouldHandleEmptyPathAsRoot() {
            RouteKey key = new RouteKey("", "GET");
            assertThat(key.getPath()).isEqualTo("/");
        }
    }

    // ─── HTTP method normalization ───────────────────────────────────

    @Nested
    @DisplayName("HTTP method normalization")
    class HttpMethodNormalization {

        @Test
        @DisplayName("should uppercase HTTP method")
        void shouldUppercaseHttpMethod() {
            RouteKey key = new RouteKey("/api", "get");
            assertThat(key.getHttpMethod()).isEqualTo("GET");
        }

        @Test
        @DisplayName("should handle null method as empty string")
        void shouldHandleNullMethodAsEmpty() {
            RouteKey key = new RouteKey("/api", null);
            assertThat(key.getHttpMethod()).isEmpty();
        }
    }

    // ─── Matching ────────────────────────────────────────────────────

    @Nested
    @DisplayName("Matching")
    class Matching {

        @Test
        @DisplayName("should match exact path and method")
        void shouldMatchExactPathAndMethod() {
            RouteKey key = new RouteKey("/users", "GET");
            assertThat(key.matches("/users", "GET")).isTrue();
        }

        @Test
        @DisplayName("should match case-insensitively on HTTP method")
        void shouldMatchCaseInsensitiveMethod() {
            RouteKey key = new RouteKey("/users", "GET");
            assertThat(key.matches("/users", "get")).isTrue();
        }

        @Test
        @DisplayName("should not match different path")
        void shouldNotMatchDifferentPath() {
            RouteKey key = new RouteKey("/users", "GET");
            assertThat(key.matches("/orders", "GET")).isFalse();
        }

        @Test
        @DisplayName("should not match different HTTP method")
        void shouldNotMatchDifferentMethod() {
            RouteKey key = new RouteKey("/users", "GET");
            assertThat(key.matches("/users", "POST")).isFalse();
        }

        @Test
        @DisplayName("should match any HTTP method when method is empty")
        void shouldMatchAnyMethod_WhenMethodEmpty() {
            RouteKey key = new RouteKey("/users", "");
            assertThat(key.matches("/users", "GET")).isTrue();
            assertThat(key.matches("/users", "POST")).isTrue();
            assertThat(key.matches("/users", "DELETE")).isTrue();
        }

        @Test
        @DisplayName("should normalize request path before matching")
        void shouldNormalizeRequestPath() {
            RouteKey key = new RouteKey("/users", "GET");
            assertThat(key.matches("/users/", "GET")).isTrue();
        }
    }

    // ─── equals and hashCode ─────────────────────────────────────────

    @Nested
    @DisplayName("equals and hashCode")
    class EqualsAndHashCode {

        @Test
        @DisplayName("should be equal for same path and method")
        void shouldBeEqual() {
            RouteKey key1 = new RouteKey("/users", "GET");
            RouteKey key2 = new RouteKey("/users", "GET");
            assertThat(key1).isEqualTo(key2);
            assertThat(key1.hashCode()).isEqualTo(key2.hashCode());
        }

        @Test
        @DisplayName("should not be equal for different paths")
        void shouldNotBeEqualForDifferentPaths() {
            RouteKey key1 = new RouteKey("/users", "GET");
            RouteKey key2 = new RouteKey("/orders", "GET");
            assertThat(key1).isNotEqualTo(key2);
        }

        @Test
        @DisplayName("should not be equal for different methods")
        void shouldNotBeEqualForDifferentMethods() {
            RouteKey key1 = new RouteKey("/users", "GET");
            RouteKey key2 = new RouteKey("/users", "POST");
            assertThat(key1).isNotEqualTo(key2);
        }
    }

    // ─── toString ────────────────────────────────────────────────────

    @Nested
    @DisplayName("toString")
    class ToString {

        @Test
        @DisplayName("should format as 'METHOD path'")
        void shouldFormatWithMethod() {
            RouteKey key = new RouteKey("/users", "GET");
            assertThat(key.toString()).isEqualTo("GET /users");
        }

        @Test
        @DisplayName("should format as 'ALL path' when method is empty")
        void shouldFormatWithAll_WhenMethodEmpty() {
            RouteKey key = new RouteKey("/users", "");
            assertThat(key.toString()).isEqualTo("ALL /users");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/mapping/SimpleHandlerMappingTest.java` [NEW]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.SimpleBeanContainer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link SimpleHandlerMapping}.
 */
class SimpleHandlerMappingTest {

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    static class UserController {

        @RequestMapping(path = "/users", method = "GET")
        public String listUsers() {
            return "user list";
        }

        @RequestMapping(path = "/users", method = "POST")
        public String createUser() {
            return "user created";
        }
    }

    @Controller
    @RequestMapping(path = "/api")
    static class ApiController {

        @RequestMapping(path = "/status", method = "GET")
        public String status() {
            return "ok";
        }

        @RequestMapping(path = "/health")
        public String health() {
            return "healthy";
        }
    }

    static class NotAController {
        @RequestMapping(path = "/hidden", method = "GET")
        public String hidden() {
            return "should not be registered";
        }
    }

    @Controller
    static class DuplicateController {

        @RequestMapping(path = "/users", method = "GET")
        public String alsoListUsers() {
            return "duplicate!";
        }
    }

    private SimpleBeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;

    @BeforeEach
    void setUp() {
        beanContainer = new SimpleBeanContainer();
        handlerMapping = new SimpleHandlerMapping();
    }

    // ─── Handler detection ───────────────────────────────────────────

    @Nested
    @DisplayName("Handler detection")
    class HandlerDetection {

        @Test
        @DisplayName("should detect @Controller beans")
        void shouldDetectControllerBeans() {
            beanContainer.registerBean(new UserController());
            handlerMapping.init(beanContainer);

            assertThat(handlerMapping.getRegisteredMappings()).hasSize(2);
        }

        @Test
        @DisplayName("should ignore beans without @Controller")
        void shouldIgnoreNonControllerBeans() {
            beanContainer.registerBean(new NotAController());
            handlerMapping.init(beanContainer);

            assertThat(handlerMapping.getRegisteredMappings()).isEmpty();
        }

        @Test
        @DisplayName("should register all @RequestMapping methods in a controller")
        void shouldRegisterAllMappingMethods() {
            beanContainer.registerBean(new UserController());
            handlerMapping.init(beanContainer);

            var mappings = handlerMapping.getRegisteredMappings();
            assertThat(mappings).containsKey(new RouteKey("/users", "GET"));
            assertThat(mappings).containsKey(new RouteKey("/users", "POST"));
        }
    }

    // ─── Path combination ────────────────────────────────────────────

    @Nested
    @DisplayName("Path combination")
    class PathCombination {

        @Test
        @DisplayName("should combine type-level and method-level paths")
        void shouldCombineTypeLevelAndMethodLevelPaths() {
            beanContainer.registerBean(new ApiController());
            handlerMapping.init(beanContainer);

            var mappings = handlerMapping.getRegisteredMappings();
            assertThat(mappings).containsKey(new RouteKey("/api/status", "GET"));
        }

        @Test
        @DisplayName("should register method with empty HTTP method as match-any")
        void shouldRegisterEmptyMethodAsMatchAny() {
            beanContainer.registerBean(new ApiController());
            handlerMapping.init(beanContainer);

            var mappings = handlerMapping.getRegisteredMappings();
            assertThat(mappings).containsKey(new RouteKey("/api/health", ""));
        }
    }

    // ─── Handler lookup ──────────────────────────────────────────────

    @Nested
    @DisplayName("Handler lookup")
    class HandlerLookup {

        @Test
        @DisplayName("should find handler for exact path and method match")
        void shouldFindHandler_WhenExactMatch() {
            beanContainer.registerBean(new UserController());
            handlerMapping.init(beanContainer);

            HandlerMethod handler = handlerMapping.lookupHandler("/users", "GET");

            assertThat(handler).isNotNull();
            assertThat(handler.getMethod().getName()).isEqualTo("listUsers");
        }

        @Test
        @DisplayName("should find correct handler when multiple methods on same path")
        void shouldFindCorrectHandler_WhenMultipleMethodsOnSamePath() {
            beanContainer.registerBean(new UserController());
            handlerMapping.init(beanContainer);

            HandlerMethod getHandler = handlerMapping.lookupHandler("/users", "GET");
            HandlerMethod postHandler = handlerMapping.lookupHandler("/users", "POST");

            assertThat(getHandler.getMethod().getName()).isEqualTo("listUsers");
            assertThat(postHandler.getMethod().getName()).isEqualTo("createUser");
        }

        @Test
        @DisplayName("should return null when no handler found")
        void shouldReturnNull_WhenNoHandlerFound() {
            beanContainer.registerBean(new UserController());
            handlerMapping.init(beanContainer);

            HandlerMethod handler = handlerMapping.lookupHandler("/nonexistent", "GET");

            assertThat(handler).isNull();
        }

        @Test
        @DisplayName("should return null when path matches but method doesn't")
        void shouldReturnNull_WhenMethodDoesNotMatch() {
            beanContainer.registerBean(new UserController());
            handlerMapping.init(beanContainer);

            HandlerMethod handler = handlerMapping.lookupHandler("/users", "DELETE");

            assertThat(handler).isNull();
        }

        @Test
        @DisplayName("should find handler with combined type + method paths")
        void shouldFindHandler_WhenCombinedPaths() {
            beanContainer.registerBean(new ApiController());
            handlerMapping.init(beanContainer);

            HandlerMethod handler = handlerMapping.lookupHandler("/api/status", "GET");

            assertThat(handler).isNotNull();
            assertThat(handler.getMethod().getName()).isEqualTo("status");
        }

        @Test
        @DisplayName("should match any-method handler for any HTTP method")
        void shouldMatchAnyMethodHandler() {
            beanContainer.registerBean(new ApiController());
            handlerMapping.init(beanContainer);

            assertThat(handlerMapping.lookupHandler("/api/health", "GET")).isNotNull();
            assertThat(handlerMapping.lookupHandler("/api/health", "POST")).isNotNull();
            assertThat(handlerMapping.lookupHandler("/api/health", "PUT")).isNotNull();
        }
    }

    // ─── Duplicate detection ─────────────────────────────────────────

    @Nested
    @DisplayName("Duplicate detection")
    class DuplicateDetection {

        @Test
        @DisplayName("should throw on ambiguous mapping")
        void shouldThrowOnAmbiguousMapping() {
            beanContainer.registerBean(new UserController());
            beanContainer.registerBean(new DuplicateController());

            assertThatThrownBy(() -> handlerMapping.init(beanContainer))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("Ambiguous mapping");
        }
    }

    // ─── Multiple controllers ────────────────────────────────────────

    @Nested
    @DisplayName("Multiple controllers")
    class MultipleControllers {

        @Test
        @DisplayName("should register handlers from multiple controllers")
        void shouldRegisterFromMultipleControllers() {
            beanContainer.registerBean(new UserController());
            beanContainer.registerBean(new ApiController());
            handlerMapping.init(beanContainer);

            assertThat(handlerMapping.getRegisteredMappings()).hasSize(4);
            assertThat(handlerMapping.lookupHandler("/users", "GET")).isNotNull();
            assertThat(handlerMapping.lookupHandler("/api/status", "GET")).isNotNull();
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/servlet/SimpleDispatcherServletTest.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.container.SimpleBeanContainer;
import jakarta.servlet.ServletException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Unit tests for {@link SimpleDispatcherServlet}.
 *
 * Uses mock HttpServletRequest/Response to test the servlet without
 * starting an actual server. The integration test in
 * {@code EmbeddedTomcatIntegrationTest} covers the full HTTP stack.
 */
class SimpleDispatcherServletTest {

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    static class TestController {

        @RequestMapping(path = "/test", method = "GET")
        public String handleGet() {
            return "test response";
        }
    }

    private SimpleBeanContainer beanContainer;
    private SimpleDispatcherServlet servlet;

    @BeforeEach
    void setUp() throws ServletException {
        beanContainer = new SimpleBeanContainer();
        beanContainer.registerBean(new TestController());
        servlet = new SimpleDispatcherServlet(beanContainer);
        servlet.init();
    }

    // ─── Construction ────────────────────────────────────────────────

    @Nested
    @DisplayName("Construction")
    class Construction {

        @Test
        @DisplayName("should store the bean container")
        void shouldStoreBeanContainer() {
            assertThat(servlet.getBeanContainer()).isSameAs(beanContainer);
        }

        @Test
        @DisplayName("should accept BeanContainer interface")
        void shouldAcceptBeanContainerInterface() {
            BeanContainer container = new SimpleBeanContainer();
            var ds = new SimpleDispatcherServlet(container);
            assertThat(ds.getBeanContainer()).isSameAs(container);
        }
    }

    // ─── doDispatch ──────────────────────────────────────────────────

    @Nested
    @DisplayName("doDispatch")
    class DoDispatch {

        @Test
        @DisplayName("should dispatch to matched handler and return its result")
        void shouldDispatchToMatchedHandler() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));
            when(request.getRequestURI()).thenReturn("/test");
            when(request.getMethod()).thenReturn("GET");

            servlet.doDispatch(request, response);

            assertThat(stringWriter.toString()).isEqualTo("test response");
        }

        @Test
        @DisplayName("should set content type to text/plain")
        void shouldSetContentType() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(response.getWriter()).thenReturn(new PrintWriter(new StringWriter()));
            when(request.getRequestURI()).thenReturn("/test");
            when(request.getMethod()).thenReturn("GET");

            servlet.doDispatch(request, response);

            verify(response).setContentType("text/plain");
        }

        @Test
        @DisplayName("should set character encoding to UTF-8")
        void shouldSetCharacterEncoding() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(response.getWriter()).thenReturn(new PrintWriter(new StringWriter()));
            when(request.getRequestURI()).thenReturn("/test");
            when(request.getMethod()).thenReturn("GET");

            servlet.doDispatch(request, response);

            verify(response).setCharacterEncoding("UTF-8");
        }

        @Test
        @DisplayName("should send 404 when no handler found")
        void shouldSend404_WhenNoHandlerFound() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(request.getRequestURI()).thenReturn("/nonexistent");
            when(request.getMethod()).thenReturn("GET");

            servlet.doDispatch(request, response);

            verify(response).sendError(eq(404), anyString());
        }
    }

    // ─── service() routing ───────────────────────────────────────────

    @Nested
    @DisplayName("service() routing")
    class ServiceRouting {

        @Test
        @DisplayName("should route GET through doDispatch to handler")
        void shouldRouteGetToDoDispatch() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));
            when(request.getRequestURI()).thenReturn("/test");
            when(request.getMethod()).thenReturn("GET");

            servlet.service(request, response);

            assertThat(stringWriter.toString()).isEqualTo("test response");
        }

        @Test
        @DisplayName("should route POST through doDispatch — returns 404 for GET-only handler")
        void shouldRoutePostToDoDispatch() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(request.getRequestURI()).thenReturn("/test");
            when(request.getMethod()).thenReturn("POST");

            servlet.service(request, response);

            verify(response).sendError(eq(404), anyString());
        }
    }

    // ─── initStrategies ──────────────────────────────────────────────

    @Nested
    @DisplayName("initStrategies")
    class InitStrategies {

        @Test
        @DisplayName("should complete without error on empty container")
        void shouldCompleteWithoutError_WhenContainerEmpty() throws ServletException {
            var freshServlet = new SimpleDispatcherServlet(new SimpleBeanContainer());
            freshServlet.init();
            assertThat(freshServlet.getBeanContainer()).isNotNull();
        }

        @Test
        @DisplayName("should initialize handler mapping during init")
        void shouldInitializeHandlerMapping() throws ServletException {
            assertThat(servlet.getHandlerMapping()).isNotNull();
            assertThat(servlet.getHandlerMapping().getRegisteredMappings()).isNotEmpty();
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/HandlerMappingIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for the handler mapping feature.
 * Starts a real Tomcat server and makes real HTTP requests.
 */
class HandlerMappingIntegrationTest {

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    static class GreetingController {

        @RequestMapping(path = "/hello", method = "GET")
        public String hello() {
            return "Hello, World!";
        }

        @RequestMapping(path = "/goodbye", method = "GET")
        public String goodbye() {
            return "Goodbye!";
        }
    }

    @Controller
    @RequestMapping(path = "/api/v1")
    static class VersionedApiController {

        @RequestMapping(path = "/ping", method = "GET")
        public String ping() {
            return "pong";
        }

        @RequestMapping(path = "/echo", method = "POST")
        public String echo() {
            return "echoed";
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new GreetingController());
        container.registerBean(new VersionedApiController());

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        baseUrl = "http://localhost:" + tomcat.getPort();
        httpClient = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        tomcat.stop();
    }

    // ─── Successful routing ──────────────────────────────────────────

    @Test
    @DisplayName("should route GET /hello to GreetingController#hello")
    void shouldRouteGetHello() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello, World!");
    }

    @Test
    @DisplayName("should route GET /goodbye to GreetingController#goodbye")
    void shouldRouteGetGoodbye() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/goodbye")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Goodbye!");
    }

    @Test
    @DisplayName("should combine type-level path prefix with method path")
    void shouldCombineTypeLevelPathPrefix() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/api/v1/ping")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("pong");
    }

    @Test
    @DisplayName("should route POST to correct handler")
    void shouldRoutePostToCorrectHandler() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/api/v1/echo"))
                        .POST(HttpRequest.BodyPublishers.noBody()).build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("echoed");
    }

    // ─── 404 handling ────────────────────────────────────────────────

    @Test
    @DisplayName("should return 404 for unmatched path")
    void shouldReturn404_WhenPathNotFound() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/nonexistent")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(404);
    }

    @Test
    @DisplayName("should return 404 when path matches but HTTP method doesn't")
    void shouldReturn404_WhenMethodDoesNotMatch() throws Exception {
        // /hello is mapped to GET only, POST should not match
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello"))
                        .POST(HttpRequest.BodyPublishers.noBody()).build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(404);
    }

    // ─── Content type ────────────────────────────────────────────────

    @Test
    @DisplayName("should set content type to text/plain")
    void shouldSetContentTypeToTextPlain() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.headers().firstValue("Content-Type").orElse(""))
                .contains("text/plain");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/EmbeddedTomcatIntegrationTest.java` [MODIFIED]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.*;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration test that starts a real embedded Tomcat, sends actual HTTP
 * requests, and verifies responses. This proves the entire pipeline works:
 *
 * <pre>
 *   HTTP client → Tomcat → SimpleDispatcherServlet.service()
 *     → doDispatch() → handlerMapping.lookupHandler() → invokeHandler() → response
 * </pre>
 *
 * Uses port 0 so Tomcat picks a random available port — safe for parallel test runs.
 */
class EmbeddedTomcatIntegrationTest {

    // ─── Test controller ─────────────────────────────────────────────

    @Controller
    static class RootController {

        @RequestMapping(path = "/", method = "GET")
        public String root() {
            return "Hello from DispatcherServlet";
        }

        @RequestMapping(path = "/submit", method = "POST")
        public String submit() {
            return "submitted";
        }

        @RequestMapping(path = "/resource", method = "PUT")
        public String putResource() {
            return "updated";
        }

        @RequestMapping(path = "/resource/1", method = "DELETE")
        public String deleteResource() {
            return "deleted";
        }
    }

    private EmbeddedTomcat server;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new RootController());
        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        server = new EmbeddedTomcat(0, servlet);
        server.start();

        baseUrl = "http://localhost:" + server.getPort();
        httpClient = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        if (server != null) {
            server.stop();
        }
    }

    // ─── GET requests ────────────────────────────────────────────────

    @Test
    @DisplayName("should respond to GET /")
    void shouldRespondToGetRoot() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/"))
                .GET()
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
    }

    @Test
    @DisplayName("should return 404 for unregistered path")
    void shouldReturn404ForUnregisteredPath() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/any/path"))
                .GET()
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(404);
    }

    @Test
    @DisplayName("should return text/plain content type")
    void shouldReturnTextPlainContentType() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/"))
                .GET()
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.headers().firstValue("Content-Type"))
                .isPresent()
                .hasValueSatisfying(ct -> assertThat(ct).contains("text/plain"));
    }

    // ─── POST requests ───────────────────────────────────────────────

    @Test
    @DisplayName("should respond to POST requests")
    void shouldRespondToPostRequests() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/submit"))
                .POST(HttpRequest.BodyPublishers.ofString("data"))
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("submitted");
    }

    // ─── PUT requests ────────────────────────────────────────────────

    @Test
    @DisplayName("should respond to PUT requests")
    void shouldRespondToPutRequests() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/resource"))
                .PUT(HttpRequest.BodyPublishers.ofString("data"))
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("updated");
    }

    // ─── DELETE requests ─────────────────────────────────────────────

    @Test
    @DisplayName("should respond to DELETE requests")
    void shouldRespondToDeleteRequests() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/resource/1")
                ).DELETE()
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("deleted");
    }

    // ─── Server lifecycle ────────────────────────────────────────────

    @Test
    @DisplayName("should allocate a real port when started with port 0")
    void shouldAllocateRealPort_WhenStartedWithZero() {
        assertThat(server.getPort()).isGreaterThan(0);
    }

    @Test
    @DisplayName("should expose the dispatcher servlet")
    void shouldExposeDispatcherServlet() {
        assertThat(server.getDispatcherServlet()).isNotNull();
        assertThat(server.getDispatcherServlet().getBeanContainer()).isNotNull();
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@Controller`** | Marker annotation that identifies a class as a web controller whose methods can handle HTTP requests |
| **`@RequestMapping`** | Annotation on classes (base path prefix) and methods (specific endpoint) mapping URL + HTTP method to a handler |
| **`HandlerMethod`** | Value object wrapping a controller bean instance + the specific `java.lang.reflect.Method` to invoke |
| **`RouteKey`** | Composite key of (normalized path + uppercase HTTP method) used as the registry key |
| **`SimpleHandlerMapping`** | Scans beans at startup for `@Controller`/`@RequestMapping`, builds a `Map<RouteKey, HandlerMethod>` registry, and provides request-time lookup |
| **Front-load scanning** | Do the expensive reflection work once at startup; request-time lookup is a simple map scan |
| **Strategy pattern** | `DispatcherServlet` delegates to `HandlerMapping` — the routing strategy is pluggable |

**Next: Chapter 4 — Handler Method Invocation** — Currently `doDispatch()` does both lookup AND invocation. Ch04 will extract invocation into a `HandlerAdapter`, following the Strategy pattern so different handler types can be invoked differently. This is also where we'll decouple "finding the handler" from "calling it".
