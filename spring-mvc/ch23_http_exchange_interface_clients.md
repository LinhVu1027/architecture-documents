# Chapter 23: @HttpExchange / HTTP Interface Clients

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Our framework handles **incoming** HTTP requests — controllers receive, process, and respond. But there's no client-side equivalent: calling external HTTP APIs requires manual `HttpClient` construction, URL building, JSON serialization, and response parsing for every endpoint. | Consuming REST APIs requires verbose, repetitive boilerplate code with no type safety — every HTTP call is a hand-crafted string-and-bytes exercise. | Build a declarative HTTP client system where annotated Java interfaces become working HTTP clients via JDK dynamic proxies — the client-side mirror of `@RequestMapping`. |

---

## 23.1 The Integration Point

This feature is unique among all 23 chapters: it has **no integration point into the existing dispatch pipeline**. Every previous feature (chapters 2–22) plugged into the `DispatcherServlet` request processing flow. This chapter builds a completely independent subsystem — a **client-side HTTP framework** that reuses annotations and patterns from the server side.

The "integration point" is the `HttpServiceProxyFactory` itself — the factory class that creates JDK dynamic proxies. It's the equivalent of `DispatcherServlet` for the client side: the place where all the pieces come together.

```java
// The heart of the client framework: create a proxy from an interface
UserService client = HttpServiceProxyFactory.builder()
        .exchangeAdapter(new JdkClientAdapter("http://localhost:8080", objectMapper))
        .build()
        .createClient(UserService.class);

// Method calls become HTTP requests
User user = client.getUser("42");  // → GET http://localhost:8080/api/users/42
```

Two key decisions here:

1. **JDK dynamic proxies instead of code generation.** At runtime, `Proxy.newProxyInstance()` creates an object that implements the interface but routes every method call through our `InvocationHandler`. No compile-time tooling needed — just annotate and go.

2. **Reuse existing annotations.** The `@PathVariable`, `@RequestParam`, and `@RequestBody` annotations from the server side work here too. This mirrors the real framework where `org.springframework.web.bind.annotation` annotations are shared between server and client.

This connects **user-defined interfaces ↔ HTTP client execution**. To make it work, we need to build:
- **Annotations** — `@HttpExchange`, `@GetExchange`, `@PostExchange`, `@PutExchange`, `@DeleteExchange`
- **HttpRequestValues** — immutable value object carrying all request data
- **HttpExchangeAdapter** — strategy interface decoupling proxy from HTTP library
- **HttpServiceArgumentResolver** — resolves method arguments into request parts
- **Three resolvers** — for `@PathVariable`, `@RequestParam`, `@RequestBody`
- **HttpServiceMethod** — per-method invocation brain (annotations → builder → execute)
- **HttpServiceProxyFactory** — factory that creates the proxy
- **JdkClientAdapter** — concrete adapter backed by `java.net.http.HttpClient`

## 23.2 The Annotations

The annotation hierarchy mirrors `@RequestMapping` / `@GetMapping` from chapter 10 — shortcut annotations are meta-annotated with `@HttpExchange`, pre-setting the HTTP method:

**New file:** `src/main/java/com/simplespringmvc/exchange/annotation/HttpExchange.java`

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface HttpExchange {
    String value() default "";       // URL path
    String method() default "";      // HTTP method (GET, POST, etc.)
    String contentType() default ""; // Content-Type header
    String[] accept() default {};    // Accept header values
}
```

**New file:** `src/main/java/com/simplespringmvc/exchange/annotation/GetExchange.java`

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@HttpExchange(method = "GET")           // ← meta-annotation fixes HTTP method
public @interface GetExchange {
    String value() default "";
    String[] accept() default {};
}
```

`@PostExchange`, `@PutExchange`, and `@DeleteExchange` follow the same pattern with `@HttpExchange(method = "POST")`, etc. `@PostExchange`, `@PutExchange`, and `@DeleteExchange` also expose a `contentType()` attribute since they typically carry request bodies.

## 23.3 The Value Object and Strategy Interfaces

**New file:** `src/main/java/com/simplespringmvc/exchange/HttpRequestValues.java`

The "currency" between the proxy layer and the HTTP client — an immutable snapshot of everything needed for one HTTP call:

```java
package com.simplespringmvc.exchange;

import java.net.URI;
import java.util.*;

public class HttpRequestValues {
    private final String httpMethod;
    private final String uriTemplate;
    private final Map<String, String> uriVariables;
    private final Map<String, String> headers;
    private final Map<String, String> queryParams;
    private final Object body;

    // All fields set via Builder, returned as unmodifiable maps
    // getExpandedUri() resolves {variables} and appends ?queryParams

    public static class Builder {
        // Mutable accumulator: setHttpMethod(), setUriTemplate(),
        // setUriVariable(), addHeader(), addQueryParam(), setBody()
        // build() → immutable HttpRequestValues
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/exchange/HttpExchangeAdapter.java`

Strategy interface that decouples the proxy from any HTTP library:

```java
package com.simplespringmvc.exchange;

public interface HttpExchangeAdapter {
    void exchange(HttpRequestValues requestValues);                    // void return
    <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType); // typed return
}
```

**New file:** `src/main/java/com/simplespringmvc/exchange/HttpServiceArgumentResolver.java`

Strategy for resolving one method argument into part of the request:

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.mapping.MethodParameter;

public interface HttpServiceArgumentResolver {
    boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues);
}
```

## 23.4 The Three Argument Resolvers

Each resolver checks for its annotation and writes to the builder:

**New file:** `src/main/java/com/simplespringmvc/exchange/ExchangePathVariableArgumentResolver.java`

```java
public class ExchangePathVariableArgumentResolver implements HttpServiceArgumentResolver {
    @Override
    public boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues) {
        PathVariable annotation = parameter.getParameterAnnotation(PathVariable.class);
        if (annotation == null) return false;

        String name = annotation.value().isEmpty() ? parameter.getParameterName() : annotation.value();
        requestValues.setUriVariable(name, String.valueOf(argument));
        return true;
    }
}
```

`ExchangeRequestParamArgumentResolver` → `requestValues.addQueryParam(name, value)`
`ExchangeRequestBodyArgumentResolver` → `requestValues.setBody(argument)`

Same pattern, different builder method. The server-side resolvers read FROM the request; the client-side resolvers write INTO the request builder.

## 23.5 The Per-Method Brain: HttpServiceMethod

**New file:** `src/main/java/com/simplespringmvc/exchange/HttpServiceMethod.java`

One instance per interface method. Extracts annotation metadata at construction time, then runs the three-phase pipeline on each invocation:

```java
public Object invoke(Object[] arguments) {
    // Phase 1: Initialize builder with annotation-derived values (HTTP method, URL, headers)
    HttpRequestValues.Builder builder = HttpRequestValues.builder()
            .setHttpMethod(httpMethod)
            .setUriTemplate(urlTemplate);

    // Phase 2: Resolve each method argument via the resolver chain
    for (int i = 0; i < arguments.length; i++) {
        MethodParameter param = new MethodParameter(method, i);
        for (HttpServiceArgumentResolver resolver : argumentResolvers) {
            if (resolver.resolve(arguments[i], param, builder)) break;
        }
    }

    // Phase 3: Build and execute
    HttpRequestValues requestValues = builder.build();
    if (method.getReturnType() == void.class) {
        adapter.exchange(requestValues);
        return null;
    } else {
        return adapter.exchangeForBody(requestValues, method.getReturnType());
    }
}
```

The constructor also handles URL merging: type-level `@HttpExchange("/api/users")` + method-level `@GetExchange("/{id}")` → `/api/users/{id}`.

## 23.6 The Factory: HttpServiceProxyFactory

**New file:** `src/main/java/com/simplespringmvc/exchange/HttpServiceProxyFactory.java`

```java
public <S> S createClient(Class<S> serviceType) {
    // Pre-compute HttpServiceMethod for each @HttpExchange method
    Map<Method, HttpServiceMethod> methodMap = new HashMap<>();
    for (Method method : serviceType.getMethods()) {
        if (MergedAnnotationUtils.hasAnnotation(method, HttpExchange.class)) {
            methodMap.put(method, new HttpServiceMethod(method, serviceType, argumentResolvers, adapter));
        }
    }

    // Create JDK dynamic proxy
    return (S) Proxy.newProxyInstance(
            serviceType.getClassLoader(),
            new Class<?>[]{serviceType},
            (proxy, method, args) -> {
                if (method.getDeclaringClass() == Object.class) { /* handle toString, equals, hashCode */ }
                return methodMap.get(method).invoke(args);
            }
    );
}
```

## 23.7 The Concrete Adapter: JdkClientAdapter

**New file:** `src/main/java/com/simplespringmvc/exchange/JdkClientAdapter.java`

Translates `HttpRequestValues` → JDK `HttpRequest`, sends it via `HttpClient`, and deserializes the JSON response with Jackson:

```java
public class JdkClientAdapter implements HttpExchangeAdapter {
    private final String baseUrl;
    private final ObjectMapper objectMapper;
    private final HttpClient httpClient;

    @Override
    public <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType) {
        HttpRequest request = buildRequest(requestValues);  // URI + method + headers + body
        HttpResponse<String> response = httpClient.send(request, BodyHandlers.ofString());
        return objectMapper.readValue(response.body(), bodyType);
    }

    private HttpRequest buildRequest(HttpRequestValues requestValues) {
        // 1. Resolve full URI (base URL + template expansion + query params)
        // 2. Set HTTP method with JSON body (if present)
        // 3. Copy headers
    }
}
```

## 23.8 Try It Yourself

<details>
<summary>Challenge 1: Add a @PatchExchange shortcut annotation</summary>

Following the pattern of the other shortcut annotations, create `@PatchExchange`:

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@HttpExchange(method = "PATCH")
public @interface PatchExchange {
    String value() default "";
    String contentType() default "";
    String[] accept() default {};
}
```

The meta-annotation pattern makes adding new HTTP methods trivial — you don't need to touch any framework code, just create a new annotation.

</details>

<details>
<summary>Challenge 2: Add a @RequestHeader argument resolver for the exchange client</summary>

Create an `ExchangeRequestHeaderArgumentResolver` that handles a hypothetical `@RequestHeader` annotation:

```java
public class ExchangeRequestHeaderArgumentResolver implements HttpServiceArgumentResolver {
    @Override
    public boolean resolve(Object argument, MethodParameter parameter,
                           HttpRequestValues.Builder requestValues) {
        // Check for @RequestHeader annotation
        // Extract header name from annotation value or parameter name
        // Call requestValues.addHeader(name, String.valueOf(argument))
        // Return true if handled
        return false; // placeholder
    }
}
```

Then register it in `HttpServiceProxyFactory.initArgumentResolvers()`.

</details>

## 23.9 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/exchange/HttpRequestValuesTest.java`

```java
class HttpRequestValuesTest {

    @Test
    void shouldBuildRequestValuesWithAllFields() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setHttpMethod("POST")
                .setUriTemplate("/users/{id}")
                .setUriVariable("id", "42")
                .addHeader("Content-Type", "application/json")
                .addQueryParam("verbose", "true")
                .setBody("test body")
                .build();

        assertThat(values.getHttpMethod()).isEqualTo("POST");
        assertThat(values.getUriVariables()).containsEntry("id", "42");
        assertThat(values.getHeaders()).containsEntry("Content-Type", "application/json");
        assertThat(values.getQueryParams()).containsEntry("verbose", "true");
        assertThat(values.getBody()).isEqualTo("test body");
    }

    @Test
    void shouldExpandUriVariables() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setUriTemplate("http://localhost/users/{id}/posts/{postId}")
                .setUriVariable("id", "42")
                .setUriVariable("postId", "7")
                .build();

        URI expanded = values.getExpandedUri();
        assertThat(expanded.toString()).isEqualTo("http://localhost/users/42/posts/7");
    }

    @Test
    void shouldAppendQueryParams_WhenNoExistingParams() { /* ... */ }
    @Test
    void shouldReturnUnmodifiableCollections() { /* ... */ }
    @Test
    void shouldBuildWithDefaults_WhenNoFieldsSet() { /* ... */ }
}
```

**New file:** `src/test/java/com/simplespringmvc/exchange/ExchangeArgumentResolversTest.java`

```java
class ExchangeArgumentResolversTest {
    @Test
    void shouldResolvePathVariable_WhenAnnotationHasExplicitName() { /* ... */ }
    @Test
    void shouldResolvePathVariable_WhenAnnotationUsesParameterName() { /* ... */ }
    @Test
    void shouldNotResolve_WhenNoPathVariableAnnotation() { /* ... */ }
    @Test
    void shouldResolveRequestParam_WhenAnnotationHasExplicitName() { /* ... */ }
    @Test
    void shouldResolveRequestBody_WhenAnnotated() { /* ... */ }
    // ... 8 tests total
}
```

**New file:** `src/test/java/com/simplespringmvc/exchange/HttpServiceMethodTest.java`

```java
class HttpServiceMethodTest {
    @Test
    void shouldMergeTypeAndMethodUrl_WhenGetExchangeWithPathVariable() { /* ... */ }
    @Test
    void shouldSetContentTypeHeader_WhenPostExchangeWithContentType() { /* ... */ }
    @Test
    void shouldResolveMultipleQueryParams() { /* ... */ }
    @Test
    void shouldUseDeleteMethod_WhenDeleteExchange() { /* ... */ }
    @Test
    void shouldWorkWithoutBaseUrl() { /* ... */ }
    @Test
    void shouldInheritAcceptFromTypeLevel() { /* ... */ }
}
```

**New file:** `src/test/java/com/simplespringmvc/exchange/HttpServiceProxyFactoryTest.java`

```java
class HttpServiceProxyFactoryTest {
    @Test
    void shouldCreateProxy_WhenValidInterface() { /* ... */ }
    @Test
    void shouldRouteGetRequest_WhenProxyMethodCalled() { /* ... */ }
    @Test
    void shouldRoutePostRequest_WhenCreateUserCalled() { /* ... */ }
    @Test
    void shouldRouteDeleteRequest_WhenDeleteUserCalled() { /* ... */ }
    @Test
    void shouldThrow_WhenAdapterNotProvided() { /* ... */ }
    @Test
    void shouldHandleObjectMethods_WhenCalledOnProxy() { /* ... */ }
    // ... 8 tests total
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/HttpExchangeIntegrationTest.java`

```java
class HttpExchangeIntegrationTest {

    public record User(String id, String name, String email) {}

    @HttpExchange(value = "/api/v1/users", accept = "application/json")
    interface UserApi {
        @GetExchange("/{id}")
        User getUser(@PathVariable("id") String id);

        @GetExchange
        String listUsers(@RequestParam("page") int page, @RequestParam("size") int size);

        @PostExchange(contentType = "application/json")
        void createUser(@RequestBody User user);

        @PutExchange("/{id}")
        void updateUser(@PathVariable("id") String id, @RequestBody User user);

        @DeleteExchange("/{id}")
        void deleteUser(@PathVariable("id") String id);
    }

    @Test
    void shouldBuildCompleteGetRequest_WhenProxyMethodCalled() { /* ... */ }
    @Test
    void shouldBuildGetWithQueryParams_WhenListUsersCalled() { /* ... */ }
    @Test
    void shouldBuildPostWithBody_WhenCreateUserCalled() { /* ... */ }
    @Test
    void shouldBuildPutWithPathVarAndBody_WhenUpdateUserCalled() { /* ... */ }
    @Test
    void shouldBuildDeleteRequest_WhenDeleteUserCalled() { /* ... */ }
    @Test
    void shouldSupportMultipleProxyCreation_WhenSameFactoryUsed() { /* ... */ }
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all 22 prior features' tests)

---

## 23.10 Why This Works

> ★ **Insight** -------------------------------------------
> **JDK Dynamic Proxy: Interface → Implementation at Runtime**
>
> `Proxy.newProxyInstance()` creates an object that implements any interface at runtime, routing every method call through an `InvocationHandler`. This is the same mechanism behind Spring's `@Transactional` proxies, JPA repositories, and Feign clients. The trade-off: it only works with interfaces (not concrete classes), and adds one layer of indirection per call. For HTTP clients where the network latency dwarfs proxy overhead, this is a negligible cost.
>
> The real framework uses Spring's `ProxyFactory` (which delegates to JDK Proxy or CGLIB), gaining AOP interceptor support. Our direct JDK Proxy demonstrates the core concept without the AOP layer.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Annotation Reuse: Server ↔ Client Symmetry**
>
> The `@PathVariable`, `@RequestParam`, and `@RequestBody` annotations are shared between server-side controllers and client-side interfaces. On the server, `@PathVariable` means "extract this from the incoming URL"; on the client, it means "insert this into the outgoing URL." Same annotation, mirror-image semantics. This is deliberate in the real framework — `org.springframework.web.bind.annotation` lives in `spring-web`, shared by both `spring-webmvc` (server) and `spring-web/service/invoker` (client). The benefit: developers learn one vocabulary that works on both sides of the HTTP boundary.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The Adapter Pattern: Decoupling Proxy from Transport**
>
> `HttpExchangeAdapter` is a textbook Strategy pattern. The proxy machinery (annotations, resolvers, method invocation) knows nothing about which HTTP library executes the request. You can swap `JdkClientAdapter` for `RestClientAdapter` or `WebClientAdapter` without changing a single annotation or resolver. This is why the real framework ships three adapters (RestClient, WebClient, RestTemplate) — the proxy layer is transport-agnostic.
> -----------------------------------------------------------

## 23.11 What We Enhanced

| Aspect | Before (ch22) | Current (ch23) | Real Framework |
|--------|---------------|----------------|----------------|
| HTTP direction | Server-only — receives and processes incoming requests | Both directions — server receives, client sends via annotated interfaces | Same dual direction: `@RequestMapping` for servers, `@HttpExchange` for clients |
| Calling external APIs | No framework support — requires manual HttpClient/URL/JSON code | Declarative interfaces with `@GetExchange`, `@PostExchange`, etc. generate proxy implementations | `HttpServiceProxyFactory` with `RestClientAdapter`, `WebClientAdapter`, reactive support |
| Annotation reuse | `@PathVariable`, `@RequestParam`, `@RequestBody` used only server-side | Same annotations work on both server controllers and client interfaces | Same — `web.bind.annotation` package shared between server and client |

## 23.12 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `HttpServiceProxyFactory.createClient()` | `HttpServiceProxyFactory.createClient()` | `HttpServiceProxyFactory.java:90` | Real uses Spring's `ProxyFactory` (AOP) with `MethodInterceptor`; we use raw JDK `Proxy` |
| `HttpServiceProxyFactory.initArgumentResolvers()` | `Builder.initArgumentResolvers()` | `HttpServiceProxyFactory.java:252` | Real registers 7+ resolvers including `@RequestHeader`, `@CookieValue`, `@RequestPart`, `@RequestAttribute`, plus type-based resolvers for `URI`, `HttpMethod` |
| `HttpServiceMethod.invoke()` | `HttpServiceMethod.invoke()` | `HttpServiceMethod.java:132` | Real has `ResponseFunction` strategy (void/headers/body/entity/reactive) and `HttpRequestValuesInitializer` record; we inline both |
| `HttpRequestValues.Builder` | `HttpRequestValues.Builder` | `HttpRequestValues.java:529` | Real builder (~370 lines) handles form encoding, multipart, cookies, UriBuilderFactory; ours is ~60 lines |
| `HttpExchangeAdapter` | `HttpExchangeAdapter` | `HttpExchangeAdapter.java:45` | Real has 5 exchange methods + reactive sub-interface; we have 2 |
| `JdkClientAdapter` | `RestClientAdapter` | `RestClientAdapter.java:95` | Real bridges to Spring's RestClient; we bridge to JDK `HttpClient` |
| `ExchangePathVariableArgumentResolver` | `PathVariableArgumentResolver` | `PathVariableArgumentResolver.java:44` | Real extends `AbstractNamedValueArgumentResolver` with `ConversionService` |
| `ExchangeRequestParamArgumentResolver` | `RequestParamArgumentResolver` | `RequestParamArgumentResolver.java:57` | Real handles multi-value params, form data encoding, `favorSingleValue` flag |
| `ExchangeRequestBodyArgumentResolver` | `RequestBodyArgumentResolver` | `RequestBodyArgumentResolver.java:40` | Real handles streaming bodies, Optional, reactive types, ParameterizedTypeReference |
| `@HttpExchange` | `@HttpExchange` | `HttpExchange.java:135` | Real has `headers` (key=value pairs), `version` (API versioning), `@AliasFor` support |

## 23.13 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/exchange/annotation/HttpExchange.java` [NEW]

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation to declare an HTTP service endpoint on an interface method, or to
 * set common properties (base URL, content type) at the interface level.
 *
 * Maps to: {@code org.springframework.web.service.annotation.HttpExchange}
 *
 * Used on:
 * <ul>
 *   <li><b>Type level</b> — sets a base URL, shared content type, or accept headers
 *       that apply to all methods in the interface</li>
 *   <li><b>Method level</b> — declares a specific HTTP endpoint (though the shortcut
 *       annotations like {@code @GetExchange} are preferred)</li>
 * </ul>
 *
 * Shortcut annotations ({@code @GetExchange}, {@code @PostExchange}, etc.) are
 * meta-annotated with {@code @HttpExchange(method = "GET")} etc., pre-setting the
 * HTTP method so the user only needs to specify the URL.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No {@code headers} attribute (real version supports {@code "key=value"} pairs)</li>
 *   <li>No {@code version} attribute (API versioning, added in Spring 7.0)</li>
 *   <li>No {@code @AliasFor} support — we read attributes via reflection</li>
 * </ul>
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface HttpExchange {

    /**
     * The URL path (or full URL) for this exchange.
     * At type level, serves as a base URL prefix.
     * At method level, appended to the type-level URL.
     */
    String value() default "";

    /**
     * The HTTP method: "GET", "POST", "PUT", "DELETE", etc.
     * Left empty at the type level. Shortcut annotations pre-fill this.
     */
    String method() default "";

    /**
     * The Content-Type for the request body.
     * Method-level overrides type-level.
     */
    String contentType() default "";

    /**
     * The acceptable media types for the response (Accept header).
     * Method-level overrides type-level.
     */
    String[] accept() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/annotation/GetExchange.java` [NEW]

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @HttpExchange(method = "GET")}.
 *
 * Maps to: {@code org.springframework.web.service.annotation.GetExchange}
 *
 * The real annotation uses {@code @AliasFor} to forward its {@code value()} and
 * {@code accept()} attributes to {@code @HttpExchange}. We achieve the same effect
 * by reading attributes from the composed annotation via reflection in
 * {@link com.simplespringmvc.exchange.HttpServiceMethod}.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@HttpExchange(method = "GET")
public @interface GetExchange {

    /**
     * The URL path for this GET endpoint.
     */
    String value() default "";

    /**
     * Acceptable response media types (Accept header).
     */
    String[] accept() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/annotation/PostExchange.java` [NEW]

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @HttpExchange(method = "POST")}.
 *
 * Maps to: {@code org.springframework.web.service.annotation.PostExchange}
 *
 * Includes {@code contentType()} because POST typically carries a request body.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@HttpExchange(method = "POST")
public @interface PostExchange {

    String value() default "";

    String contentType() default "";

    String[] accept() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/annotation/PutExchange.java` [NEW]

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @HttpExchange(method = "PUT")}.
 *
 * Maps to: {@code org.springframework.web.service.annotation.PutExchange}
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@HttpExchange(method = "PUT")
public @interface PutExchange {

    String value() default "";

    String contentType() default "";

    String[] accept() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/annotation/DeleteExchange.java` [NEW]

```java
package com.simplespringmvc.exchange.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @HttpExchange(method = "DELETE")}.
 *
 * Maps to: {@code org.springframework.web.service.annotation.DeleteExchange}
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@HttpExchange(method = "DELETE")
public @interface DeleteExchange {

    String value() default "";

    String contentType() default "";

    String[] accept() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/HttpRequestValues.java` [NEW]

```java
package com.simplespringmvc.exchange;

import java.net.URI;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * An immutable value object that carries all the data needed to make an HTTP request:
 * method, URI, headers, query parameters, URI variables, and body.
 *
 * Maps to: {@code org.springframework.web.service.invoker.HttpRequestValues}
 *
 * This is the "currency" exchanged between the proxy layer ({@link HttpServiceMethod})
 * and the adapter layer ({@link HttpExchangeAdapter}). The proxy builds it from
 * annotations + method arguments; the adapter translates it into an actual HTTP call.
 *
 * <h3>How the real framework does it:</h3>
 * The real {@code HttpRequestValues} has a nested Builder class (~370 lines) that
 * handles URI template expansion, form data encoding, multipart assembly, cookie
 * formatting, and request attributes. The immutable value object has ~15 fields.
 *
 * Simplifications:
 * <ul>
 *   <li>No cookie support</li>
 *   <li>No multipart support</li>
 *   <li>No request attributes</li>
 *   <li>No UriBuilderFactory — URI templates are expanded with simple string replacement</li>
 *   <li>No API versioning</li>
 * </ul>
 */
public class HttpRequestValues {

    private final String httpMethod;
    private final String uriTemplate;
    private final Map<String, String> uriVariables;
    private final Map<String, String> headers;
    private final Map<String, String> queryParams;
    private final Object body;

    private HttpRequestValues(Builder builder) {
        this.httpMethod = builder.httpMethod;
        this.uriTemplate = builder.uriTemplate;
        this.uriVariables = Collections.unmodifiableMap(new LinkedHashMap<>(builder.uriVariables));
        this.headers = Collections.unmodifiableMap(new LinkedHashMap<>(builder.headers));
        this.queryParams = Collections.unmodifiableMap(new LinkedHashMap<>(builder.queryParams));
        this.body = builder.body;
    }

    public String getHttpMethod() {
        return httpMethod;
    }

    public String getUriTemplate() {
        return uriTemplate;
    }

    public Map<String, String> getUriVariables() {
        return uriVariables;
    }

    public Map<String, String> getHeaders() {
        return headers;
    }

    public Map<String, String> getQueryParams() {
        return queryParams;
    }

    public Object getBody() {
        return body;
    }

    /**
     * Build the final URI by expanding the template with variables and appending
     * query parameters.
     *
     * Maps to: the URI expansion logic in {@code HttpRequestValues.Builder.build()}
     * which delegates to a {@code UriBuilderFactory}. We use simple string replacement.
     */
    public URI getExpandedUri() {
        String expanded = uriTemplate;
        for (Map.Entry<String, String> entry : uriVariables.entrySet()) {
            expanded = expanded.replace("{" + entry.getKey() + "}", entry.getValue());
        }
        if (!queryParams.isEmpty()) {
            StringBuilder sb = new StringBuilder(expanded);
            sb.append(expanded.contains("?") ? "&" : "?");
            boolean first = true;
            for (Map.Entry<String, String> entry : queryParams.entrySet()) {
                if (!first) {
                    sb.append("&");
                }
                sb.append(entry.getKey()).append("=").append(entry.getValue());
                first = false;
            }
            expanded = sb.toString();
        }
        return URI.create(expanded);
    }

    public static Builder builder() {
        return new Builder();
    }

    /**
     * Mutable builder that accumulates request data from annotations and argument resolvers.
     *
     * Maps to: {@code HttpRequestValues.Builder} (inner class, ~370 lines in real framework)
     *
     * The real builder handles:
     * <ul>
     *   <li>Merging query params into the URI template or form body based on Content-Type</li>
     *   <li>Multipart body assembly</li>
     *   <li>Cookie header formatting</li>
     *   <li>UriBuilderFactory delegation for URI expansion</li>
     * </ul>
     *
     * We keep it simple: just setters for each field.
     */
    public static class Builder {
        private String httpMethod = "";
        private String uriTemplate = "";
        private final Map<String, String> uriVariables = new LinkedHashMap<>();
        private final Map<String, String> headers = new LinkedHashMap<>();
        private final Map<String, String> queryParams = new LinkedHashMap<>();
        private Object body;

        public Builder setHttpMethod(String httpMethod) {
            this.httpMethod = httpMethod;
            return this;
        }

        public Builder setUriTemplate(String uriTemplate) {
            this.uriTemplate = uriTemplate;
            return this;
        }

        public Builder setUriVariable(String name, String value) {
            this.uriVariables.put(name, value);
            return this;
        }

        public Builder addHeader(String name, String value) {
            this.headers.put(name, value);
            return this;
        }

        public Builder addQueryParam(String name, String value) {
            this.queryParams.put(name, value);
            return this;
        }

        public Builder setBody(Object body) {
            this.body = body;
            return this;
        }

        public HttpRequestValues build() {
            return new HttpRequestValues(this);
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/HttpExchangeAdapter.java` [NEW]

```java
package com.simplespringmvc.exchange;

/**
 * Strategy interface that decouples the proxy machinery from any specific HTTP client library.
 * Concrete implementations exist for different HTTP clients (JDK HttpClient, RestClient, etc.).
 *
 * Maps to: {@code org.springframework.web.service.invoker.HttpExchangeAdapter}
 *
 * <h3>How the real framework does it:</h3>
 * The real interface has 5 exchange methods for different return type shapes:
 * <pre>
 *   void exchange(HttpRequestValues)                                    → fire-and-forget
 *   HttpHeaders exchangeForHeaders(HttpRequestValues)                   → headers only
 *   T exchangeForBody(HttpRequestValues, ParameterizedTypeReference)    → decoded body
 *   ResponseEntity&lt;Void&gt; exchangeForBodilessEntity(HttpRequestValues)  → status + headers
 *   ResponseEntity&lt;T&gt; exchangeForEntity(HttpRequestValues, TypeRef)    → full response
 * </pre>
 *
 * Plus a reactive sub-interface for Mono/Flux returns.
 *
 * Simplifications:
 * <ul>
 *   <li>Only three exchange shapes: void, body-as-type, and body-as-String</li>
 *   <li>No ResponseEntity support</li>
 *   <li>No reactive (Mono/Flux) support</li>
 *   <li>Body type passed as Class instead of ParameterizedTypeReference</li>
 * </ul>
 */
public interface HttpExchangeAdapter {

    /**
     * Execute the request and discard the response (fire-and-forget).
     *
     * Maps to: {@code HttpExchangeAdapter.exchange(HttpRequestValues)}
     */
    void exchange(HttpRequestValues requestValues);

    /**
     * Execute the request and decode the response body to the given type.
     *
     * Maps to: {@code HttpExchangeAdapter.exchangeForBody(HttpRequestValues, ParameterizedTypeReference)}
     *
     * @param requestValues the request data
     * @param bodyType      the expected response body type
     * @return the decoded response body
     */
    <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType);
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/HttpServiceArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.mapping.MethodParameter;

/**
 * Strategy interface for resolving a method argument into part of an HTTP request.
 * Each implementation handles one kind of annotation ({@code @PathVariable},
 * {@code @RequestParam}, {@code @RequestBody}) and writes the resolved value
 * into the {@link HttpRequestValues.Builder}.
 *
 * Maps to: {@code org.springframework.web.service.invoker.HttpServiceArgumentResolver}
 *
 * <h3>How the real framework does it:</h3>
 * The real interface has a single method:
 * <pre>
 *   boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder builder)
 * </pre>
 * Returns true if the resolver handled the argument, false to pass to the next resolver.
 * This is the same "first match wins" pattern used by server-side
 * {@link com.simplespringmvc.adapter.HandlerMethodArgumentResolver}.
 *
 * Simplifications:
 * <ul>
 *   <li>Uses our simplified MethodParameter (no generic type resolution)</li>
 *   <li>No ConversionService integration — values are converted to String via toString()</li>
 * </ul>
 */
public interface HttpServiceArgumentResolver {

    /**
     * Try to resolve the given argument and populate the request builder.
     *
     * @param argument      the actual value passed by the caller (may be null)
     * @param parameter     metadata about the method parameter
     * @param requestValues the builder to populate
     * @return true if this resolver handled the argument
     */
    boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues);
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/ExchangePathVariableArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.mapping.MethodParameter;

/**
 * Resolves {@code @PathVariable}-annotated arguments by writing them as URI variables
 * into the {@link HttpRequestValues.Builder}.
 *
 * Maps to: {@code org.springframework.web.service.invoker.PathVariableArgumentResolver}
 *
 * This reuses the same {@code @PathVariable} annotation from the server side — the real
 * framework does this too. The annotation is in {@code org.springframework.web.bind.annotation},
 * shared by both server-side resolvers and client-side resolvers.
 *
 * <h3>How the real framework does it:</h3>
 * Extends {@code AbstractNamedValueArgumentResolver} which provides:
 * <ul>
 *   <li>Name extraction from annotation (with parameter name fallback)</li>
 *   <li>ConversionService integration for formatting values as Strings</li>
 *   <li>Required/optional handling</li>
 * </ul>
 * Then calls {@code requestValues.setUriVariable(name, value)}.
 *
 * Simplifications:
 * <ul>
 *   <li>No AbstractNamedValueArgumentResolver base class — logic is inline</li>
 *   <li>Values converted to String via toString() instead of ConversionService</li>
 * </ul>
 */
public class ExchangePathVariableArgumentResolver implements HttpServiceArgumentResolver {

    @Override
    public boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues) {
        PathVariable annotation = parameter.getParameterAnnotation(PathVariable.class);
        if (annotation == null) {
            return false;
        }

        String name = annotation.value();
        if (name.isEmpty()) {
            name = parameter.getParameterName();
        }

        requestValues.setUriVariable(name, String.valueOf(argument));
        return true;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/ExchangeRequestParamArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;

/**
 * Resolves {@code @RequestParam}-annotated arguments by writing them as query parameters
 * into the {@link HttpRequestValues.Builder}.
 *
 * Maps to: {@code org.springframework.web.service.invoker.RequestParamArgumentResolver}
 *
 * This reuses the same {@code @RequestParam} annotation from the server side.
 *
 * <h3>How the real framework does it:</h3>
 * Extends {@code AbstractNamedValueArgumentResolver} and calls
 * {@code requestValues.addRequestParameter(name, value)}. Supports multi-value
 * parameters (collections) and form data encoding based on Content-Type.
 *
 * Simplifications:
 * <ul>
 *   <li>No multi-value (collection) support</li>
 *   <li>No form data encoding — always appended as query parameters</li>
 *   <li>Values converted to String via toString()</li>
 * </ul>
 */
public class ExchangeRequestParamArgumentResolver implements HttpServiceArgumentResolver {

    @Override
    public boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues) {
        RequestParam annotation = parameter.getParameterAnnotation(RequestParam.class);
        if (annotation == null) {
            return false;
        }

        String name = annotation.value();
        if (name.isEmpty()) {
            name = parameter.getParameterName();
        }

        requestValues.addQueryParam(name, String.valueOf(argument));
        return true;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/ExchangeRequestBodyArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.mapping.MethodParameter;

/**
 * Resolves {@code @RequestBody}-annotated arguments by setting the body on the
 * {@link HttpRequestValues.Builder}.
 *
 * Maps to: {@code org.springframework.web.service.invoker.RequestBodyArgumentResolver}
 *
 * This reuses the same {@code @RequestBody} annotation from the server side.
 *
 * <h3>How the real framework does it:</h3>
 * The real resolver handles:
 * <ul>
 *   <li>StreamingHttpOutputMessage.Body for streaming writes</li>
 *   <li>Optional unwrapping</li>
 *   <li>Reactive types (Publisher) via ReactiveHttpRequestValues.Builder</li>
 *   <li>ParameterizedTypeReference preservation for generic types</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No streaming body support</li>
 *   <li>No Optional unwrapping</li>
 *   <li>No reactive type support</li>
 *   <li>Body is passed as-is; serialization is the adapter's responsibility</li>
 * </ul>
 */
public class ExchangeRequestBodyArgumentResolver implements HttpServiceArgumentResolver {

    @Override
    public boolean resolve(Object argument, MethodParameter parameter, HttpRequestValues.Builder requestValues) {
        RequestBody annotation = parameter.getParameterAnnotation(RequestBody.class);
        if (annotation == null) {
            return false;
        }

        requestValues.setBody(argument);
        return true;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/HttpServiceMethod.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.exchange.annotation.HttpExchange;
import com.simplespringmvc.mapping.MethodParameter;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.List;

/**
 * Encapsulates everything needed to invoke a single {@code @HttpExchange}-annotated method.
 * One instance is created per method during proxy creation.
 *
 * Maps to: {@code org.springframework.web.service.invoker.HttpServiceMethod}
 *
 * The three-phase pipeline:
 * <pre>
 *   1. Initialize builder with annotation-derived metadata (HTTP method, URL, headers)
 *   2. Resolve each argument via the chain of HttpServiceArgumentResolver instances
 *   3. Build HttpRequestValues and execute via the HttpExchangeAdapter
 * </pre>
 *
 * <h3>How the real framework does it:</h3>
 * <ul>
 *   <li>{@code HttpRequestValuesInitializer} — extracts static metadata at construction time</li>
 *   <li>{@code ResponseFunction} — selects the right adapter method based on return type
 *       (void, HttpHeaders, ResponseEntity, body, Mono, Flux, etc.)</li>
 *   <li>Supports Kotlin coroutines via continuation argument stripping</li>
 *   <li>Handles ${...} placeholder resolution via StringValueResolver</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No ResponseFunction — we only support void and typed body returns</li>
 *   <li>No placeholder resolution</li>
 *   <li>No reactive type support</li>
 *   <li>URL merging is simple string concatenation</li>
 * </ul>
 */
public class HttpServiceMethod {

    private final Method method;
    private final String httpMethod;
    private final String urlTemplate;
    private final String contentType;
    private final String[] accept;
    private final List<HttpServiceArgumentResolver> argumentResolvers;
    private final HttpExchangeAdapter adapter;

    /**
     * Create an HttpServiceMethod by extracting annotation metadata and binding
     * it to the resolvers and adapter.
     *
     * @param method             the interface method
     * @param serviceType        the interface class (for type-level annotations)
     * @param argumentResolvers  the chain of argument resolvers
     * @param adapter            the HTTP client adapter
     */
    public HttpServiceMethod(Method method, Class<?> serviceType,
                             List<HttpServiceArgumentResolver> argumentResolvers,
                             HttpExchangeAdapter adapter) {
        this.method = method;
        this.argumentResolvers = argumentResolvers;
        this.adapter = adapter;

        // --- Extract annotation metadata (done once at construction time) ---
        // Type-level @HttpExchange provides base URL, shared content type, accept
        HttpExchange typeExchange = serviceType.getAnnotation(HttpExchange.class);
        String baseUrl = (typeExchange != null) ? typeExchange.value() : "";
        String typeContentType = (typeExchange != null) ? typeExchange.contentType() : "";
        String[] typeAccept = (typeExchange != null) ? typeExchange.accept() : new String[0];

        // Method-level: direct @HttpExchange or meta-annotation on @GetExchange etc.
        HttpExchange methodExchange = MergedAnnotationUtils.findAnnotation(method, HttpExchange.class);

        // HTTP method comes from the annotation (direct or meta)
        this.httpMethod = (methodExchange != null) ? methodExchange.method() : "";

        // URL from the composed annotation's value() attribute (e.g., @GetExchange("/users/{id}"))
        String methodUrl = extractUrl(method);

        // Merge type-level base URL + method-level URL
        this.urlTemplate = mergeUrls(baseUrl, methodUrl);

        // Content type: method-level overrides type-level
        String methodContentType = extractContentType(method);
        this.contentType = !methodContentType.isEmpty() ? methodContentType : typeContentType;

        // Accept: method-level overrides type-level
        String[] methodAccept = extractAccept(method);
        this.accept = (methodAccept.length > 0) ? methodAccept : typeAccept;
    }

    /**
     * Invoke this method: build request values from annotations + arguments, then execute.
     *
     * Maps to: {@code HttpServiceMethod.invoke(Object[] arguments)}
     *
     * The three-phase pipeline:
     * 1. Initialize builder with static annotation metadata
     * 2. Resolve each argument into the builder
     * 3. Build and execute
     */
    public Object invoke(Object[] arguments) {
        // Phase 1: Initialize builder with annotation-derived values
        HttpRequestValues.Builder builder = HttpRequestValues.builder()
                .setHttpMethod(httpMethod)
                .setUriTemplate(urlTemplate);

        if (!contentType.isEmpty()) {
            builder.addHeader("Content-Type", contentType);
        }
        if (accept.length > 0) {
            builder.addHeader("Accept", String.join(", ", accept));
        }

        // Phase 2: Resolve each method argument
        if (arguments != null) {
            for (int i = 0; i < arguments.length; i++) {
                MethodParameter param = new MethodParameter(method, i);
                boolean resolved = false;
                for (HttpServiceArgumentResolver resolver : argumentResolvers) {
                    if (resolver.resolve(arguments[i], param, builder)) {
                        resolved = true;
                        break;
                    }
                }
                if (!resolved) {
                    throw new IllegalStateException(
                            "No resolver for parameter " + i + " of " + method);
                }
            }
        }

        // Phase 3: Build and execute
        HttpRequestValues requestValues = builder.build();

        if (method.getReturnType() == void.class) {
            adapter.exchange(requestValues);
            return null;
        } else {
            return adapter.exchangeForBody(requestValues, method.getReturnType());
        }
    }

    /**
     * Extract the URL from the method annotation — either from @HttpExchange.value()
     * or from the composed annotation's value() (e.g., @GetExchange("/path")).
     */
    private String extractUrl(Method method) {
        // Direct @HttpExchange
        HttpExchange direct = method.getAnnotation(HttpExchange.class);
        if (direct != null) {
            return direct.value();
        }

        // Composed annotation (e.g., @GetExchange) — read its value() attribute
        Annotation composed = MergedAnnotationUtils.findComposedAnnotation(method, HttpExchange.class);
        if (composed != null) {
            return MergedAnnotationUtils.getStringAttribute(composed, "value");
        }

        return "";
    }

    /**
     * Extract contentType from the composed annotation (e.g., @PostExchange(contentType = "...")).
     */
    private String extractContentType(Method method) {
        HttpExchange direct = method.getAnnotation(HttpExchange.class);
        if (direct != null) {
            return direct.contentType();
        }

        Annotation composed = MergedAnnotationUtils.findComposedAnnotation(method, HttpExchange.class);
        if (composed != null) {
            return MergedAnnotationUtils.getStringAttribute(composed, "contentType");
        }

        return "";
    }

    /**
     * Extract accept from the composed annotation (e.g., @GetExchange(accept = "...")).
     */
    private String[] extractAccept(Method method) {
        HttpExchange direct = method.getAnnotation(HttpExchange.class);
        if (direct != null && direct.accept().length > 0) {
            return direct.accept();
        }

        Annotation composed = MergedAnnotationUtils.findComposedAnnotation(method, HttpExchange.class);
        if (composed != null) {
            return MergedAnnotationUtils.getStringArrayAttribute(composed, "accept");
        }

        return new String[0];
    }

    /**
     * Merge type-level base URL with method-level URL, handling slashes.
     *
     * Maps to: {@code HttpRequestValuesInitializer} constructor logic that
     * combines type-level and method-level URL with smart slash handling.
     */
    private String mergeUrls(String baseUrl, String methodUrl) {
        if (baseUrl.isEmpty()) {
            return methodUrl;
        }
        if (methodUrl.isEmpty()) {
            return baseUrl;
        }
        // Avoid double slashes
        if (baseUrl.endsWith("/") && methodUrl.startsWith("/")) {
            return baseUrl + methodUrl.substring(1);
        }
        // Ensure at least one slash
        if (!baseUrl.endsWith("/") && !methodUrl.startsWith("/")) {
            return baseUrl + "/" + methodUrl;
        }
        return baseUrl + methodUrl;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/HttpServiceProxyFactory.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.exchange.annotation.HttpExchange;

import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Factory that creates JDK dynamic proxies for {@code @HttpExchange}-annotated interfaces.
 * This is the user-facing entry point for creating HTTP service clients.
 *
 * Maps to: {@code org.springframework.web.service.invoker.HttpServiceProxyFactory}
 *
 * Usage:
 * <pre>
 *   HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
 *       .exchangeAdapter(new JdkClientAdapter("http://localhost:8080", objectMapper))
 *       .build();
 *
 *   UserService client = factory.createClient(UserService.class);
 *   User user = client.getUser("42");
 * </pre>
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. Uses Spring's ProxyFactory (AOP) instead of raw JDK Proxy
 *   2. Wraps methods in a MethodInterceptor (Spring AOP concept)
 *   3. Handles default methods via InvocationHandler.invokeDefault()
 *   4. Strips Kotlin continuation arguments
 *   5. Supports custom argument resolvers, ConversionService, and StringValueResolver
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Uses JDK dynamic proxy directly instead of Spring's ProxyFactory</li>
 *   <li>No default method handling</li>
 *   <li>No custom argument resolver registration</li>
 *   <li>No ConversionService or StringValueResolver</li>
 *   <li>Fixed set of resolvers: @PathVariable, @RequestParam, @RequestBody</li>
 * </ul>
 */
public class HttpServiceProxyFactory {

    private final HttpExchangeAdapter adapter;
    private final List<HttpServiceArgumentResolver> argumentResolvers;

    private HttpServiceProxyFactory(Builder builder) {
        this.adapter = builder.adapter;
        this.argumentResolvers = initArgumentResolvers();
    }

    /**
     * Assemble the argument resolver chain.
     *
     * Maps to: {@code HttpServiceProxyFactory.Builder.initArgumentResolvers()}
     * which registers annotation-based resolvers (@PathVariable, @RequestParam,
     * @RequestBody, @RequestHeader, @CookieValue, etc.) and type-based resolvers
     * (URI, HttpMethod, UriBuilderFactory).
     *
     * Order matters: first match wins.
     */
    private List<HttpServiceArgumentResolver> initArgumentResolvers() {
        List<HttpServiceArgumentResolver> resolvers = new ArrayList<>();
        resolvers.add(new ExchangePathVariableArgumentResolver());
        resolvers.add(new ExchangeRequestParamArgumentResolver());
        resolvers.add(new ExchangeRequestBodyArgumentResolver());
        return resolvers;
    }

    /**
     * Create a proxy that implements the given interface, translating method calls
     * into HTTP requests.
     *
     * Maps to: {@code HttpServiceProxyFactory.createClient(Class)}
     *
     * <h3>How the real framework does it:</h3>
     * <ol>
     *   <li>Uses {@code MethodIntrospector.selectMethods()} to find @HttpExchange methods</li>
     *   <li>Creates an {@code HttpServiceMethod} for each</li>
     *   <li>Wraps them in an {@code HttpServiceMethodInterceptor} (MethodInterceptor)</li>
     *   <li>Builds a proxy via Spring's {@code ProxyFactory}</li>
     * </ol>
     *
     * We use JDK {@link Proxy} directly — same concept, simpler API.
     */
    @SuppressWarnings("unchecked")
    public <S> S createClient(Class<S> serviceType) {
        // Pre-compute HttpServiceMethod for each @HttpExchange method
        Map<Method, HttpServiceMethod> methodMap = new HashMap<>();
        for (Method method : serviceType.getMethods()) {
            if (MergedAnnotationUtils.hasAnnotation(method, HttpExchange.class)) {
                HttpServiceMethod serviceMethod = new HttpServiceMethod(
                        method, serviceType, argumentResolvers, adapter);
                methodMap.put(method, serviceMethod);
            }
        }

        // Create JDK dynamic proxy
        return (S) Proxy.newProxyInstance(
                serviceType.getClassLoader(),
                new Class<?>[]{serviceType},
                (proxy, method, args) -> {
                    // Handle Object methods (toString, equals, hashCode)
                    if (method.getDeclaringClass() == Object.class) {
                        return switch (method.getName()) {
                            case "toString" -> serviceType.getSimpleName() + "@HttpExchange proxy";
                            case "hashCode" -> System.identityHashCode(proxy);
                            case "equals" -> proxy == args[0];
                            default -> throw new UnsupportedOperationException(method.getName());
                        };
                    }

                    HttpServiceMethod serviceMethod = methodMap.get(method);
                    if (serviceMethod == null) {
                        throw new IllegalStateException(
                                "No @HttpExchange annotation on method: " + method);
                    }

                    return serviceMethod.invoke(args);
                }
        );
    }

    public static Builder builder() {
        return new Builder();
    }

    /**
     * Fluent builder for HttpServiceProxyFactory.
     *
     * Maps to: {@code HttpServiceProxyFactory.Builder} which supports:
     * <ul>
     *   <li>{@code exchangeAdapter()} — required, the HTTP client</li>
     *   <li>{@code customArgumentResolver()} — prepended before defaults</li>
     *   <li>{@code conversionService()} — for formatting values</li>
     *   <li>{@code embeddedValueResolver()} — for ${} placeholders</li>
     * </ul>
     *
     * We only require the adapter.
     */
    public static class Builder {
        private HttpExchangeAdapter adapter;

        public Builder exchangeAdapter(HttpExchangeAdapter adapter) {
            this.adapter = adapter;
            return this;
        }

        public HttpServiceProxyFactory build() {
            if (adapter == null) {
                throw new IllegalArgumentException("HttpExchangeAdapter is required");
            }
            return new HttpServiceProxyFactory(this);
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exchange/JdkClientAdapter.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.fasterxml.jackson.databind.ObjectMapper;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Map;

/**
 * {@link HttpExchangeAdapter} implementation backed by the JDK's built-in
 * {@link java.net.http.HttpClient} with Jackson for JSON serialization/deserialization.
 *
 * Maps to: {@code org.springframework.web.client.support.RestClientAdapter}
 * (which bridges to Spring's RestClient)
 *
 * <h3>Why JDK HttpClient instead of RestClient?</h3>
 * Our simplified project doesn't have Spring's RestClient/WebClient infrastructure.
 * The JDK HttpClient (introduced in Java 11) is a zero-dependency HTTP client
 * that demonstrates the adapter pattern perfectly: the proxy doesn't know or care
 * which HTTP library we use — it just calls {@code exchange()} or {@code exchangeForBody()}.
 *
 * <h3>How RestClientAdapter does it:</h3>
 * <ol>
 *   <li>{@code newRequest(values)} translates HttpRequestValues → RestClient.RequestBodySpec</li>
 *   <li>Sets URI, headers, cookies, body, and attributes on the spec</li>
 *   <li>Calls the appropriate retrieve method based on the desired return type</li>
 * </ol>
 *
 * We do the same translation, but targeting JDK HttpClient's request builder.
 *
 * Simplifications:
 * <ul>
 *   <li>Base URL is required (RestClientAdapter inherits from RestClient's UriBuilderFactory)</li>
 *   <li>Only supports JSON for request/response bodies</li>
 *   <li>No streaming body support</li>
 *   <li>No cookie handling</li>
 *   <li>No request attribute support</li>
 *   <li>Synchronous only (no async/reactive)</li>
 * </ul>
 */
public class JdkClientAdapter implements HttpExchangeAdapter {

    private final String baseUrl;
    private final ObjectMapper objectMapper;
    private final HttpClient httpClient;

    /**
     * Create an adapter with a base URL and an ObjectMapper for JSON processing.
     *
     * @param baseUrl      the base URL to prepend to relative URI templates
     *                     (e.g., "http://localhost:8080")
     * @param objectMapper the Jackson ObjectMapper for JSON serialization/deserialization
     */
    public JdkClientAdapter(String baseUrl, ObjectMapper objectMapper) {
        this.baseUrl = baseUrl.endsWith("/") ? baseUrl.substring(0, baseUrl.length() - 1) : baseUrl;
        this.objectMapper = objectMapper;
        this.httpClient = HttpClient.newHttpClient();
    }

    /**
     * Package-private constructor for testing with a custom HttpClient.
     */
    JdkClientAdapter(String baseUrl, ObjectMapper objectMapper, HttpClient httpClient) {
        this.baseUrl = baseUrl.endsWith("/") ? baseUrl.substring(0, baseUrl.length() - 1) : baseUrl;
        this.objectMapper = objectMapper;
        this.httpClient = httpClient;
    }

    @Override
    public void exchange(HttpRequestValues requestValues) {
        try {
            HttpRequest request = buildRequest(requestValues);
            httpClient.send(request, HttpResponse.BodyHandlers.discarding());
        } catch (Exception e) {
            throw new RuntimeException("HTTP exchange failed", e);
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType) {
        try {
            HttpRequest request = buildRequest(requestValues);
            HttpResponse<String> response = httpClient.send(request,
                    HttpResponse.BodyHandlers.ofString());

            String responseBody = response.body();
            if (responseBody == null || responseBody.isEmpty()) {
                return null;
            }

            // String return type — return the raw body
            if (bodyType == String.class) {
                return (T) responseBody;
            }

            // Otherwise, deserialize JSON
            return objectMapper.readValue(responseBody, bodyType);
        } catch (Exception e) {
            throw new RuntimeException("HTTP exchange failed", e);
        }
    }

    /**
     * Translate {@link HttpRequestValues} into a JDK {@link HttpRequest}.
     *
     * Maps to: {@code RestClientAdapter.newRequest(HttpRequestValues)} which
     * translates into a RestClient.RequestBodySpec.
     *
     * Steps:
     * 1. Resolve the full URI (base URL + expanded template + query params)
     * 2. Set the HTTP method with appropriate body publisher
     * 3. Copy headers
     */
    private HttpRequest buildRequest(HttpRequestValues requestValues) throws Exception {
        // Resolve URI: prepend base URL if the template is relative
        String uriTemplate = requestValues.getUriTemplate();
        String fullTemplate;
        if (uriTemplate.startsWith("http://") || uriTemplate.startsWith("https://")) {
            fullTemplate = uriTemplate;
        } else {
            String path = uriTemplate.startsWith("/") ? uriTemplate : "/" + uriTemplate;
            fullTemplate = baseUrl + path;
        }

        // Expand URI variables
        String expanded = fullTemplate;
        for (Map.Entry<String, String> entry : requestValues.getUriVariables().entrySet()) {
            expanded = expanded.replace("{" + entry.getKey() + "}", entry.getValue());
        }

        // Append query parameters
        if (!requestValues.getQueryParams().isEmpty()) {
            StringBuilder sb = new StringBuilder(expanded);
            sb.append(expanded.contains("?") ? "&" : "?");
            boolean first = true;
            for (Map.Entry<String, String> entry : requestValues.getQueryParams().entrySet()) {
                if (!first) sb.append("&");
                sb.append(entry.getKey()).append("=").append(entry.getValue());
                first = false;
            }
            expanded = sb.toString();
        }

        URI uri = URI.create(expanded);

        // Build the request
        HttpRequest.Builder builder = HttpRequest.newBuilder().uri(uri);

        // Set headers
        for (Map.Entry<String, String> header : requestValues.getHeaders().entrySet()) {
            builder.header(header.getKey(), header.getValue());
        }

        // Set method with body
        String method = requestValues.getHttpMethod().toUpperCase();
        if (requestValues.getBody() != null) {
            String json = objectMapper.writeValueAsString(requestValues.getBody());
            builder.method(method, HttpRequest.BodyPublishers.ofString(json));
            // Set Content-Type if not already set
            if (!requestValues.getHeaders().containsKey("Content-Type")) {
                builder.header("Content-Type", "application/json");
            }
        } else {
            builder.method(method, HttpRequest.BodyPublishers.noBody());
        }

        return builder.build();
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/exchange/HttpRequestValuesTest.java` [NEW]

```java
package com.simplespringmvc.exchange;

import org.junit.jupiter.api.Test;

import java.net.URI;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link HttpRequestValues} — the immutable value object and its builder.
 */
class HttpRequestValuesTest {

    @Test
    void shouldBuildRequestValuesWithAllFields() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setHttpMethod("POST")
                .setUriTemplate("/users/{id}")
                .setUriVariable("id", "42")
                .addHeader("Content-Type", "application/json")
                .addQueryParam("verbose", "true")
                .setBody("test body")
                .build();

        assertThat(values.getHttpMethod()).isEqualTo("POST");
        assertThat(values.getUriTemplate()).isEqualTo("/users/{id}");
        assertThat(values.getUriVariables()).containsEntry("id", "42");
        assertThat(values.getHeaders()).containsEntry("Content-Type", "application/json");
        assertThat(values.getQueryParams()).containsEntry("verbose", "true");
        assertThat(values.getBody()).isEqualTo("test body");
    }

    @Test
    void shouldExpandUriVariables() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setUriTemplate("http://localhost/users/{id}/posts/{postId}")
                .setUriVariable("id", "42")
                .setUriVariable("postId", "7")
                .build();

        URI expanded = values.getExpandedUri();
        assertThat(expanded.toString()).isEqualTo("http://localhost/users/42/posts/7");
    }

    @Test
    void shouldAppendQueryParams_WhenNoExistingParams() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setUriTemplate("http://localhost/search")
                .addQueryParam("q", "spring")
                .addQueryParam("page", "1")
                .build();

        URI expanded = values.getExpandedUri();
        assertThat(expanded.toString()).isEqualTo("http://localhost/search?q=spring&page=1");
    }

    @Test
    void shouldAppendQueryParams_WhenTemplateAlreadyHasParams() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setUriTemplate("http://localhost/search?sort=name")
                .addQueryParam("page", "2")
                .build();

        URI expanded = values.getExpandedUri();
        assertThat(expanded.toString()).isEqualTo("http://localhost/search?sort=name&page=2");
    }

    @Test
    void shouldReturnUnmodifiableCollections() {
        HttpRequestValues values = HttpRequestValues.builder()
                .setUriVariable("id", "1")
                .addHeader("Accept", "text/plain")
                .addQueryParam("q", "test")
                .build();

        assertThat(values.getUriVariables()).isUnmodifiable();
        assertThat(values.getHeaders()).isUnmodifiable();
        assertThat(values.getQueryParams()).isUnmodifiable();
    }

    @Test
    void shouldBuildWithDefaults_WhenNoFieldsSet() {
        HttpRequestValues values = HttpRequestValues.builder().build();

        assertThat(values.getHttpMethod()).isEmpty();
        assertThat(values.getUriTemplate()).isEmpty();
        assertThat(values.getUriVariables()).isEmpty();
        assertThat(values.getHeaders()).isEmpty();
        assertThat(values.getQueryParams()).isEmpty();
        assertThat(values.getBody()).isNull();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/exchange/ExchangeArgumentResolversTest.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for the three exchange argument resolvers:
 * {@link ExchangePathVariableArgumentResolver},
 * {@link ExchangeRequestParamArgumentResolver},
 * {@link ExchangeRequestBodyArgumentResolver}.
 */
class ExchangeArgumentResolversTest {

    // --- Sample methods for reflection ---
    @SuppressWarnings("unused")
    void pathVariableMethod(@PathVariable("id") String id) {}

    @SuppressWarnings("unused")
    void pathVariableByNameMethod(@PathVariable String userId) {}

    @SuppressWarnings("unused")
    void requestParamMethod(@RequestParam("page") int page) {}

    @SuppressWarnings("unused")
    void requestParamByNameMethod(@RequestParam String query) {}

    @SuppressWarnings("unused")
    void requestBodyMethod(@RequestBody Object body) {}

    @SuppressWarnings("unused")
    void unannotatedMethod(String plain) {}

    // --- PathVariable tests ---

    @Test
    void shouldResolvePathVariable_WhenAnnotationHasExplicitName() throws Exception {
        ExchangePathVariableArgumentResolver resolver = new ExchangePathVariableArgumentResolver();
        Method method = getClass().getDeclaredMethod("pathVariableMethod", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve("42", param, builder);

        assertThat(handled).isTrue();
        HttpRequestValues values = builder.build();
        assertThat(values.getUriVariables()).containsEntry("id", "42");
    }

    @Test
    void shouldResolvePathVariable_WhenAnnotationUsesParameterName() throws Exception {
        ExchangePathVariableArgumentResolver resolver = new ExchangePathVariableArgumentResolver();
        Method method = getClass().getDeclaredMethod("pathVariableByNameMethod", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve("user-99", param, builder);

        assertThat(handled).isTrue();
        HttpRequestValues values = builder.build();
        assertThat(values.getUriVariables()).containsEntry("userId", "user-99");
    }

    @Test
    void shouldNotResolve_WhenNoPathVariableAnnotation() throws Exception {
        ExchangePathVariableArgumentResolver resolver = new ExchangePathVariableArgumentResolver();
        Method method = getClass().getDeclaredMethod("unannotatedMethod", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve("value", param, builder);

        assertThat(handled).isFalse();
    }

    // --- RequestParam tests ---

    @Test
    void shouldResolveRequestParam_WhenAnnotationHasExplicitName() throws Exception {
        ExchangeRequestParamArgumentResolver resolver = new ExchangeRequestParamArgumentResolver();
        Method method = getClass().getDeclaredMethod("requestParamMethod", int.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve(5, param, builder);

        assertThat(handled).isTrue();
        HttpRequestValues values = builder.build();
        assertThat(values.getQueryParams()).containsEntry("page", "5");
    }

    @Test
    void shouldResolveRequestParam_WhenAnnotationUsesParameterName() throws Exception {
        ExchangeRequestParamArgumentResolver resolver = new ExchangeRequestParamArgumentResolver();
        Method method = getClass().getDeclaredMethod("requestParamByNameMethod", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve("spring", param, builder);

        assertThat(handled).isTrue();
        HttpRequestValues values = builder.build();
        assertThat(values.getQueryParams()).containsEntry("query", "spring");
    }

    @Test
    void shouldNotResolve_WhenNoRequestParamAnnotation() throws Exception {
        ExchangeRequestParamArgumentResolver resolver = new ExchangeRequestParamArgumentResolver();
        Method method = getClass().getDeclaredMethod("unannotatedMethod", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve("value", param, builder);

        assertThat(handled).isFalse();
    }

    // --- RequestBody tests ---

    @Test
    void shouldResolveRequestBody_WhenAnnotated() throws Exception {
        ExchangeRequestBodyArgumentResolver resolver = new ExchangeRequestBodyArgumentResolver();
        Method method = getClass().getDeclaredMethod("requestBodyMethod", Object.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        Object bodyObj = new Object();
        boolean handled = resolver.resolve(bodyObj, param, builder);

        assertThat(handled).isTrue();
        HttpRequestValues values = builder.build();
        assertThat(values.getBody()).isSameAs(bodyObj);
    }

    @Test
    void shouldNotResolve_WhenNoRequestBodyAnnotation() throws Exception {
        ExchangeRequestBodyArgumentResolver resolver = new ExchangeRequestBodyArgumentResolver();
        Method method = getClass().getDeclaredMethod("unannotatedMethod", String.class);
        MethodParameter param = new MethodParameter(method, 0);
        HttpRequestValues.Builder builder = HttpRequestValues.builder();

        boolean handled = resolver.resolve("value", param, builder);

        assertThat(handled).isFalse();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/exchange/HttpServiceMethodTest.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.exchange.annotation.DeleteExchange;
import com.simplespringmvc.exchange.annotation.GetExchange;
import com.simplespringmvc.exchange.annotation.HttpExchange;
import com.simplespringmvc.exchange.annotation.PostExchange;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link HttpServiceMethod} — the per-method invocation logic.
 */
class HttpServiceMethodTest {

    // --- Test interfaces ---

    @HttpExchange("/api/users")
    interface UserService {
        @GetExchange("/{id}")
        String getUser(@PathVariable("id") String id);

        @PostExchange(contentType = "application/json")
        void createUser(@RequestBody Object user);

        @GetExchange("/search")
        String search(@RequestParam("q") String query, @RequestParam("page") int page);

        @DeleteExchange("/{id}")
        void deleteUser(@PathVariable("id") String id);
    }

    interface NoBaseUrlService {
        @GetExchange("/items/{id}")
        String getItem(@PathVariable("id") String id);
    }

    @HttpExchange(value = "/api", accept = "application/json")
    interface AcceptService {
        @GetExchange("/data")
        String getData();
    }

    // --- Capturing adapter for testing ---

    static class CapturingAdapter implements HttpExchangeAdapter {
        final List<HttpRequestValues> captured = new ArrayList<>();
        String responseBody;

        @Override
        public void exchange(HttpRequestValues requestValues) {
            captured.add(requestValues);
        }

        @Override
        @SuppressWarnings("unchecked")
        public <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType) {
            captured.add(requestValues);
            return (T) responseBody;
        }
    }

    // --- Tests ---

    @Test
    void shouldMergeTypeAndMethodUrl_WhenGetExchangeWithPathVariable() throws Exception {
        CapturingAdapter adapter = new CapturingAdapter();
        adapter.responseBody = "John";
        HttpServiceMethod serviceMethod = new HttpServiceMethod(
                UserService.class.getMethod("getUser", String.class),
                UserService.class,
                List.of(new ExchangePathVariableArgumentResolver(),
                        new ExchangeRequestParamArgumentResolver(),
                        new ExchangeRequestBodyArgumentResolver()),
                adapter
        );

        Object result = serviceMethod.invoke(new Object[]{"42"});

        assertThat(result).isEqualTo("John");
        assertThat(adapter.captured).hasSize(1);
        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("GET");
        assertThat(values.getUriTemplate()).isEqualTo("/api/users/{id}");
        assertThat(values.getUriVariables()).containsEntry("id", "42");
    }

    @Test
    void shouldSetContentTypeHeader_WhenPostExchangeWithContentType() throws Exception {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceMethod serviceMethod = new HttpServiceMethod(
                UserService.class.getMethod("createUser", Object.class),
                UserService.class,
                List.of(new ExchangePathVariableArgumentResolver(),
                        new ExchangeRequestParamArgumentResolver(),
                        new ExchangeRequestBodyArgumentResolver()),
                adapter
        );

        Object body = new Object();
        serviceMethod.invoke(new Object[]{body});

        assertThat(adapter.captured).hasSize(1);
        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("POST");
        assertThat(values.getUriTemplate()).isEqualTo("/api/users");
        assertThat(values.getHeaders()).containsEntry("Content-Type", "application/json");
        assertThat(values.getBody()).isSameAs(body);
    }

    @Test
    void shouldResolveMultipleQueryParams() throws Exception {
        CapturingAdapter adapter = new CapturingAdapter();
        adapter.responseBody = "results";
        HttpServiceMethod serviceMethod = new HttpServiceMethod(
                UserService.class.getMethod("search", String.class, int.class),
                UserService.class,
                List.of(new ExchangePathVariableArgumentResolver(),
                        new ExchangeRequestParamArgumentResolver(),
                        new ExchangeRequestBodyArgumentResolver()),
                adapter
        );

        serviceMethod.invoke(new Object[]{"spring", 3});

        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getQueryParams()).containsEntry("q", "spring");
        assertThat(values.getQueryParams()).containsEntry("page", "3");
    }

    @Test
    void shouldUseDeleteMethod_WhenDeleteExchange() throws Exception {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceMethod serviceMethod = new HttpServiceMethod(
                UserService.class.getMethod("deleteUser", String.class),
                UserService.class,
                List.of(new ExchangePathVariableArgumentResolver(),
                        new ExchangeRequestParamArgumentResolver(),
                        new ExchangeRequestBodyArgumentResolver()),
                adapter
        );

        serviceMethod.invoke(new Object[]{"42"});

        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("DELETE");
        assertThat(values.getUriTemplate()).isEqualTo("/api/users/{id}");
    }

    @Test
    void shouldWorkWithoutBaseUrl() throws Exception {
        CapturingAdapter adapter = new CapturingAdapter();
        adapter.responseBody = "item";
        HttpServiceMethod serviceMethod = new HttpServiceMethod(
                NoBaseUrlService.class.getMethod("getItem", String.class),
                NoBaseUrlService.class,
                List.of(new ExchangePathVariableArgumentResolver(),
                        new ExchangeRequestParamArgumentResolver(),
                        new ExchangeRequestBodyArgumentResolver()),
                adapter
        );

        serviceMethod.invoke(new Object[]{"99"});

        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getUriTemplate()).isEqualTo("/items/{id}");
    }

    @Test
    void shouldInheritAcceptFromTypeLevel() throws Exception {
        CapturingAdapter adapter = new CapturingAdapter();
        adapter.responseBody = "data";
        HttpServiceMethod serviceMethod = new HttpServiceMethod(
                AcceptService.class.getMethod("getData"),
                AcceptService.class,
                List.of(new ExchangePathVariableArgumentResolver(),
                        new ExchangeRequestParamArgumentResolver(),
                        new ExchangeRequestBodyArgumentResolver()),
                adapter
        );

        serviceMethod.invoke(null);

        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHeaders()).containsEntry("Accept", "application/json");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/exchange/HttpServiceProxyFactoryTest.java` [NEW]

```java
package com.simplespringmvc.exchange;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.exchange.annotation.DeleteExchange;
import com.simplespringmvc.exchange.annotation.GetExchange;
import com.simplespringmvc.exchange.annotation.HttpExchange;
import com.simplespringmvc.exchange.annotation.PostExchange;
import com.simplespringmvc.exchange.annotation.PutExchange;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link HttpServiceProxyFactory} — the proxy creation entry point.
 */
class HttpServiceProxyFactoryTest {

    // --- Test interfaces ---

    @HttpExchange("/api/users")
    interface UserService {
        @GetExchange("/{id}")
        String getUser(@PathVariable("id") String id);

        @PostExchange(value = "", contentType = "application/json")
        void createUser(@RequestBody Object user);

        @GetExchange("/search")
        String search(@RequestParam("q") String query);

        @PutExchange("/{id}")
        void updateUser(@PathVariable("id") String id, @RequestBody Object user);

        @DeleteExchange("/{id}")
        void deleteUser(@PathVariable("id") String id);
    }

    // --- Capturing adapter ---

    static class CapturingAdapter implements HttpExchangeAdapter {
        final List<HttpRequestValues> captured = new ArrayList<>();
        String responseBody;

        @Override
        public void exchange(HttpRequestValues requestValues) {
            captured.add(requestValues);
        }

        @Override
        @SuppressWarnings("unchecked")
        public <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType) {
            captured.add(requestValues);
            return (T) responseBody;
        }
    }

    // --- Tests ---

    @Test
    void shouldCreateProxy_WhenValidInterface() {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);

        assertThat(proxy).isNotNull();
        assertThat(proxy.toString()).contains("UserService");
    }

    @Test
    void shouldRouteGetRequest_WhenProxyMethodCalled() {
        CapturingAdapter adapter = new CapturingAdapter();
        adapter.responseBody = "John Doe";
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);
        String result = proxy.getUser("42");

        assertThat(result).isEqualTo("John Doe");
        assertThat(adapter.captured).hasSize(1);
        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("GET");
        assertThat(values.getUriTemplate()).isEqualTo("/api/users/{id}");
        assertThat(values.getUriVariables()).containsEntry("id", "42");
    }

    @Test
    void shouldRoutePostRequest_WhenCreateUserCalled() {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);
        proxy.createUser("new user");

        assertThat(adapter.captured).hasSize(1);
        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("POST");
        assertThat(values.getBody()).isEqualTo("new user");
        assertThat(values.getHeaders()).containsEntry("Content-Type", "application/json");
    }

    @Test
    void shouldResolveQueryParams_WhenSearchCalled() {
        CapturingAdapter adapter = new CapturingAdapter();
        adapter.responseBody = "results";
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);
        String result = proxy.search("spring");

        assertThat(result).isEqualTo("results");
        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getQueryParams()).containsEntry("q", "spring");
    }

    @Test
    void shouldRoutePutRequest_WhenUpdateUserCalled() {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);
        proxy.updateUser("42", "updated");

        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("PUT");
        assertThat(values.getUriVariables()).containsEntry("id", "42");
        assertThat(values.getBody()).isEqualTo("updated");
    }

    @Test
    void shouldRouteDeleteRequest_WhenDeleteUserCalled() {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);
        proxy.deleteUser("42");

        HttpRequestValues values = adapter.captured.get(0);
        assertThat(values.getHttpMethod()).isEqualTo("DELETE");
        assertThat(values.getUriVariables()).containsEntry("id", "42");
    }

    @Test
    void shouldThrow_WhenAdapterNotProvided() {
        assertThatThrownBy(() -> HttpServiceProxyFactory.builder().build())
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("required");
    }

    @Test
    void shouldHandleObjectMethods_WhenCalledOnProxy() {
        CapturingAdapter adapter = new CapturingAdapter();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserService proxy = factory.createClient(UserService.class);

        // toString should not trigger HTTP call
        String str = proxy.toString();
        assertThat(str).contains("UserService");
        assertThat(adapter.captured).isEmpty();

        // hashCode should not trigger HTTP call
        int hash = proxy.hashCode();
        assertThat(hash).isNotZero();
        assertThat(adapter.captured).isEmpty();

        // equals should not trigger HTTP call
        boolean eq = proxy.equals(proxy);
        assertThat(eq).isTrue();
        assertThat(adapter.captured).isEmpty();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/HttpExchangeIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.exchange.HttpExchangeAdapter;
import com.simplespringmvc.exchange.HttpRequestValues;
import com.simplespringmvc.exchange.HttpServiceProxyFactory;
import com.simplespringmvc.exchange.JdkClientAdapter;
import com.simplespringmvc.exchange.annotation.DeleteExchange;
import com.simplespringmvc.exchange.annotation.GetExchange;
import com.simplespringmvc.exchange.annotation.HttpExchange;
import com.simplespringmvc.exchange.annotation.PostExchange;
import com.simplespringmvc.exchange.annotation.PutExchange;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration test for the @HttpExchange / HTTP Interface Clients feature.
 *
 * Tests the full proxy pipeline: annotations → metadata extraction → argument
 * resolution → HttpRequestValues → adapter execution.
 *
 * Uses a capturing adapter rather than a real HTTP server to keep the test
 * fast and focused on the proxy logic.
 */
class HttpExchangeIntegrationTest {

    // --- Domain objects ---

    public record User(String id, String name, String email) {}

    // --- Service interface using all annotation types ---

    @HttpExchange(value = "/api/v1/users", accept = "application/json")
    interface UserApi {

        @GetExchange("/{id}")
        User getUser(@PathVariable("id") String id);

        @GetExchange
        String listUsers(@RequestParam("page") int page, @RequestParam("size") int size);

        @PostExchange(contentType = "application/json")
        void createUser(@RequestBody User user);

        @PutExchange("/{id}")
        void updateUser(@PathVariable("id") String id, @RequestBody User user);

        @DeleteExchange("/{id}")
        void deleteUser(@PathVariable("id") String id);
    }

    // --- Capturing adapter that records all requests ---

    static class RecordingAdapter implements HttpExchangeAdapter {
        final List<HttpRequestValues> requests = new ArrayList<>();
        private Object nextResponse;

        void willReturn(Object response) {
            this.nextResponse = response;
        }

        @Override
        public void exchange(HttpRequestValues requestValues) {
            requests.add(requestValues);
        }

        @Override
        @SuppressWarnings("unchecked")
        public <T> T exchangeForBody(HttpRequestValues requestValues, Class<T> bodyType) {
            requests.add(requestValues);
            return (T) nextResponse;
        }

        HttpRequestValues lastRequest() {
            return requests.get(requests.size() - 1);
        }
    }

    @Test
    void shouldBuildCompleteGetRequest_WhenProxyMethodCalled() {
        RecordingAdapter adapter = new RecordingAdapter();
        adapter.willReturn(new User("42", "Alice", "alice@example.com"));

        UserApi api = createClient(adapter, UserApi.class);
        User user = api.getUser("42");

        assertThat(user.name()).isEqualTo("Alice");

        HttpRequestValues req = adapter.lastRequest();
        assertThat(req.getHttpMethod()).isEqualTo("GET");
        assertThat(req.getUriTemplate()).isEqualTo("/api/v1/users/{id}");
        assertThat(req.getUriVariables()).containsEntry("id", "42");
        assertThat(req.getHeaders()).containsEntry("Accept", "application/json");
    }

    @Test
    void shouldBuildGetWithQueryParams_WhenListUsersCalled() {
        RecordingAdapter adapter = new RecordingAdapter();
        adapter.willReturn("[{\"id\":\"1\"},{\"id\":\"2\"}]");

        UserApi api = createClient(adapter, UserApi.class);
        String result = api.listUsers(1, 20);

        assertThat(result).contains("1");

        HttpRequestValues req = adapter.lastRequest();
        assertThat(req.getHttpMethod()).isEqualTo("GET");
        assertThat(req.getUriTemplate()).isEqualTo("/api/v1/users");
        assertThat(req.getQueryParams()).containsEntry("page", "1");
        assertThat(req.getQueryParams()).containsEntry("size", "20");
    }

    @Test
    void shouldBuildPostWithBody_WhenCreateUserCalled() {
        RecordingAdapter adapter = new RecordingAdapter();

        UserApi api = createClient(adapter, UserApi.class);
        api.createUser(new User("1", "Bob", "bob@example.com"));

        HttpRequestValues req = adapter.lastRequest();
        assertThat(req.getHttpMethod()).isEqualTo("POST");
        assertThat(req.getUriTemplate()).isEqualTo("/api/v1/users");
        assertThat(req.getHeaders()).containsEntry("Content-Type", "application/json");
        assertThat(req.getBody()).isInstanceOf(User.class);
        User body = (User) req.getBody();
        assertThat(body.name()).isEqualTo("Bob");
    }

    @Test
    void shouldBuildPutWithPathVarAndBody_WhenUpdateUserCalled() {
        RecordingAdapter adapter = new RecordingAdapter();

        UserApi api = createClient(adapter, UserApi.class);
        api.updateUser("42", new User("42", "Alice Updated", "alice@new.com"));

        HttpRequestValues req = adapter.lastRequest();
        assertThat(req.getHttpMethod()).isEqualTo("PUT");
        assertThat(req.getUriTemplate()).isEqualTo("/api/v1/users/{id}");
        assertThat(req.getUriVariables()).containsEntry("id", "42");
        assertThat(req.getBody()).isInstanceOf(User.class);
    }

    @Test
    void shouldBuildDeleteRequest_WhenDeleteUserCalled() {
        RecordingAdapter adapter = new RecordingAdapter();

        UserApi api = createClient(adapter, UserApi.class);
        api.deleteUser("42");

        HttpRequestValues req = adapter.lastRequest();
        assertThat(req.getHttpMethod()).isEqualTo("DELETE");
        assertThat(req.getUriTemplate()).isEqualTo("/api/v1/users/{id}");
        assertThat(req.getUriVariables()).containsEntry("id", "42");
    }

    @Test
    void shouldSupportMultipleProxyCreation_WhenSameFactoryUsed() {
        RecordingAdapter adapter = new RecordingAdapter();

        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build();

        UserApi proxy1 = factory.createClient(UserApi.class);
        UserApi proxy2 = factory.createClient(UserApi.class);

        assertThat(proxy1).isNotSameAs(proxy2);
        assertThat(proxy1.toString()).contains("UserApi");
        assertThat(proxy2.toString()).contains("UserApi");
    }

    @Test
    void shouldCreateJdkClientAdapter_WhenBaseUrlProvided() {
        ObjectMapper mapper = new ObjectMapper();
        JdkClientAdapter adapter = new JdkClientAdapter("http://localhost:8080", mapper);

        // Verify it implements the interface
        assertThat(adapter).isInstanceOf(HttpExchangeAdapter.class);
    }

    private <T> T createClient(HttpExchangeAdapter adapter, Class<T> serviceType) {
        return HttpServiceProxyFactory.builder()
                .exchangeAdapter(adapter)
                .build()
                .createClient(serviceType);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **JDK Dynamic Proxy** | Runtime-generated object that implements an interface, routing all method calls through an InvocationHandler — the foundation of declarative clients |
| **HttpExchangeAdapter** | Strategy pattern decoupling the proxy from the HTTP transport — swap JDK HttpClient for RestClient without touching annotations |
| **HttpServiceMethod** | Per-method brain: extracts annotation metadata once at construction, runs the 3-phase pipeline (initialize → resolve → execute) on each call |
| **HttpRequestValues** | Immutable value object — the "currency" between proxy and adapter, carrying method, URL, headers, params, and body |
| **Meta-annotation** | `@GetExchange` is annotated with `@HttpExchange(method = "GET")` — the same compose-and-inherit pattern as `@GetMapping` / `@RequestMapping` |
| **Annotation reuse** | `@PathVariable`, `@RequestParam`, `@RequestBody` work on both server (extract from request) and client (insert into request) sides |

**This is the final chapter of Simple Spring MVC.** You've built a complete simplified web framework — from a bean container to an HTTP server, through the full request dispatch pipeline, and finally a declarative HTTP client. The journey covered 23 components across 7 architectural layers, all following the patterns in the real Spring Framework source code.
