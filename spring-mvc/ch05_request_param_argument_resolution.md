# Chapter 5: @RequestParam Argument Resolution

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The `SimpleHandlerAdapter` invokes handler methods via `method.invoke(bean)` with zero arguments — only no-arg controller methods work | Cannot extract data from the HTTP request (query parameters, form data) and pass it to handler methods | Introduce the argument resolver pattern so `@RequestParam("name") String name` automatically extracts `?name=Alice` from the query string |

---

## 5.1 The Integration Point

The integration point is `SimpleHandlerAdapter.handle()` — the method that sits between the dispatcher and the controller. Currently it calls `method.invoke(bean)` with no arguments. We need to insert an argument resolution step **before** invocation.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Add a `resolveArguments()` step before `invokeHandlerMethod()`, and pass the resolved args to `method.invoke()`

```java
@Override
public void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    HandlerMethod handlerMethod = (HandlerMethod) handler;

    // Step 1: Resolve method arguments from the request (NEW — ch05)
    Object[] args = resolveArguments(handlerMethod, request);

    // Step 2: Invoke the controller method with resolved arguments
    Object result = invokeHandlerMethod(handlerMethod, args);

    // Step 3: Write the result (unchanged from ch04)
    writeResponse(response, result);
}
```

The `resolveArguments()` method iterates each `MethodParameter` and delegates to a `HandlerMethodArgumentResolverComposite`:

```java
private Object[] resolveArguments(HandlerMethod handlerMethod, HttpServletRequest request) throws Exception {
    MethodParameter[] parameters = handlerMethod.getMethodParameters();
    if (parameters.length == 0) {
        return new Object[0];
    }

    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter param = parameters[i];
        if (argumentResolvers.supportsParameter(param)) {
            args[i] = argumentResolvers.resolveArgument(param, request);
        } else {
            throw new IllegalStateException(
                    "No suitable resolver for argument " + i + " of type '"
                            + param.getParameterType().getName()
                            + "' on method " + handlerMethod);
        }
    }
    return args;
}
```

Two key decisions here:

1. **Why resolve in the adapter, not the dispatcher?** The dispatcher doesn't know about method signatures — it just hands a handler to an adapter. Argument resolution is part of *how* the handler gets invoked, which is the adapter's responsibility. The real `RequestMappingHandlerAdapter` follows the same separation.

2. **Why a composite of resolvers instead of one big resolver?** Each resolver knows one annotation (`@RequestParam`, `@PathVariable`, `@RequestBody`). Adding a new parameter type means adding a resolver, not modifying existing code. This is the Open/Closed Principle in action.

This connects **the resolver chain** to **the handler invocation pipeline**. To make it work, we need to build:
- `MethodParameter` — wraps a `java.lang.reflect.Parameter` so resolvers can inspect type and annotations
- `HandlerMethodArgumentResolver` — the strategy interface that each resolver implements
- `HandlerMethodArgumentResolverComposite` — the composite that delegates to the right resolver
- `@RequestParam` — the annotation that marks parameters for query string extraction
- `RequestParamArgumentResolver` — the concrete resolver for `@RequestParam`

We also need to enhance `HandlerMethod` to expose `MethodParameter[]`.

## 5.2 The @RequestParam Annotation

**New file:** `src/main/java/com/simplespringmvc/annotation/RequestParam.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestParam {

    /**
     * The name of the request parameter to bind to.
     * If empty, the method parameter name is used as a fallback
     * (requires the -parameters compiler flag).
     */
    String value() default "";
}
```

The real `@RequestParam` has `value`, `name` (aliased), `required`, and `defaultValue`. We start with just `value` — enough to demonstrate the resolver pattern.

## 5.3 The MethodParameter Wrapper

**New file:** `src/main/java/com/simplespringmvc/mapping/MethodParameter.java`

```java
package com.simplespringmvc.mapping;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

public class MethodParameter {

    private final Method method;
    private final Parameter parameter;
    private final int parameterIndex;

    public MethodParameter(Method method, int parameterIndex) {
        this.method = method;
        this.parameter = method.getParameters()[parameterIndex];
        this.parameterIndex = parameterIndex;
    }

    public Method getMethod() { return method; }
    public Parameter getParameter() { return parameter; }
    public int getParameterIndex() { return parameterIndex; }

    public Class<?> getParameterType() {
        return parameter.getType();
    }

    public String getParameterName() {
        return parameter.getName();
    }

    public <A extends Annotation> A getParameterAnnotation(Class<A> annotationType) {
        return parameter.getAnnotation(annotationType);
    }

    public boolean hasParameterAnnotation(Class<? extends Annotation> annotationType) {
        return parameter.isAnnotationPresent(annotationType);
    }
}
```

This wraps `java.lang.reflect.Parameter` to provide a clean API for resolvers. The real `MethodParameter` (1000+ lines in `spring-core`) handles generic types, nesting levels, Kotlin coroutines, and pluggable name discoverers. We delegate to the JDK's `Parameter.getName()` which works with the `-parameters` compiler flag.

**Modifying:** `src/main/java/com/simplespringmvc/mapping/HandlerMethod.java`
**Change:** Add `MethodParameter[]` field initialized eagerly in the constructor, and `getMethodParameters()` accessor

```java
private final MethodParameter[] parameters;

public HandlerMethod(Object bean, Method method) {
    // ... existing validation ...
    this.bean = bean;
    this.method = method;
    this.parameters = initMethodParameters();  // NEW
}

private MethodParameter[] initMethodParameters() {
    int count = method.getParameterCount();
    MethodParameter[] result = new MethodParameter[count];
    for (int i = 0; i < count; i++) {
        result[i] = new MethodParameter(method, i);
    }
    return result;
}

public MethodParameter[] getMethodParameters() {
    return parameters;
}
```

## 5.4 The Resolver Interface and Composite

**New file:** `src/main/java/com/simplespringmvc/adapter/HandlerMethodArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

public interface HandlerMethodArgumentResolver {

    boolean supportsParameter(MethodParameter parameter);

    Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception;
}
```

The real interface takes `(MethodParameter, ModelAndViewContainer, NativeWebRequest, WebDataBinderFactory)`. We simplify to `(MethodParameter, HttpServletRequest)` — no model or binder needed yet.

**New file:** `src/main/java/com/simplespringmvc/adapter/HandlerMethodArgumentResolverComposite.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {

    private final List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
    private final Map<MethodParameter, HandlerMethodArgumentResolver> resolverCache =
            new ConcurrentHashMap<>(256);

    public HandlerMethodArgumentResolverComposite addResolver(HandlerMethodArgumentResolver resolver) {
        this.resolvers.add(resolver);
        return this;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return getArgumentResolver(parameter) != null;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
        if (resolver == null) {
            throw new IllegalArgumentException("Unsupported parameter type [" +
                    parameter.getParameterType().getName() + "]");
        }
        return resolver.resolveArgument(parameter, request);
    }

    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
        HandlerMethodArgumentResolver cached = this.resolverCache.get(parameter);
        if (cached != null) return cached;

        for (HandlerMethodArgumentResolver resolver : this.resolvers) {
            if (resolver.supportsParameter(parameter)) {
                this.resolverCache.put(parameter, resolver);
                return resolver;
            }
        }
        return null;
    }
}
```

The `ConcurrentHashMap` cache is lifted directly from the real `HandlerMethodArgumentResolverComposite` (line 103). Since controller methods are invoked repeatedly with the same parameter shapes, caching which resolver handles which parameter avoids re-scanning the list on every request.

**New file:** `src/main/java/com/simplespringmvc/adapter/RequestParamArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

public class RequestParamArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestParam.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String parameterName = getParameterName(parameter);
        String value = request.getParameter(parameterName);

        if (value == null) {
            throw new IllegalStateException(
                    "Missing required request parameter '" + parameterName + "'");
        }
        return value;
    }

    private String getParameterName(MethodParameter parameter) {
        RequestParam annotation = parameter.getParameterAnnotation(RequestParam.class);
        return (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
    }
}
```

The name resolution logic: use the annotation's `value()` if specified, otherwise fall back to the Java parameter name. This fallback is why the `-parameters` compiler flag matters — without it, you'd get names like `arg0`.

## 5.5 Try It Yourself

<details>
<summary>Challenge: Implement a custom argument resolver that injects the HttpServletRequest itself</summary>

Some controller methods need the raw `HttpServletRequest`. Write a `ServletRequestArgumentResolver` that supports parameters of type `HttpServletRequest` and passes the request object through.

Hint: Check `parameter.getParameterType()` in `supportsParameter()`.

```java
public class ServletRequestArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return HttpServletRequest.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) {
        return request;
    }
}
```

To register it, add it to the `SimpleHandlerAdapter` constructor:
```java
argumentResolvers.addResolver(new ServletRequestArgumentResolver());
```

</details>

<details>
<summary>Challenge: Add the composite's ConcurrentHashMap caching from scratch</summary>

Why does the composite use a `ConcurrentHashMap<MethodParameter, HandlerMethodArgumentResolver>` instead of just scanning the list each time?

Think about: A typical web app handles hundreds of requests per second. Each request resolves the same parameters on the same controller method. The resolver list scan is O(n) per parameter, but the cache makes repeated lookups O(1).

The real framework does exactly the same thing — see `HandlerMethodArgumentResolverComposite.java` line 103.

```java
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    // Check cache first — O(1)
    HandlerMethodArgumentResolver cached = this.resolverCache.get(parameter);
    if (cached != null) {
        return cached;
    }
    // Linear scan — O(n), but only happens once per parameter
    for (HandlerMethodArgumentResolver resolver : this.resolvers) {
        if (resolver.supportsParameter(parameter)) {
            this.resolverCache.put(parameter, resolver);
            return resolver;
        }
    }
    return null;
}
```

</details>

## 5.6 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/mapping/MethodParameterTest.java`

```java
@Test
@DisplayName("should return the declared parameter type")
void shouldReturnParameterType() throws Exception {
    Method method = SampleController.class.getMethod("greet", String.class);
    MethodParameter param = new MethodParameter(method, 0);

    assertThat(param.getParameterType()).isEqualTo(String.class);
}

@Test
@DisplayName("should detect @RequestParam annotation")
void shouldDetectRequestParam() throws Exception {
    Method method = SampleController.class.getMethod("greet", String.class);
    MethodParameter param = new MethodParameter(method, 0);

    assertThat(param.hasParameterAnnotation(RequestParam.class)).isTrue();
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/RequestParamArgumentResolverTest.java`

```java
@Test
@DisplayName("should resolve parameter using explicit annotation name")
void shouldResolveByExplicitName() throws Exception {
    Method method = TestController.class.getMethod("withExplicitName", String.class);
    MethodParameter param = new MethodParameter(method, 0);
    when(request.getParameter("q")).thenReturn("spring mvc");

    Object result = resolver.resolveArgument(param, request);

    assertThat(result).isEqualTo("spring mvc");
}

@Test
@DisplayName("should throw when required parameter is missing")
void shouldThrow_WhenParameterMissing() throws Exception {
    Method method = TestController.class.getMethod("withExplicitName", String.class);
    MethodParameter param = new MethodParameter(method, 0);
    when(request.getParameter("q")).thenReturn(null);

    assertThatThrownBy(() -> resolver.resolveArgument(param, request))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Missing required request parameter");
}
```

**Modified file:** `src/test/java/com/simplespringmvc/adapter/SimpleHandlerAdapterTest.java` — added `@RequestParam` tests:

```java
@Test
@DisplayName("should resolve single @RequestParam and pass to handler")
void shouldResolveSingleRequestParam() throws Exception {
    ParamController controller = new ParamController();
    Method method = controller.getClass().getMethod("greet", String.class);
    HandlerMethod handlerMethod = new HandlerMethod(controller, method);
    when(request.getParameter("name")).thenReturn("Alice");

    adapter.handle(request, response, handlerMethod);

    assertThat(responseBody.toString()).isEqualTo("Hello, Alice!");
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/RequestParamIntegrationTest.java`

Tests the full flow through real HTTP: Tomcat → DispatcherServlet → HandlerMapping → HandlerAdapter (with resolvers) → Controller.

```java
@Test
@DisplayName("should resolve @RequestParam from query string")
void shouldResolveRequestParam() throws Exception {
    HttpResponse<String> response = httpClient.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/greet?name=Alice")).GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("Hello, Alice!");
}

@Test
@DisplayName("should still handle no-arg handler methods after adding resolver support")
void shouldHandleNoArgMethods() throws Exception {
    HttpResponse<String> response = httpClient.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/ping")).GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("pong");
}
```

**Run:** `./gradlew test` — expected: all 129 tests pass (including all prior features' tests)

---

## 5.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why Strategy + Composite over a switch statement?** A `switch` on annotation type would work for 2-3 parameter types. But Spring MVC has 30+ resolvers. The Strategy pattern lets each resolver be self-contained — it declares what it supports and how to resolve it. The Composite groups them behind a single interface. Adding `@PathVariable` (ch07) or `@RequestBody` (ch08) means creating a new class and registering it — zero changes to existing code. This is the Open/Closed Principle in its purest form.
> - **When to skip the pattern:** If your framework will only ever have one or two parameter sources, a strategy chain adds indirection without benefit. The pattern pays off when the number of resolvers is large or user-extensible.
> - **Real-world parallel:** The real `RequestMappingHandlerAdapter` registers resolvers in `getDefaultArgumentResolvers()` (line 644). Notice it registers `RequestParamMethodArgumentResolver` twice — once for annotated params and once at the end as a catch-all for unannotated simple types. Order matters.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why does `MethodParameter` exist instead of just using `java.lang.reflect.Parameter`?** Spring's `MethodParameter` predates JDK 8's `Parameter` class (it was introduced in Spring 3.1, JDK 7 era). But even today it adds value: it provides a single object that encapsulates the method, parameter index, nesting level, and annotation synthesis. The Composite's `ConcurrentHashMap` cache uses `MethodParameter` as the key, making resolver lookups O(1) after the first request.
> - **The `-parameters` flag trade-off:** Without it, `@RequestParam String name` would need explicit `@RequestParam("name")` everywhere. Spring Boot enables this flag by default, making implicit name resolution the norm. Our `build.gradle` already has `options.compilerArgs.add('-parameters')` for the same reason.
> -----------------------------------------------------------

## 5.8 What We Enhanced

| Aspect | Before (ch04) | Current (ch05) | Real Framework |
|--------|---------------|----------------|----------------|
| Method invocation | `method.invoke(bean)` — no arguments, only no-arg handlers work | `method.invoke(bean, args)` — arguments resolved from request via resolver chain | `InvocableHandlerMethod.invokeForRequest()` resolves args, invokes, handles Kotlin/bridged methods (`InvocableHandlerMethod.java:171`) |
| Handler parameters | `HandlerMethod` wraps bean + method only | `HandlerMethod` exposes `MethodParameter[]` eagerly built at construction | `HandlerMethod.initMethodParameters()` creates `SynthesizingMethodParameter[]` (`HandlerMethod.java:189`) |
| Request data extraction | Not supported — controller methods are blind to request content | `@RequestParam("name")` extracts query string values into String parameters | 30+ resolvers handle `@RequestParam`, `@PathVariable`, `@RequestBody`, `@RequestHeader`, multipart, model attributes, etc. (`RequestMappingHandlerAdapter.java:644`) |
| Argument resolution | None | Strategy + Composite pattern: `HandlerMethodArgumentResolver` + `HandlerMethodArgumentResolverComposite` with `ConcurrentHashMap` caching | Same pattern, same caching strategy (`HandlerMethodArgumentResolverComposite.java:103`) |

## 5.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `MethodParameter` | `MethodParameter` | `MethodParameter.java:55` | Real supports generic type nesting, Kotlin nullability, `ParameterNameDiscoverer` chain, constructor params |
| `HandlerMethodArgumentResolver` | `HandlerMethodArgumentResolver` | `HandlerMethodArgumentResolver.java:38` | Real takes `(MethodParameter, ModelAndViewContainer, NativeWebRequest, WebDataBinderFactory)` |
| `HandlerMethodArgumentResolverComposite` | `HandlerMethodArgumentResolverComposite` | `HandlerMethodArgumentResolverComposite.java:42` | Same pattern and caching; real has `clear()`, bulk `addResolvers()` |
| `RequestParamArgumentResolver` | `RequestParamMethodArgumentResolver` | `RequestParamMethodArgumentResolver.java:81` | Real extends `AbstractNamedValueMethodArgumentResolver` (shared required/default logic), handles multipart, supports "default resolution mode" for unannotated simple types |
| `@RequestParam` annotation | `@RequestParam` | `RequestParam.java:66` | Real has `value`/`name` aliases, `required`, `defaultValue` |
| `SimpleHandlerAdapter.resolveArguments()` | `InvocableHandlerMethod.getMethodArgumentValues()` | `InvocableHandlerMethod.java:186` | Real handles provided argument values, `@Nullable` defaults, Kotlin optional params |
| `HandlerMethod.getMethodParameters()` | `HandlerMethod.getMethodParameters()` | `HandlerMethod.java:247` | Real returns `SynthesizingMethodParameter[]` with annotation attribute merging |

## 5.10 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/RequestParam.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation indicating that a method parameter should be bound to a web
 * request parameter (query string or form data).
 *
 * Maps to: {@code org.springframework.web.bind.annotation.RequestParam}
 *
 * Example:
 * <pre>
 * {@literal @}RequestMapping(path = "/greet", method = "GET")
 * public String greet({@literal @}RequestParam("name") String name) {
 *     return "Hello, " + name;
 * }
 * </pre>
 *
 * When no value is specified, the parameter name from the method signature is
 * used (requires {@code -parameters} compiler flag).
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No {@code required} attribute — all parameters are required (ch09 adds defaults)</li>
 *   <li>No {@code defaultValue} attribute</li>
 *   <li>No support for Map parameters (all query params as Map)</li>
 *   <li>No multipart file support</li>
 * </ul>
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestParam {

    /**
     * The name of the request parameter to bind to.
     * If empty, the method parameter name is used as a fallback
     * (requires the {@code -parameters} compiler flag).
     */
    String value() default "";
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/MethodParameter.java` [NEW]

```java
package com.simplespringmvc.mapping;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

/**
 * Wraps a single method parameter, providing access to its type, annotations,
 * and name. Designed to be the common currency that argument resolvers inspect.
 *
 * Maps to: {@code org.springframework.core.MethodParameter}
 *
 * The real MethodParameter is a 1000+ line class in spring-core that supports:
 * <ul>
 *   <li>Nested generic type resolution (e.g., element type of {@code List<String>})</li>
 *   <li>Constructor parameters (not just method parameters)</li>
 *   <li>Kotlin suspend function detection</li>
 *   <li>Optional/nullable type handling</li>
 *   <li>ParameterNameDiscoverer pluggable name resolution</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Only wraps method parameters (not constructor parameters)</li>
 *   <li>Uses {@code java.lang.reflect.Parameter.getName()} directly (requires {@code -parameters} flag)</li>
 *   <li>No generic type resolution — just the raw parameter type</li>
 *   <li>No nesting level support</li>
 * </ul>
 */
public class MethodParameter {

    private final Method method;
    private final Parameter parameter;
    private final int parameterIndex;

    public MethodParameter(Method method, int parameterIndex) {
        this.method = method;
        this.parameter = method.getParameters()[parameterIndex];
        this.parameterIndex = parameterIndex;
    }

    /**
     * The method that declares this parameter.
     */
    public Method getMethod() {
        return method;
    }

    /**
     * The underlying {@link Parameter} reflective object.
     */
    public Parameter getParameter() {
        return parameter;
    }

    /**
     * Zero-based index of this parameter in the method signature.
     */
    public int getParameterIndex() {
        return parameterIndex;
    }

    /**
     * The declared type of this parameter.
     *
     * Maps to: {@code MethodParameter.getParameterType()} (line 504)
     */
    public Class<?> getParameterType() {
        return parameter.getType();
    }

    /**
     * The name of this parameter as declared in source code.
     * Requires the {@code -parameters} compiler flag; otherwise returns
     * synthetic names like "arg0", "arg1".
     *
     * Maps to: {@code MethodParameter.getParameterName()} (line 753)
     * Real version uses a ParameterNameDiscoverer chain that can read
     * bytecode debug info or annotation hints.
     */
    public String getParameterName() {
        return parameter.getName();
    }

    /**
     * Return the annotation of the given type on this parameter, if present.
     *
     * Maps to: {@code MethodParameter.getParameterAnnotation()} (line 644)
     */
    public <A extends Annotation> A getParameterAnnotation(Class<A> annotationType) {
        return parameter.getAnnotation(annotationType);
    }

    /**
     * Check if this parameter has the given annotation.
     *
     * Maps to: {@code MethodParameter.hasParameterAnnotation()} (line 660)
     */
    public boolean hasParameterAnnotation(Class<? extends Annotation> annotationType) {
        return parameter.isAnnotationPresent(annotationType);
    }

    @Override
    public String toString() {
        return method.getDeclaringClass().getSimpleName() + "#" + method.getName()
                + " parameter " + parameterIndex + " [" + getParameterType().getSimpleName()
                + " " + getParameterName() + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerMethodArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Strategy interface for resolving method parameters into argument values
 * from a given HTTP request.
 *
 * Maps to: {@code org.springframework.web.method.support.HandlerMethodArgumentResolver}
 *
 * Each implementation knows how to extract one kind of argument:
 * <ul>
 *   <li>{@code RequestParamArgumentResolver} — extracts {@code @RequestParam} values from query strings</li>
 *   <li>Future: {@code PathVariableArgumentResolver} — extracts {@code @PathVariable} values (ch07)</li>
 *   <li>Future: {@code RequestBodyArgumentResolver} — deserializes {@code @RequestBody} JSON (ch08)</li>
 * </ul>
 *
 * The real framework's resolveArgument() takes four parameters:
 * {@code (MethodParameter, ModelAndViewContainer, NativeWebRequest, WebDataBinderFactory)}.
 * We simplify to {@code (MethodParameter, HttpServletRequest)} — no model or binder needed yet.
 *
 * Simplifications:
 * <ul>
 *   <li>Takes {@code HttpServletRequest} instead of {@code NativeWebRequest}</li>
 *   <li>No {@code ModelAndViewContainer} parameter (ch13 adds view/model support)</li>
 *   <li>No {@code WebDataBinderFactory} parameter (ch15 adds data binding)</li>
 * </ul>
 */
public interface HandlerMethodArgumentResolver {

    /**
     * Whether this resolver supports the given parameter.
     * Called before {@link #resolveArgument} to find the right resolver.
     *
     * @param parameter the method parameter to check
     * @return true if this resolver can resolve a value for this parameter
     */
    boolean supportsParameter(MethodParameter parameter);

    /**
     * Resolve a method parameter into an argument value from the given request.
     *
     * @param parameter the method parameter to resolve
     * @param request   the current HTTP request
     * @return the resolved argument value (may be null)
     * @throws Exception if resolution fails
     */
    Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerMethodArgumentResolverComposite.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Delegates to a list of registered {@link HandlerMethodArgumentResolver}s
 * and caches the resolver-per-parameter mapping for faster repeated lookups.
 *
 * Maps to: {@code org.springframework.web.method.support.HandlerMethodArgumentResolverComposite}
 *
 * This is the Composite pattern applied to argument resolution: the adapter
 * holds ONE composite resolver, which internally fans out to many specialized
 * resolvers. The adapter never knows which resolver actually handled a parameter.
 *
 * <h3>Caching:</h3>
 * The real framework caches the mapping from {@code MethodParameter → Resolver}
 * in a {@code ConcurrentHashMap<MethodParameter, HandlerMethodArgumentResolver>}
 * (line 103). Since controller methods are invoked repeatedly with the same
 * parameter shapes, this avoids re-scanning the resolver list on every request.
 * We replicate this optimization.
 *
 * Simplifications:
 * <ul>
 *   <li>No custom resolver registration API (ch17 adds component scanning for resolvers)</li>
 *   <li>Simplified {@code resolveArgument()} — delegates to our 2-arg version, not the 4-arg real one</li>
 * </ul>
 */
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {

    private final List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
    private final Map<MethodParameter, HandlerMethodArgumentResolver> resolverCache =
            new ConcurrentHashMap<>(256);

    /**
     * Add a resolver to the chain.
     */
    public HandlerMethodArgumentResolverComposite addResolver(HandlerMethodArgumentResolver resolver) {
        this.resolvers.add(resolver);
        return this;
    }

    /**
     * Add multiple resolvers at once.
     */
    public HandlerMethodArgumentResolverComposite addResolvers(List<? extends HandlerMethodArgumentResolver> resolvers) {
        if (resolvers != null) {
            this.resolvers.addAll(resolvers);
        }
        return this;
    }

    /**
     * Return a read-only view of registered resolvers.
     */
    public List<HandlerMethodArgumentResolver> getResolvers() {
        return Collections.unmodifiableList(this.resolvers);
    }

    /**
     * Whether any registered resolver supports this parameter.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return getArgumentResolver(parameter) != null;
    }

    /**
     * Find the supporting resolver and delegate to it.
     *
     * Maps to: {@code HandlerMethodArgumentResolverComposite.resolveArgument()} (line 106)
     *
     * @throws IllegalArgumentException if no resolver supports the parameter
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
        if (resolver == null) {
            throw new IllegalArgumentException(
                    "Unsupported parameter type [" + parameter.getParameterType().getName()
                            + "]. No argument resolver found for: " + parameter);
        }
        return resolver.resolveArgument(parameter, request);
    }

    /**
     * Find the first resolver that supports the given parameter, caching the
     * result for subsequent calls.
     *
     * Maps to: {@code HandlerMethodArgumentResolverComposite.getArgumentResolver()} (line 121)
     * The real version uses the same linear scan + ConcurrentHashMap caching strategy.
     */
    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
        HandlerMethodArgumentResolver cached = this.resolverCache.get(parameter);
        if (cached != null) {
            return cached;
        }
        for (HandlerMethodArgumentResolver resolver : this.resolvers) {
            if (resolver.supportsParameter(parameter)) {
                this.resolverCache.put(parameter, resolver);
                return resolver;
            }
        }
        return null;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/RequestParamArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves method arguments annotated with {@link RequestParam} by extracting
 * values from the HTTP request's query string (or form data).
 *
 * Maps to: {@code org.springframework.web.method.annotation.RequestParamMethodArgumentResolver}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @RequestParam}</li>
 *   <li>{@code resolveArgument()} reads the annotation's value to get the parameter name</li>
 *   <li>Falls back to the Java parameter name if no explicit name is given</li>
 *   <li>Calls {@code request.getParameter(name)} to get the value</li>
 * </ol>
 *
 * <h3>Parameter name resolution:</h3>
 * When {@code @RequestParam} has no explicit value (e.g., {@code @RequestParam String name}),
 * we fall back to the method parameter name from the Java reflection API. This requires
 * the {@code -parameters} compiler flag, which is already configured in build.gradle.
 * The real framework uses a {@code ParameterNameDiscoverer} chain that can also read
 * bytecode debug info via ASM — we rely solely on the JDK's built-in support.
 *
 * Simplifications:
 * <ul>
 *   <li>Only handles String parameters — no type conversion (ch09 adds ConversionService)</li>
 *   <li>No {@code required} flag or default value handling</li>
 *   <li>No multipart file support</li>
 *   <li>No "default resolution mode" for unannotated simple types</li>
 *   <li>Doesn't extend AbstractNamedValueMethodArgumentResolver (no shared name/default/required logic)</li>
 * </ul>
 */
public class RequestParamArgumentResolver implements HandlerMethodArgumentResolver {

    /**
     * Supports parameters annotated with {@code @RequestParam}.
     *
     * Maps to: {@code RequestParamMethodArgumentResolver.supportsParameter()} (line 128)
     * The real version also supports multipart types and (in default mode) unannotated
     * simple types. We only handle the explicit annotation case.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestParam.class);
    }

    /**
     * Extract the query parameter value from the request.
     *
     * Maps to: {@code RequestParamMethodArgumentResolver.resolveName()} (line 163)
     * which is called by the AbstractNamedValueMethodArgumentResolver template method.
     *
     * @throws IllegalStateException if the parameter is missing from the request
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String parameterName = getParameterName(parameter);
        String value = request.getParameter(parameterName);

        if (value == null) {
            throw new IllegalStateException(
                    "Missing required request parameter '" + parameterName
                            + "' for method parameter type " + parameter.getParameterType().getSimpleName());
        }

        return value;
    }

    /**
     * Determine the request parameter name: use the annotation's value() if
     * specified, otherwise fall back to the method parameter name.
     *
     * Maps to: {@code AbstractNamedValueMethodArgumentResolver.updateNamedValueInfo()} (line 168)
     * The real version has a full NamedValueInfo chain with required/defaultValue support.
     */
    private String getParameterName(MethodParameter parameter) {
        RequestParam annotation = parameter.getParameterAnnotation(RequestParam.class);
        String name = (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
        return name;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;

/**
 * The default HandlerAdapter that knows how to invoke {@link HandlerMethod}
 * instances via reflection, resolving method arguments from the request first.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 * which extends {@code AbstractHandlerMethodAdapter}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handleInternal() creates a ServletInvocableHandlerMethod
 *   3. Sets argument resolvers on the invocable method
 *   4. Sets return value handlers on the invocable method
 *   5. Calls invocableMethod.invokeAndHandle()
 *   6. invokeAndHandle() resolves arguments, invokes via reflection,
 *      handles the return value
 * </pre>
 *
 * We collapse that chain but now include argument resolution:
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handle() resolves arguments via the resolver chain
 *   3. Invokes the method via reflection with the resolved arguments
 *   4. Writes the String return value directly to the response
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No InvocableHandlerMethod wrapper — we call Method.invoke() directly</li>
 *   <li>No return value handlers — always writes toString() as text/plain (ch06 adds @ResponseBody)</li>
 *   <li>No ModelAndView return — response is written directly</li>
 *   <li>No WebDataBinderFactory</li>
 *   <li>No ModelFactory</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();

    public SimpleHandlerAdapter() {
        // Register the default argument resolvers.
        // Maps to: RequestMappingHandlerAdapter.getDefaultArgumentResolvers() (line 644)
        // Real version registers ~30 resolvers in a specific order. We start with one.
        argumentResolvers.addResolver(new RequestParamArgumentResolver());
    }

    /**
     * Create an adapter with custom resolvers added BEFORE the defaults.
     * This mirrors how the real framework's setCustomArgumentResolvers() works.
     */
    public SimpleHandlerAdapter(List<HandlerMethodArgumentResolver> customResolvers) {
        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        argumentResolvers.addResolver(new RequestParamArgumentResolver());
    }

    /**
     * This adapter supports only HandlerMethod instances — the type produced
     * by our SimpleHandlerMapping when scanning @RequestMapping methods.
     *
     * Maps to: {@code AbstractHandlerMethodAdapter.supports()} (line 44)
     * which checks {@code handler instanceof HandlerMethod}
     */
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    /**
     * Resolve arguments, invoke the handler method, and write the result to the response.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.handleInternal()} (line 831)
     * → {@code invokeHandlerMethod()} (line 885)
     * → {@code ServletInvocableHandlerMethod.invokeAndHandle()} (line 114)
     * → {@code InvocableHandlerMethod.invokeForRequest()} (line 171)
     *
     * The real chain creates an InvocableHandlerMethod, configures argument
     * resolvers and return value handlers, then invokes. We do a simplified
     * version that still follows the same resolve → invoke → handle-return flow.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the HandlerMethod to invoke
     */
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // Step 1: Resolve method arguments from the request (ch05).
        // Maps to: InvocableHandlerMethod.getMethodArgumentValues() (line 186)
        Object[] args = resolveArguments(handlerMethod, request);

        // Step 2: Invoke the controller method via reflection.
        // Maps to: InvocableHandlerMethod.doInvoke() (line 243)
        Object result = invokeHandlerMethod(handlerMethod, args);

        // Step 3: Write the result to the response.
        // Maps to: HandlerMethodReturnValueHandler pattern
        // The real framework uses ReturnValueHandlers to decide how to write
        // the response (JSON, view name, redirect, etc.).
        // We write everything as text/plain for now.
        writeResponse(response, result);
    }

    /**
     * Resolve each method parameter into an argument value using the resolver chain.
     *
     * Maps to: {@code InvocableHandlerMethod.getMethodArgumentValues()} (line 186)
     *
     * The real version:
     * <pre>
     *   for each parameter:
     *     if resolvers.supportsParameter(param):
     *       args[i] = resolvers.resolveArgument(param, mavContainer, webRequest, binderFactory)
     *     else:
     *       throw IllegalStateException("no suitable resolver")
     * </pre>
     *
     * We follow the same pattern but with our simplified resolver interface.
     * Parameters without a matching resolver get null — this allows no-arg
     * methods and methods with a mix of resolved and unresolved params to work.
     * Future chapters will tighten this as more resolvers are added.
     */
    private Object[] resolveArguments(HandlerMethod handlerMethod, HttpServletRequest request) throws Exception {
        MethodParameter[] parameters = handlerMethod.getMethodParameters();
        if (parameters.length == 0) {
            return new Object[0];
        }

        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter param = parameters[i];
            if (argumentResolvers.supportsParameter(param)) {
                args[i] = argumentResolvers.resolveArgument(param, request);
            } else {
                throw new IllegalStateException(
                        "No suitable resolver for argument " + i + " of type '"
                                + param.getParameterType().getName()
                                + "' on method " + handlerMethod);
            }
        }
        return args;
    }

    /**
     * Invoke the handler method with the resolved arguments and return the result.
     *
     * Maps to: {@code InvocableHandlerMethod.doInvoke()} (line 243)
     * The real version uses getBridgedMethod() for generic type bridge methods,
     * handles Kotlin suspend functions, and formats detailed error messages.
     */
    private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
        } catch (InvocationTargetException ex) {
            // Unwrap the real exception from the reflection wrapper.
            // Maps to: InvocableHandlerMethod.doInvoke() (line 260-275)
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    /**
     * Write the handler method's return value to the HTTP response.
     *
     * Maps to: {@code HandlerMethodReturnValueHandler.handleReturnValue()}
     * Real version iterates a chain of ReturnValueHandlers:
     * - RequestResponseBodyMethodProcessor for @ResponseBody (writes JSON via HttpMessageConverter)
     * - ViewNameMethodReturnValueHandler for String returns (resolves to a View)
     * - ModelAndViewMethodReturnValueHandler for ModelAndView returns
     *
     * We just write the String result as text/plain. Ch06 will introduce the
     * ReturnValueHandler abstraction and add JSON support.
     */
    private void writeResponse(HttpServletResponse response, Object result) throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        if (result != null) {
            response.getWriter().write(result.toString());
        }
        response.getWriter().flush();
    }

    /**
     * Provides access to the argument resolver composite for testing.
     */
    public HandlerMethodArgumentResolverComposite getArgumentResolvers() {
        return argumentResolvers;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/HandlerMethod.java` [MODIFIED]

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
 *   <li>No @ResponseStatus evaluation</li>
 *   <li>No validation flag support</li>
 * </ul>
 */
public class HandlerMethod {

    private final Object bean;
    private final Method method;
    private final MethodParameter[] parameters;

    public HandlerMethod(Object bean, Method method) {
        if (bean == null) {
            throw new IllegalArgumentException("Bean must not be null");
        }
        if (method == null) {
            throw new IllegalArgumentException("Method must not be null");
        }
        this.bean = bean;
        this.method = method;
        this.parameters = initMethodParameters();
    }

    /**
     * Build the MethodParameter[] array eagerly at construction time.
     *
     * Maps to: {@code HandlerMethod.initMethodParameters()} (line 189)
     * Real version creates SynthesizingMethodParameter instances.
     */
    private MethodParameter[] initMethodParameters() {
        int count = method.getParameterCount();
        MethodParameter[] result = new MethodParameter[count];
        for (int i = 0; i < count; i++) {
            result[i] = new MethodParameter(method, i);
        }
        return result;
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

    /**
     * The method parameters, wrapped as MethodParameter objects.
     *
     * Maps to: {@code HandlerMethod.getMethodParameters()} (line 247)
     * Real version returns MethodParameter[] with generic type info,
     * annotation synthesis, and nesting support.
     */
    public MethodParameter[] getMethodParameters() {
        return parameters;
    }

    @Override
    public String toString() {
        return bean.getClass().getSimpleName() + "#" + method.getName();
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/mapping/MethodParameterTest.java` [NEW]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.RequestParam;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link MethodParameter}.
 */
class MethodParameterTest {

    // ─── Test fixture methods ─────────────────────────────────────

    @SuppressWarnings("unused")
    static class SampleController {

        public String greet(@RequestParam("name") String name) {
            return "Hello, " + name;
        }

        public String add(int a, int b) {
            return String.valueOf(a + b);
        }

        public void noArgs() {
        }
    }

    // ─── Type and name ────────────────────────────────────────────

    @Nested
    @DisplayName("Type and name resolution")
    class TypeAndName {

        @Test
        @DisplayName("should return the declared parameter type")
        void shouldReturnParameterType() throws Exception {
            Method method = SampleController.class.getMethod("greet", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(param.getParameterType()).isEqualTo(String.class);
        }

        @Test
        @DisplayName("should return the parameter name from bytecode")
        void shouldReturnParameterName() throws Exception {
            Method method = SampleController.class.getMethod("greet", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            // Requires -parameters compiler flag (set in build.gradle)
            assertThat(param.getParameterName()).isEqualTo("name");
        }

        @Test
        @DisplayName("should return correct types for multiple parameters")
        void shouldReturnCorrectTypesForMultipleParams() throws Exception {
            Method method = SampleController.class.getMethod("add", int.class, int.class);

            MethodParameter first = new MethodParameter(method, 0);
            MethodParameter second = new MethodParameter(method, 1);

            assertThat(first.getParameterType()).isEqualTo(int.class);
            assertThat(second.getParameterType()).isEqualTo(int.class);
            assertThat(first.getParameterName()).isEqualTo("a");
            assertThat(second.getParameterName()).isEqualTo("b");
        }
    }

    // ─── Annotations ──────────────────────────────────────────────

    @Nested
    @DisplayName("Annotation access")
    class Annotations {

        @Test
        @DisplayName("should detect @RequestParam annotation")
        void shouldDetectRequestParam() throws Exception {
            Method method = SampleController.class.getMethod("greet", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(param.hasParameterAnnotation(RequestParam.class)).isTrue();
        }

        @Test
        @DisplayName("should return @RequestParam annotation instance")
        void shouldReturnRequestParamAnnotation() throws Exception {
            Method method = SampleController.class.getMethod("greet", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            RequestParam annotation = param.getParameterAnnotation(RequestParam.class);

            assertThat(annotation).isNotNull();
            assertThat(annotation.value()).isEqualTo("name");
        }

        @Test
        @DisplayName("should return false when annotation is absent")
        void shouldReturnFalse_WhenAnnotationAbsent() throws Exception {
            Method method = SampleController.class.getMethod("add", int.class, int.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(param.hasParameterAnnotation(RequestParam.class)).isFalse();
        }

        @Test
        @DisplayName("should return null when getting absent annotation")
        void shouldReturnNull_WhenGettingAbsentAnnotation() throws Exception {
            Method method = SampleController.class.getMethod("add", int.class, int.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(param.getParameterAnnotation(RequestParam.class)).isNull();
        }
    }

    // ─── Method reference ─────────────────────────────────────────

    @Nested
    @DisplayName("Method reference")
    class MethodRef {

        @Test
        @DisplayName("should return the declaring method")
        void shouldReturnDeclaringMethod() throws Exception {
            Method method = SampleController.class.getMethod("greet", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(param.getMethod()).isSameAs(method);
        }

        @Test
        @DisplayName("should return the parameter index")
        void shouldReturnParameterIndex() throws Exception {
            Method method = SampleController.class.getMethod("add", int.class, int.class);

            assertThat(new MethodParameter(method, 0).getParameterIndex()).isEqualTo(0);
            assertThat(new MethodParameter(method, 1).getParameterIndex()).isEqualTo(1);
        }
    }

    // ─── toString ─────────────────────────────────────────────────

    @Test
    @DisplayName("should produce readable toString")
    void shouldProduceReadableToString() throws Exception {
        Method method = SampleController.class.getMethod("greet", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(param.toString())
                .contains("SampleController")
                .contains("greet")
                .contains("String")
                .contains("name");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/RequestParamArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

/**
 * Unit tests for {@link RequestParamArgumentResolver}.
 */
class RequestParamArgumentResolverTest {

    // ─── Test fixture ─────────────────────────────────────────────

    @SuppressWarnings("unused")
    static class TestController {

        public String withExplicitName(@RequestParam("q") String query) {
            return query;
        }

        public String withImplicitName(@RequestParam String name) {
            return name;
        }

        public String noAnnotation(String value) {
            return value;
        }
    }

    private RequestParamArgumentResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        resolver = new RequestParamArgumentResolver();
        request = mock(HttpServletRequest.class);
    }

    // ─── supportsParameter() ──────────────────────────────────────

    @Nested
    @DisplayName("supportsParameter()")
    class SupportsParameter {

        @Test
        @DisplayName("should support parameter with @RequestParam")
        void shouldSupport_WhenAnnotatedWithRequestParam() throws Exception {
            Method method = TestController.class.getMethod("withExplicitName", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(resolver.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should support @RequestParam without explicit value")
        void shouldSupport_WhenRequestParamHasNoValue() throws Exception {
            Method method = TestController.class.getMethod("withImplicitName", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(resolver.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should not support parameter without @RequestParam")
        void shouldNotSupport_WhenNoAnnotation() throws Exception {
            Method method = TestController.class.getMethod("noAnnotation", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(resolver.supportsParameter(param)).isFalse();
        }
    }

    // ─── resolveArgument() ────────────────────────────────────────

    @Nested
    @DisplayName("resolveArgument()")
    class ResolveArgument {

        @Test
        @DisplayName("should resolve parameter using explicit annotation name")
        void shouldResolveByExplicitName() throws Exception {
            Method method = TestController.class.getMethod("withExplicitName", String.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getParameter("q")).thenReturn("spring mvc");

            Object result = resolver.resolveArgument(param, request);

            assertThat(result).isEqualTo("spring mvc");
        }

        @Test
        @DisplayName("should resolve parameter using implicit method parameter name")
        void shouldResolveByImplicitName() throws Exception {
            Method method = TestController.class.getMethod("withImplicitName", String.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getParameter("name")).thenReturn("Alice");

            Object result = resolver.resolveArgument(param, request);

            assertThat(result).isEqualTo("Alice");
        }

        @Test
        @DisplayName("should throw when required parameter is missing")
        void shouldThrow_WhenParameterMissing() throws Exception {
            Method method = TestController.class.getMethod("withExplicitName", String.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getParameter("q")).thenReturn(null);

            assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("Missing required request parameter")
                    .hasMessageContaining("q");
        }

        @Test
        @DisplayName("should resolve empty string as valid value")
        void shouldResolveEmptyString() throws Exception {
            Method method = TestController.class.getMethod("withExplicitName", String.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getParameter("q")).thenReturn("");

            Object result = resolver.resolveArgument(param, request);

            assertThat(result).isEqualTo("");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/HandlerMethodArgumentResolverCompositeTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

/**
 * Unit tests for {@link HandlerMethodArgumentResolverComposite}.
 */
class HandlerMethodArgumentResolverCompositeTest {

    // ─── Test fixtures ────────────────────────────────────────────

    @SuppressWarnings("unused")
    static class TestController {

        public String search(@RequestParam("q") String query) {
            return query;
        }

        public String noAnnotation(String value) {
            return value;
        }
    }

    private HandlerMethodArgumentResolverComposite composite;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        composite = new HandlerMethodArgumentResolverComposite();
        composite.addResolver(new RequestParamArgumentResolver());
        request = mock(HttpServletRequest.class);
    }

    // ─── supportsParameter() ──────────────────────────────────────

    @Nested
    @DisplayName("supportsParameter()")
    class SupportsParameter {

        @Test
        @DisplayName("should support parameter when a resolver handles it")
        void shouldSupport_WhenResolverExists() throws Exception {
            Method method = TestController.class.getMethod("search", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(composite.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should not support parameter when no resolver handles it")
        void shouldNotSupport_WhenNoResolverExists() throws Exception {
            Method method = TestController.class.getMethod("noAnnotation", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(composite.supportsParameter(param)).isFalse();
        }
    }

    // ─── resolveArgument() ────────────────────────────────────────

    @Nested
    @DisplayName("resolveArgument()")
    class ResolveArgument {

        @Test
        @DisplayName("should delegate to the matching resolver")
        void shouldDelegateToMatchingResolver() throws Exception {
            Method method = TestController.class.getMethod("search", String.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getParameter("q")).thenReturn("hello");

            Object result = composite.resolveArgument(param, request);

            assertThat(result).isEqualTo("hello");
        }

        @Test
        @DisplayName("should throw when no resolver supports the parameter")
        void shouldThrow_WhenNoResolverSupports() throws Exception {
            Method method = TestController.class.getMethod("noAnnotation", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThatThrownBy(() -> composite.resolveArgument(param, request))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Unsupported parameter type");
        }
    }

    // ─── Resolver management ──────────────────────────────────────

    @Nested
    @DisplayName("Resolver management")
    class ResolverManagement {

        @Test
        @DisplayName("should return unmodifiable list of resolvers")
        void shouldReturnUnmodifiableList() {
            assertThat(composite.getResolvers()).hasSize(1);
            assertThatThrownBy(() -> composite.getResolvers().add(new RequestParamArgumentResolver()))
                    .isInstanceOf(UnsupportedOperationException.class);
        }

        @Test
        @DisplayName("should support adding multiple resolvers")
        void shouldSupportAddingMultipleResolvers() {
            HandlerMethodArgumentResolverComposite fresh = new HandlerMethodArgumentResolverComposite();
            fresh.addResolver(new RequestParamArgumentResolver());
            fresh.addResolver(new RequestParamArgumentResolver());

            assertThat(fresh.getResolvers()).hasSize(2);
        }
    }

    // ─── Caching ──────────────────────────────────────────────────

    @Test
    @DisplayName("should cache resolver for repeated parameter lookups")
    void shouldCacheResolverForRepeatedLookups() throws Exception {
        Method method = TestController.class.getMethod("search", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        when(request.getParameter("q")).thenReturn("first").thenReturn("second");

        // First call triggers resolver scan
        Object first = composite.resolveArgument(param, request);
        // Second call should use the cache
        Object second = composite.resolveArgument(param, request);

        assertThat(first).isEqualTo("first");
        assertThat(second).isEqualTo("second");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/SimpleHandlerAdapterTest.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link SimpleHandlerAdapter}.
 */
class SimpleHandlerAdapterTest {

    // ─── Test controllers ────────────────────────────────────────

    static class StringController {
        public String hello() {
            return "hello world";
        }

        public String empty() {
            return "";
        }

        public Object nullResult() {
            return null;
        }

        public String throwing() {
            throw new IllegalArgumentException("test error");
        }
    }

    static class ParamController {
        public String greet(@RequestParam("name") String name) {
            return "Hello, " + name + "!";
        }

        public String search(@RequestParam("q") String query, @RequestParam("page") String page) {
            return "query=" + query + "&page=" + page;
        }
    }

    private SimpleHandlerAdapter adapter;
    private HttpServletRequest request;
    private HttpServletResponse response;
    private StringWriter responseBody;

    @BeforeEach
    void setUp() throws Exception {
        adapter = new SimpleHandlerAdapter();
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    // ─── supports() ─────────────────────────────────────────────

    @Nested
    @DisplayName("supports()")
    class Supports {

        @Test
        @DisplayName("should support HandlerMethod instances")
        void shouldSupportHandlerMethod() throws Exception {
            Object controller = new StringController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            assertThat(adapter.supports(handlerMethod)).isTrue();
        }

        @Test
        @DisplayName("should not support arbitrary objects")
        void shouldNotSupportArbitraryObjects() {
            assertThat(adapter.supports("not a handler")).isFalse();
            assertThat(adapter.supports(42)).isFalse();
            assertThat(adapter.supports(new Object())).isFalse();
        }
    }

    // ─── handle() — no-arg methods ──────────────────────────────

    @Nested
    @DisplayName("handle() — no-arg methods")
    class HandleNoArgs {

        @Test
        @DisplayName("should invoke handler method and write result to response")
        void shouldInvokeHandlerAndWriteResult() throws Exception {
            StringController controller = new StringController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            adapter.handle(request, response, handlerMethod);

            assertThat(responseBody.toString()).isEqualTo("hello world");
        }

        @Test
        @DisplayName("should set content type to text/plain")
        void shouldSetContentType() throws Exception {
            StringController controller = new StringController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            adapter.handle(request, response, handlerMethod);

            verify(response).setContentType("text/plain");
        }

        @Test
        @DisplayName("should set character encoding to UTF-8")
        void shouldSetCharacterEncoding() throws Exception {
            StringController controller = new StringController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            adapter.handle(request, response, handlerMethod);

            verify(response).setCharacterEncoding("UTF-8");
        }

        @Test
        @DisplayName("should handle empty string result")
        void shouldHandleEmptyStringResult() throws Exception {
            StringController controller = new StringController();
            Method method = controller.getClass().getMethod("empty");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            adapter.handle(request, response, handlerMethod);

            assertThat(responseBody.toString()).isEmpty();
        }

        @Test
        @DisplayName("should handle null result without writing body")
        void shouldHandleNullResult() throws Exception {
            StringController controller = new StringController();
            Method method = controller.getClass().getMethod("nullResult");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            adapter.handle(request, response, handlerMethod);

            assertThat(responseBody.toString()).isEmpty();
        }

        @Test
        @DisplayName("should unwrap InvocationTargetException and throw cause")
        void shouldUnwrapInvocationTargetException() throws Exception {
            StringController controller = new StringController();
            Method method = controller.getClass().getMethod("throwing");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            assertThatThrownBy(() -> adapter.handle(request, response, handlerMethod))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessage("test error");
        }
    }

    // ─── handle() — @RequestParam methods ───────────────────────

    @Nested
    @DisplayName("handle() — @RequestParam methods")
    class HandleWithRequestParam {

        @Test
        @DisplayName("should resolve single @RequestParam and pass to handler")
        void shouldResolveSingleRequestParam() throws Exception {
            ParamController controller = new ParamController();
            Method method = controller.getClass().getMethod("greet", String.class);
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            when(request.getParameter("name")).thenReturn("Alice");

            adapter.handle(request, response, handlerMethod);

            assertThat(responseBody.toString()).isEqualTo("Hello, Alice!");
        }

        @Test
        @DisplayName("should resolve multiple @RequestParam parameters")
        void shouldResolveMultipleRequestParams() throws Exception {
            ParamController controller = new ParamController();
            Method method = controller.getClass().getMethod("search", String.class, String.class);
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            when(request.getParameter("q")).thenReturn("spring");
            when(request.getParameter("page")).thenReturn("2");

            adapter.handle(request, response, handlerMethod);

            assertThat(responseBody.toString()).isEqualTo("query=spring&page=2");
        }

        @Test
        @DisplayName("should throw when required @RequestParam is missing")
        void shouldThrow_WhenRequiredParamMissing() throws Exception {
            ParamController controller = new ParamController();
            Method method = controller.getClass().getMethod("greet", String.class);
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            when(request.getParameter("name")).thenReturn(null);

            assertThatThrownBy(() -> adapter.handle(request, response, handlerMethod))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("Missing required request parameter")
                    .hasMessageContaining("name");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/RequestParamIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.annotation.RequestParam;
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
 * Integration tests for the @RequestParam argument resolution feature (ch05).
 *
 * Verifies that query parameters are extracted from real HTTP requests,
 * resolved through the argument resolver chain, and passed as method
 * arguments end-to-end through the full dispatch pipeline.
 */
class RequestParamIntegrationTest {

    // ─── Test controllers ────────────────────────────────────────

    @Controller
    static class GreetingController {

        @RequestMapping(path = "/greet", method = "GET")
        public String greet(@RequestParam("name") String name) {
            return "Hello, " + name + "!";
        }

        @RequestMapping(path = "/search", method = "GET")
        public String search(@RequestParam("q") String query, @RequestParam("limit") String limit) {
            return "Searching for '" + query + "' (limit=" + limit + ")";
        }
    }

    @Controller
    static class ImplicitNameController {

        @RequestMapping(path = "/echo", method = "GET")
        public String echo(@RequestParam String message) {
            return "echo: " + message;
        }
    }

    @Controller
    static class NoArgController {

        @RequestMapping(path = "/ping", method = "GET")
        public String ping() {
            return "pong";
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new GreetingController());
        container.registerBean(new ImplicitNameController());
        container.registerBean(new NoArgController());

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

    // ─── Single @RequestParam ────────────────────────────────────

    @Test
    @DisplayName("should resolve @RequestParam from query string")
    void shouldResolveRequestParam() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/greet?name=Alice")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello, Alice!");
    }

    @Test
    @DisplayName("should resolve @RequestParam with special characters")
    void shouldResolveWithSpecialCharacters() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/greet?name=John%20Doe")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello, John Doe!");
    }

    // ─── Multiple @RequestParam ──────────────────────────────────

    @Test
    @DisplayName("should resolve multiple @RequestParam parameters")
    void shouldResolveMultipleRequestParams() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/search?q=spring&limit=10")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Searching for 'spring' (limit=10)");
    }

    // ─── Implicit parameter name ─────────────────────────────────

    @Test
    @DisplayName("should resolve @RequestParam using method parameter name as fallback")
    void shouldResolveByImplicitName() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/echo?message=hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("echo: hello");
    }

    // ─── No-arg methods still work ───────────────────────────────

    @Test
    @DisplayName("should still handle no-arg handler methods after adding resolver support")
    void shouldHandleNoArgMethods() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/ping")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("pong");
    }

    // ─── Missing parameter ───────────────────────────────────────

    @Test
    @DisplayName("should return 500 when required @RequestParam is missing")
    void shouldReturn500_WhenRequiredParamMissing() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/greet")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(500);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **HandlerMethodArgumentResolver** | Strategy interface — each implementation knows how to extract one type of argument from the request |
| **Composite pattern** | `HandlerMethodArgumentResolverComposite` delegates to N resolvers behind a single interface, with a `ConcurrentHashMap` cache |
| **MethodParameter** | Wrapper around `java.lang.reflect.Parameter` providing type, name, and annotation access for resolvers |
| **@RequestParam** | Annotation that marks a method parameter for extraction from the HTTP query string |
| **Integration point** | The adapter's `handle()` method — argument resolution sits between handler lookup and method invocation |
| **`-parameters` flag** | JDK compiler option that preserves method parameter names in bytecode, enabling implicit name resolution |

**Next: Chapter 6 — @ResponseBody & JSON Conversion** — Currently every handler returns text/plain. Chapter 6 introduces the `HandlerMethodReturnValueHandler` pattern and `HttpMessageConverter` interface, so controllers can return Java objects that are automatically serialized to JSON via Jackson.
