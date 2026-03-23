# Chapter 10: DispatcherServlet & MVC

> **Build Challenge**
>
> | Current State | Limitation | Objective |
> |---|---|---|
> | Embedded Tomcat starts and binds a port (Feature 9) | No servlets registered — Tomcat serves nothing; HTTP requests get default 404 | Route HTTP requests to `@Controller` methods via `@GetMapping`/`@PostMapping`, resolve `@RequestParam` and `@PathVariable`, and write responses |

---

## 10.1 The Integration Point: ServletWebServerApplicationContext Creates the DispatcherServlet

The integration point is `ServletWebServerApplicationContext.createWebServer()` — the exact place where the MVC subsystem connects to the embedded server subsystem.

**Before this feature**, `createWebServer()` called `factory.getWebServer()` with no arguments, creating an empty Tomcat with no servlets. **After this feature**, it creates a `DispatcherServlet` and passes it to the factory for registration.

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/web/context/ServletWebServerApplicationContext.java`
**Change:** `createWebServer()` now creates a `DispatcherServlet` and passes it to the factory

```java
private void createWebServer() {
    ServletWebServerFactory factory = getWebServerFactory();
    DispatcherServlet dispatcherServlet = new DispatcherServlet(this);
    this.webServer = factory.getWebServer(dispatcherServlet);
}
```

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/web/server/servlet/ServletWebServerFactory.java`
**Change:** `getWebServer()` now accepts `Servlet... servlets` parameter

```java
@FunctionalInterface
public interface ServletWebServerFactory {
    WebServer getWebServer(Servlet... servlets);
}
```

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatServletWebServerFactory.java`
**Change:** `prepareContext()` registers each servlet with Tomcat and maps it to "/"

```java
private void prepareContext(Tomcat tomcat, Servlet... servlets) {
    Context context = tomcat.addContext("", createTempDir("tomcat-docbase").getAbsolutePath());
    context.setParentClassLoader(getClass().getClassLoader());

    // Register each servlet with the Tomcat context
    for (Servlet servlet : servlets) {
        String servletName = servlet.getClass().getSimpleName();
        Wrapper wrapper = Tomcat.addServlet(context, servletName, servlet);
        wrapper.setLoadOnStartup(1);
        context.addServletMappingDecoded("/", servletName);
    }
}
```

**Direction:** Two subsystems connect here — **Embedded Web Server** ↔ **MVC Dispatch Pipeline**. The `ServletWebServerApplicationContext` is the bridge. From this integration point, we need to build everything the DispatcherServlet requires: `HandlerMapping` (finds which method handles a request), `HandlerAdapter` (invokes that method), and the annotations that mark controller methods.

**Modifying:** `build.gradle` (root)
**Change:** Add `-parameters` compiler flag so method parameter names are available at runtime for `@RequestParam` and `@PathVariable` name resolution

```groovy
tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    options.compilerArgs += ['-parameters']
}
```

---

## 10.2 The Annotations: Marking Controllers and Their Endpoints

These annotations go in `iris-framework` (package `com.iris.framework.web.bind.annotation`) because they are part of the web framework layer, not the boot layer.

### RequestMethod enum

```java
package com.iris.framework.web.bind.annotation;

public enum RequestMethod {
    GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
}
```

### @RequestMapping — the universal mapping annotation

```java
package com.iris.framework.web.bind.annotation;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {
    String[] value() default {};
    String[] path() default {};
    RequestMethod[] method() default {};
}
```

When used on a class, `@RequestMapping("/api")` provides a URL prefix for all methods in that controller. When used on a method, it maps a specific path and HTTP method.

### @GetMapping and @PostMapping — composed shortcuts

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface GetMapping {
    String[] value() default {};
}
```

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PostMapping {
    String[] value() default {};
}
```

In the real Spring, these are meta-annotated with `@RequestMapping(method = GET)` and resolved via `@AliasFor`. We check for them independently in the handler mapping scanner — simpler and equally effective for our purposes.

### @RequestParam — bind query/form parameters

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestParam {
    String value() default "";
    boolean required() default true;
    String defaultValue() default "\n\t\t\n\t\t\n\uE000\uE001\uE002\n\t\t\t\t\n";
}
```

The `defaultValue` sentinel is the same Unicode string that real Spring uses — an unprintable value that no real default would ever match.

### @PathVariable — bind URI template variables

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface PathVariable {
    String value() default "";
}
```

---

## 10.3 The MVC Core: HandlerMapping, HandlerAdapter, and DispatcherServlet

### HandlerMethod — pairs a bean with its method

```java
package com.iris.framework.web.servlet;

public class HandlerMethod {
    private final Object bean;
    private final Method method;

    public HandlerMethod(Object bean, Method method) {
        this.bean = bean;
        this.method = method;
    }

    public Object getBean() { return bean; }
    public Method getMethod() { return method; }
}
```

### HandlerMapping — strategy for finding the handler

```java
public interface HandlerMapping {
    String PATH_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".pathVariables";

    HandlerMethod getHandler(HttpServletRequest request);
}
```

The `PATH_VARIABLES_ATTRIBUTE` constant is the request attribute key where extracted path variables are stored as a `Map<String, String>`. The HandlerMapping sets it; the HandlerAdapter reads it.

### HandlerAdapter — strategy for invoking the handler

```java
public interface HandlerAdapter {
    boolean supports(Object handler);
    void handle(HttpServletRequest request, HttpServletResponse response,
                HandlerMethod handler) throws Exception;
}
```

### RequestMappingInfo — path pattern + HTTP method

```java
public class RequestMappingInfo {
    private final String pattern;
    private final RequestMethod requestMethod;
    private final String[] patternSegments;

    public RequestMappingInfo(String pattern, RequestMethod requestMethod) {
        this.pattern = normalizePath(pattern);
        this.requestMethod = requestMethod;
        this.patternSegments = splitPath(this.pattern);
    }

    public Map<String, String> match(String requestPath, String httpMethod) {
        // Check HTTP method
        if (this.requestMethod != null) {
            if (!this.requestMethod.name().equalsIgnoreCase(httpMethod)) {
                return null;
            }
        }
        // Split and compare segments
        String[] requestSegments = splitPath(normalizePath(requestPath));
        if (requestSegments.length != patternSegments.length) return null;

        Map<String, String> pathVariables = new LinkedHashMap<>();
        for (int i = 0; i < patternSegments.length; i++) {
            String patternSeg = patternSegments[i];
            String requestSeg = requestSegments[i];
            if (patternSeg.startsWith("{") && patternSeg.endsWith("}")) {
                String varName = patternSeg.substring(1, patternSeg.length() - 1);
                pathVariables.put(varName, requestSeg);
            } else if (!patternSeg.equals(requestSeg)) {
                return null;
            }
        }
        return pathVariables;
    }
}
```

The matching is segment-by-segment: literal segments must match exactly, `{name}` segments capture values. Returns `null` on no match, or an (possibly empty) map of path variables on match.

### RequestMappingHandlerMapping — scans @Controller beans

This is the heart of the routing system. On construction, it scans the ApplicationContext:

```java
public class RequestMappingHandlerMapping implements HandlerMapping {
    private final List<MappingRegistration> registry = new ArrayList<>();

    public RequestMappingHandlerMapping(ApplicationContext context) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        for (String name : factory.getBeanDefinitionNames()) {
            Object bean = factory.getBean(name);
            if (isController(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    @Override
    public HandlerMethod getHandler(HttpServletRequest request) {
        String path = getRequestPath(request);
        String method = request.getMethod();
        for (MappingRegistration registration : registry) {
            Map<String, String> pathVariables = registration.info.match(path, method);
            if (pathVariables != null) {
                request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);
                return registration.handler;
            }
        }
        return null;
    }
}
```

The `detectHandlerMethods()` method checks for class-level `@RequestMapping` (URL prefix), then scans each method for `@GetMapping`, `@PostMapping`, or `@RequestMapping` to build `RequestMappingInfo` → `HandlerMethod` pairs.

### RequestMappingHandlerAdapter — resolves parameters and invokes

```java
public class RequestMappingHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       HandlerMethod handler) throws Exception {
        Method method = handler.getMethod();
        method.setAccessible(true);

        // Resolve all method parameters
        Object[] args = resolveArguments(method, request, response);

        // Invoke the handler method
        Object result = method.invoke(handler.getBean(), args);

        // Write return value to response (plain text for now)
        if (result != null && !response.isCommitted()) {
            response.setContentType("text/plain;charset=UTF-8");
            response.getWriter().write(result.toString());
            response.getWriter().flush();
        }
    }
}
```

Parameter resolution follows this priority:
1. `HttpServletRequest` / `HttpServletResponse` → inject directly
2. `@PathVariable` → extract from path variables (set by HandlerMapping)
3. `@RequestParam` → extract from query/form parameters

Type conversion handles `String`, `int`/`Integer`, `long`/`Long`, `boolean`/`Boolean`, `double`/`Double`.

### DispatcherServlet — the front controller

```java
public class DispatcherServlet extends HttpServlet {
    private final ApplicationContext applicationContext;
    private HandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    public DispatcherServlet(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Override
    public void init() throws ServletException {
        this.handlerMapping = new RequestMappingHandlerMapping(applicationContext);
        this.handlerAdapter = new RequestMappingHandlerAdapter();
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            // ... wrap in ServletException
        }
    }

    private void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        // Step 1: Find handler
        HandlerMethod handler = handlerMapping.getHandler(request);
        if (handler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }
        // Step 2: Verify adapter supports it
        if (!handlerAdapter.supports(handler)) {
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            return;
        }
        // Step 3: Invoke handler
        handlerAdapter.handle(request, response, handler);
    }
}
```

The three-step `doDispatch()` is the simplified version of real Spring's 9-step `DispatcherServlet.doDispatch()` (line 935). The core flow is identical: find handler → find adapter → invoke.

**Key timing:** The DispatcherServlet is created during `onRefresh()` (before beans are instantiated), but `init()` is called by Tomcat during `start()` (after `refresh()` completes and all singletons exist). This ensures all `@Controller` beans are available for scanning.

---

## 10.4 Try It Yourself

<details><summary>Challenge 1: Implement path matching for /users/{id}</summary>

Given the pattern `/users/{id}` and request path `/users/42`, implement matching that extracts `id=42`:

```java
public Map<String, String> match(String requestPath, String httpMethod) {
    if (this.requestMethod != null &&
        !this.requestMethod.name().equalsIgnoreCase(httpMethod)) {
        return null;
    }

    String[] requestSegments = splitPath(normalizePath(requestPath));
    if (requestSegments.length != patternSegments.length) return null;

    Map<String, String> pathVariables = new LinkedHashMap<>();
    for (int i = 0; i < patternSegments.length; i++) {
        String patternSeg = patternSegments[i];
        String requestSeg = requestSegments[i];
        if (patternSeg.startsWith("{") && patternSeg.endsWith("}")) {
            pathVariables.put(patternSeg.substring(1, patternSeg.length() - 1), requestSeg);
        } else if (!patternSeg.equals(requestSeg)) {
            return null;
        }
    }
    return pathVariables;
}
```

</details>

<details><summary>Challenge 2: Implement @RequestParam resolution</summary>

Given a method parameter annotated with `@RequestParam("q")`, extract the value from the HTTP request:

```java
RequestParam reqParam = param.getAnnotation(RequestParam.class);
if (reqParam != null) {
    String name = reqParam.value().isEmpty() ? param.getName() : reqParam.value();
    String value = request.getParameter(name);
    if (value == null) {
        String defaultValue = reqParam.defaultValue();
        if (!NO_DEFAULT.equals(defaultValue)) {
            value = defaultValue;
        } else if (reqParam.required()) {
            throw new IllegalArgumentException(
                    "Required request parameter '" + name + "' is not present");
        }
    }
    return convertValue(value, type);
}
```

</details>

<details><summary>Challenge 3: Wire the DispatcherServlet into Tomcat</summary>

Modify `ServletWebServerApplicationContext.createWebServer()` to create and register a DispatcherServlet:

```java
private void createWebServer() {
    ServletWebServerFactory factory = getWebServerFactory();
    DispatcherServlet dispatcherServlet = new DispatcherServlet(this);
    this.webServer = factory.getWebServer(dispatcherServlet);
}
```

And modify `TomcatServletWebServerFactory.prepareContext()` to register the servlet with Tomcat:

```java
for (Servlet servlet : servlets) {
    String servletName = servlet.getClass().getSimpleName();
    Wrapper wrapper = Tomcat.addServlet(context, servletName, servlet);
    wrapper.setLoadOnStartup(1);
    context.addServletMappingDecoded("/", servletName);
}
```

</details>

---

## 10.5 Tests

### Unit Tests

**`RequestMappingInfoTest`** — path pattern matching:

```java
@Test
void shouldMatchExactPath_WhenPathIsLiteral() {
    RequestMappingInfo info = new RequestMappingInfo("/users", RequestMethod.GET);
    Map<String, String> result = info.match("/users", "GET");
    assertThat(result).isNotNull().isEmpty();
}

@Test
void shouldExtractPathVariable_WhenPatternContainsVariable() {
    RequestMappingInfo info = new RequestMappingInfo("/users/{id}", RequestMethod.GET);
    Map<String, String> result = info.match("/users/42", "GET");
    assertThat(result).isNotNull().containsEntry("id", "42");
}

@Test
void shouldReturnNull_WhenHttpMethodDoesNotMatch() {
    RequestMappingInfo info = new RequestMappingInfo("/users", RequestMethod.GET);
    assertThat(info.match("/users", "POST")).isNull();
}
```

**`RequestMappingHandlerMappingTest`** — controller scanning:

```java
@Test
void shouldDetectGetMappingHandler_WhenControllerHasGetMapping() {
    context = new AnnotationConfigApplicationContext(SimpleControllerConfig.class);
    RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping(context);

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/hello");
    HandlerMethod handler = mapping.getHandler(request);

    assertThat(handler).isNotNull();
    assertThat(handler.getMethod().getName()).isEqualTo("hello");
}

@Test
void shouldApplyClassLevelPrefix_WhenControllerHasRequestMapping() {
    context = new AnnotationConfigApplicationContext(PrefixedControllerConfig.class);
    RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping(context);

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/items");
    assertThat(mapping.getHandler(request)).isNotNull();
}
```

**`RequestMappingHandlerAdapterTest`** — parameter resolution and invocation:

```java
@Test
void shouldResolveRequestParam_WhenAnnotationPresent() throws Exception {
    HandlerMethod handler = createHandler("withRequestParam");
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/test");
    request.addParameter("name", "Alice");
    MockHttpServletResponse response = new MockHttpServletResponse();

    adapter.handle(request, response, handler);
    assertThat(response.getContentAsString()).isEqualTo("Hello Alice");
}

@Test
void shouldConvertPathVariableToInt_WhenParameterIsInt() throws Exception {
    HandlerMethod handler = createHandler("withIntPathVariable");
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/users/7");
    request.setAttribute(HandlerMapping.PATH_VARIABLES_ATTRIBUTE, Map.of("id", "7"));
    MockHttpServletResponse response = new MockHttpServletResponse();

    adapter.handle(request, response, handler);
    assertThat(response.getContentAsString()).isEqualTo("User ID: 7");
}
```

**`DispatcherServletTest`** — full dispatch pipeline with mock objects:

```java
@Test
void shouldDispatchGetRequest_WhenHandlerExists() throws Exception {
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/greet");
    MockHttpServletResponse response = new MockHttpServletResponse();

    servlet.service(request, response);

    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(response.getContentAsString()).isEqualTo("Hello from Iris MVC!");
}

@Test
void shouldReturn404_WhenNoHandlerFound() throws Exception {
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/unknown");
    MockHttpServletResponse response = new MockHttpServletResponse();

    servlet.service(request, response);
    assertThat(response.getStatus()).isEqualTo(HttpServletResponse.SC_NOT_FOUND);
}
```

### Integration Test

**`DispatcherServletMvcIntegrationTest`** — full end-to-end with embedded Tomcat and real HTTP:

```java
@Test
void shouldReturnResponse_WhenGetEndpointCalled() throws Exception {
    HttpResponse<String> response = get("/hello");
    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("Hello from Iris!");
}

@Test
void shouldResolvePathVariable_WhenPathContainsVariable() throws Exception {
    HttpResponse<String> response = get("/users/42");
    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("User ID: 42");
}

@Test
void shouldHandlePostRequest_WhenPostMappingDefined() throws Exception {
    HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("http://localhost:" + port + "/items"))
            .header("Content-Type", "application/x-www-form-urlencoded")
            .POST(HttpRequest.BodyPublishers.ofString("name=Widget"))
            .build();
    HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
    assertThat(response.body()).isEqualTo("Created: Widget");
}
```

---

## 10.6 Why This Works

`★ Insight ─────────────────────────────────────`
**Front Controller + Strategy Pattern = Extensible Dispatch.**
The DispatcherServlet is the single entry point (Front Controller), but it doesn't know how to find or invoke handlers. It delegates to `HandlerMapping` (find) and `HandlerAdapter` (invoke) — both are Strategy interfaces. This means you can swap routing strategies without touching the dispatch loop. Real Spring ships 5+ HandlerMapping implementations and 3+ HandlerAdapter implementations, all coexisting. Our simplified version has one of each, but the architecture is identical.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Path Variables via Request Attributes — Decoupling Matching from Invocation.**
The HandlerMapping extracts path variables during matching and stores them as a request attribute (`PATH_VARIABLES_ATTRIBUTE`). The HandlerAdapter reads them during parameter resolution. This decouples two concerns: the mapping layer knows about URL patterns; the adapter layer knows about method signatures. Neither knows about the other's internals. This is exactly how real Spring works — `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE` is the same pattern.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Timing: Create Early, Initialize Late.**
The DispatcherServlet is *created* during `onRefresh()` (step 3 of refresh, before beans are instantiated) but *initialized* during Tomcat `start()` (after refresh completes). This two-phase lifecycle is essential: the servlet must be registered with Tomcat before the server starts, but it can't scan for `@Controller` beans until they exist. Setting `loadOnStartup=1` ensures Tomcat calls `init()` during `start()`, not on the first HTTP request.
`─────────────────────────────────────────────────`

---

## 10.7 What We Enhanced

| File | What Changed | Why |
|---|---|---|
| `ServletWebServerFactory` | `getWebServer()` → `getWebServer(Servlet...)` | Factory must accept servlets to register with the embedded server |
| `TomcatServletWebServerFactory` | `prepareContext()` now registers servlets via `Tomcat.addServlet()` | Each servlet is mapped to "/" with `loadOnStartup=1` |
| `ServletWebServerApplicationContext` | `createWebServer()` creates a `DispatcherServlet(this)` | The context itself becomes the DispatcherServlet's ApplicationContext |
| `build.gradle` (root) | Added `-parameters` compiler flag | Method parameter names available at runtime for @RequestParam/@PathVariable resolution |

---

## 10.8 Connection to Real Spring Framework

All file references are from Spring Framework commit `11ab0b4351`.

| Iris Class | Real Spring Class | Key File & Line |
|---|---|---|
| `DispatcherServlet` | `DispatcherServlet` | `spring-webmvc/.../web/servlet/DispatcherServlet.java` — `doDispatch()` at line 935 |
| `HandlerMapping` | `HandlerMapping` | `spring-webmvc/.../web/servlet/HandlerMapping.java` |
| `HandlerAdapter` | `HandlerAdapter` | `spring-webmvc/.../web/servlet/HandlerAdapter.java` |
| `RequestMappingHandlerMapping` | `RequestMappingHandlerMapping` | `spring-webmvc/.../mvc/method/annotation/RequestMappingHandlerMapping.java` — `getMappingForMethod()` at line 204 |
| `RequestMappingHandlerAdapter` | `RequestMappingHandlerAdapter` | `spring-webmvc/.../mvc/method/annotation/RequestMappingHandlerAdapter.java` — `invokeHandlerMethod()` at line 885 |
| `RequestMappingInfo` | `RequestMappingInfo` + `PathPattern` | `spring-webmvc/.../mvc/method/RequestMappingInfo.java` + `spring-web/.../web/util/pattern/PathPattern.java` |
| `HandlerMethod` | `HandlerMethod` | `spring-web/.../web/method/HandlerMethod.java` |
| `@RequestMapping` | `@RequestMapping` | `spring-web/.../web/bind/annotation/RequestMapping.java` |
| `@GetMapping` | `@GetMapping` | `spring-web/.../web/bind/annotation/GetMapping.java` |
| `@PostMapping` | `@PostMapping` | `spring-web/.../web/bind/annotation/PostMapping.java` |
| `@RequestParam` | `@RequestParam` | `spring-web/.../web/bind/annotation/RequestParam.java` |
| `@PathVariable` | `@PathVariable` | `spring-web/.../web/bind/annotation/PathVariable.java` |

**Key differences from real Spring:**
- Real Spring uses `HandlerExecutionChain` (handler + interceptors) — we return `HandlerMethod` directly
- Real Spring returns `ModelAndView` from the adapter — we write to the response directly
- Real Spring resolves parameters via a pluggable `HandlerMethodArgumentResolver` SPI with 30+ implementations — we inline the 4 most common resolvers
- Real Spring uses `PathPattern` (a compiled regex-like matcher) — we use segment-by-segment comparison
- Real Spring discovers `HandlerMapping`/`HandlerAdapter` beans from the context — we create them directly in `DispatcherServlet.init()`

---

## 10.9 Complete Code

### Production Code

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/RequestMethod.java`

```java
package com.iris.framework.web.bind.annotation;

public enum RequestMethod {
    GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/RequestMapping.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {
    String[] value() default {};
    String[] path() default {};
    RequestMethod[] method() default {};
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/GetMapping.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface GetMapping {
    String[] value() default {};
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/PostMapping.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PostMapping {
    String[] value() default {};
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/RequestParam.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestParam {
    String value() default "";
    boolean required() default true;
    String defaultValue() default "\n\t\t\n\t\t\n\uE000\uE001\uE002\n\t\t\t\t\n";
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/PathVariable.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface PathVariable {
    String value() default "";
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/HandlerMethod.java`

```java
package com.iris.framework.web.servlet;

import java.lang.reflect.Method;

public class HandlerMethod {
    private final Object bean;
    private final Method method;

    public HandlerMethod(Object bean, Method method) {
        this.bean = bean;
        this.method = method;
    }

    public Object getBean() { return bean; }
    public Method getMethod() { return method; }

    @Override
    public String toString() {
        return bean.getClass().getSimpleName() + "#" + method.getName();
    }
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/HandlerMapping.java`

```java
package com.iris.framework.web.servlet;

import jakarta.servlet.http.HttpServletRequest;

public interface HandlerMapping {
    String PATH_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".pathVariables";

    HandlerMethod getHandler(HttpServletRequest request);
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/HandlerAdapter.java`

```java
package com.iris.framework.web.servlet;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public interface HandlerAdapter {
    boolean supports(Object handler);
    void handle(HttpServletRequest request, HttpServletResponse response,
                HandlerMethod handler) throws Exception;
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingInfo.java`

```java
package com.iris.framework.web.servlet;

import java.util.LinkedHashMap;
import java.util.Map;
import com.iris.framework.web.bind.annotation.RequestMethod;

public class RequestMappingInfo {
    private final String pattern;
    private final RequestMethod requestMethod;
    private final String[] patternSegments;

    public RequestMappingInfo(String pattern, RequestMethod requestMethod) {
        this.pattern = normalizePath(pattern);
        this.requestMethod = requestMethod;
        this.patternSegments = splitPath(this.pattern);
    }

    public Map<String, String> match(String requestPath, String httpMethod) {
        if (this.requestMethod != null) {
            if (!this.requestMethod.name().equalsIgnoreCase(httpMethod)) {
                return null;
            }
        }
        String normalizedPath = normalizePath(requestPath);
        String[] requestSegments = splitPath(normalizedPath);
        if (requestSegments.length != patternSegments.length) {
            return null;
        }
        Map<String, String> pathVariables = new LinkedHashMap<>();
        for (int i = 0; i < patternSegments.length; i++) {
            String patternSeg = patternSegments[i];
            String requestSeg = requestSegments[i];
            if (patternSeg.startsWith("{") && patternSeg.endsWith("}")) {
                String varName = patternSeg.substring(1, patternSeg.length() - 1);
                pathVariables.put(varName, requestSeg);
            } else {
                if (!patternSeg.equals(requestSeg)) {
                    return null;
                }
            }
        }
        return pathVariables;
    }

    public String getPattern() { return pattern; }
    public RequestMethod getRequestMethod() { return requestMethod; }

    @Override
    public String toString() {
        return (requestMethod != null ? requestMethod + " " : "") + pattern;
    }

    private static String normalizePath(String path) {
        if (path == null || path.isEmpty()) return "/";
        if (!path.startsWith("/")) path = "/" + path;
        if (path.length() > 1 && path.endsWith("/")) path = path.substring(0, path.length() - 1);
        return path;
    }

    private static String[] splitPath(String path) {
        if ("/".equals(path)) return new String[0];
        return path.substring(1).split("/");
    }
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingHandlerMapping.java`

```java
package com.iris.framework.web.servlet;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import jakarta.servlet.http.HttpServletRequest;

import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.web.bind.annotation.GetMapping;
import com.iris.framework.web.bind.annotation.PostMapping;
import com.iris.framework.web.bind.annotation.RequestMapping;
import com.iris.framework.web.bind.annotation.RequestMethod;

public class RequestMappingHandlerMapping implements HandlerMapping {
    private final List<MappingRegistration> registry = new ArrayList<>();

    public RequestMappingHandlerMapping(ApplicationContext context) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        for (String name : factory.getBeanDefinitionNames()) {
            Object bean = factory.getBean(name);
            if (isController(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    @Override
    public HandlerMethod getHandler(HttpServletRequest request) {
        String path = getRequestPath(request);
        String method = request.getMethod();
        for (MappingRegistration registration : registry) {
            Map<String, String> pathVariables = registration.info.match(path, method);
            if (pathVariables != null) {
                request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);
                return registration.handler;
            }
        }
        return null;
    }

    public int getHandlerMethodCount() { return registry.size(); }

    private boolean isController(Class<?> clazz) {
        return clazz.isAnnotationPresent(Controller.class);
    }

    private void detectHandlerMethods(Object bean) {
        Class<?> clazz = bean.getClass();
        String classPrefix = getClassLevelPath(clazz);
        for (Method method : clazz.getDeclaredMethods()) {
            List<RequestMappingInfo> infos = getMappingInfos(method, classPrefix);
            for (RequestMappingInfo info : infos) {
                registry.add(new MappingRegistration(info, new HandlerMethod(bean, method)));
            }
        }
    }

    private String getClassLevelPath(Class<?> clazz) {
        RequestMapping rm = clazz.getAnnotation(RequestMapping.class);
        if (rm != null) {
            String path = getFirstPath(rm.value(), rm.path());
            if (path != null) return path;
        }
        return "";
    }

    private List<RequestMappingInfo> getMappingInfos(Method method, String classPrefix) {
        List<RequestMappingInfo> infos = new ArrayList<>();
        RequestMapping rm = method.getAnnotation(RequestMapping.class);
        if (rm != null) {
            String path = classPrefix + getPathOrDefault(rm.value(), rm.path());
            RequestMethod[] methods = rm.method();
            if (methods.length == 0) {
                infos.add(new RequestMappingInfo(path, null));
            } else {
                for (RequestMethod m : methods) {
                    infos.add(new RequestMappingInfo(path, m));
                }
            }
        }
        GetMapping gm = method.getAnnotation(GetMapping.class);
        if (gm != null) {
            String path = classPrefix + getPathOrDefault(gm.value(), new String[0]);
            infos.add(new RequestMappingInfo(path, RequestMethod.GET));
        }
        PostMapping pm = method.getAnnotation(PostMapping.class);
        if (pm != null) {
            String path = classPrefix + getPathOrDefault(pm.value(), new String[0]);
            infos.add(new RequestMappingInfo(path, RequestMethod.POST));
        }
        return infos;
    }

    private String getRequestPath(HttpServletRequest request) {
        String uri = request.getRequestURI();
        String contextPath = request.getContextPath();
        if (contextPath != null && !contextPath.isEmpty() && uri.startsWith(contextPath)) {
            uri = uri.substring(contextPath.length());
        }
        return uri;
    }

    private String getPathOrDefault(String[] value, String[] path) {
        String result = getFirstPath(value, path);
        return result != null ? result : "";
    }

    private String getFirstPath(String[] value, String[] path) {
        if (value.length > 0 && !value[0].isEmpty()) return value[0];
        if (path.length > 0 && !path[0].isEmpty()) return path[0];
        return null;
    }

    private record MappingRegistration(RequestMappingInfo info, HandlerMethod handler) {}
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingHandlerAdapter.java`

```java
package com.iris.framework.web.servlet;

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.Collections;
import java.util.Map;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import com.iris.framework.web.bind.annotation.PathVariable;
import com.iris.framework.web.bind.annotation.RequestParam;

public class RequestMappingHandlerAdapter implements HandlerAdapter {
    private static final String NO_DEFAULT = "\n\t\t\n\t\t\n\uE000\uE001\uE002\n\t\t\t\t\n";

    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       HandlerMethod handler) throws Exception {
        Method method = handler.getMethod();
        method.setAccessible(true);
        Object[] args = resolveArguments(method, request, response);
        Object result = method.invoke(handler.getBean(), args);
        if (result != null && !response.isCommitted()) {
            response.setContentType("text/plain;charset=UTF-8");
            response.getWriter().write(result.toString());
            response.getWriter().flush();
        }
    }

    private Object[] resolveArguments(Method method, HttpServletRequest request,
                                       HttpServletResponse response) {
        Parameter[] params = method.getParameters();
        Object[] args = new Object[params.length];
        @SuppressWarnings("unchecked")
        Map<String, String> pathVariables = (Map<String, String>) request.getAttribute(
                HandlerMapping.PATH_VARIABLES_ATTRIBUTE);
        if (pathVariables == null) pathVariables = Collections.emptyMap();
        for (int i = 0; i < params.length; i++) {
            args[i] = resolveArgument(params[i], request, response, pathVariables);
        }
        return args;
    }

    private Object resolveArgument(Parameter param, HttpServletRequest request,
                                    HttpServletResponse response,
                                    Map<String, String> pathVariables) {
        Class<?> type = param.getType();
        if (HttpServletRequest.class.isAssignableFrom(type)) return request;
        if (HttpServletResponse.class.isAssignableFrom(type)) return response;

        PathVariable pathVar = param.getAnnotation(PathVariable.class);
        if (pathVar != null) {
            String name = pathVar.value().isEmpty() ? param.getName() : pathVar.value();
            return convertValue(pathVariables.get(name), type);
        }

        RequestParam reqParam = param.getAnnotation(RequestParam.class);
        if (reqParam != null) {
            String name = reqParam.value().isEmpty() ? param.getName() : reqParam.value();
            String value = request.getParameter(name);
            if (value == null) {
                String defaultValue = reqParam.defaultValue();
                if (!NO_DEFAULT.equals(defaultValue)) {
                    value = defaultValue;
                } else if (reqParam.required()) {
                    throw new IllegalArgumentException(
                            "Required request parameter '" + name + "' is not present");
                }
            }
            return convertValue(value, type);
        }
        return null;
    }

    static Object convertValue(String value, Class<?> targetType) {
        if (value == null) {
            if (targetType.isPrimitive()) return getDefaultPrimitiveValue(targetType);
            return null;
        }
        if (targetType == String.class) return value;
        if (targetType == int.class || targetType == Integer.class) return Integer.parseInt(value);
        if (targetType == long.class || targetType == Long.class) return Long.parseLong(value);
        if (targetType == boolean.class || targetType == Boolean.class) return Boolean.parseBoolean(value);
        if (targetType == double.class || targetType == Double.class) return Double.parseDouble(value);
        return value;
    }

    private static Object getDefaultPrimitiveValue(Class<?> type) {
        if (type == int.class) return 0;
        if (type == long.class) return 0L;
        if (type == boolean.class) return false;
        if (type == double.class) return 0.0;
        return null;
    }
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/servlet/DispatcherServlet.java`

```java
package com.iris.framework.web.servlet;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import com.iris.framework.context.ApplicationContext;

public class DispatcherServlet extends HttpServlet {
    private final ApplicationContext applicationContext;
    private HandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    public DispatcherServlet(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Override
    public void init() throws ServletException {
        this.handlerMapping = new RequestMappingHandlerMapping(applicationContext);
        this.handlerAdapter = new RequestMappingHandlerAdapter();
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            if (ex instanceof ServletException se) throw se;
            if (ex instanceof IOException ioe) throw ioe;
            throw new ServletException("Request processing failed", ex);
        }
    }

    private void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerMethod handler = handlerMapping.getHandler(request);
        if (handler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    "No handler found for " + request.getMethod() + " " + request.getRequestURI());
            return;
        }
        if (!handlerAdapter.supports(handler)) {
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
                    "No adapter for handler: " + handler);
            return;
        }
        handlerAdapter.handle(request, response, handler);
    }

    public ApplicationContext getApplicationContext() { return applicationContext; }
}
```

#### [MODIFIED] `iris-boot-core/src/main/java/com/iris/boot/web/server/servlet/ServletWebServerFactory.java`

```java
package com.iris.boot.web.server.servlet;

import jakarta.servlet.Servlet;
import com.iris.boot.web.server.WebServer;

@FunctionalInterface
public interface ServletWebServerFactory {
    WebServer getWebServer(Servlet... servlets);
}
```

#### [MODIFIED] `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatServletWebServerFactory.java`

Key change — `getWebServer()` and `prepareContext()` now accept and register servlets:

```java
@Override
public WebServer getWebServer(Servlet... servlets) {
    Tomcat tomcat = createTomcat();
    prepareContext(tomcat, servlets);
    return new TomcatWebServer(tomcat);
}

private void prepareContext(Tomcat tomcat, Servlet... servlets) {
    Context context = tomcat.addContext("", createTempDir("tomcat-docbase").getAbsolutePath());
    context.setParentClassLoader(getClass().getClassLoader());
    for (Servlet servlet : servlets) {
        String servletName = servlet.getClass().getSimpleName();
        Wrapper wrapper = Tomcat.addServlet(context, servletName, servlet);
        wrapper.setLoadOnStartup(1);
        context.addServletMappingDecoded("/", servletName);
    }
}
```

#### [MODIFIED] `iris-boot-core/src/main/java/com/iris/boot/web/context/ServletWebServerApplicationContext.java`

Key change — `createWebServer()` creates and registers a DispatcherServlet:

```java
private void createWebServer() {
    ServletWebServerFactory factory = getWebServerFactory();
    DispatcherServlet dispatcherServlet = new DispatcherServlet(this);
    this.webServer = factory.getWebServer(dispatcherServlet);
}
```

#### [MODIFIED] `build.gradle` (root)

```groovy
tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    options.compilerArgs += ['-parameters']
}
```

### Test Code

#### [NEW] `iris-framework/src/test/java/com/iris/framework/web/servlet/RequestMappingInfoTest.java`

Tests path pattern matching: exact paths, path variables, HTTP method filtering, root path, trailing slash normalization.

#### [NEW] `iris-framework/src/test/java/com/iris/framework/web/servlet/RequestMappingHandlerMappingTest.java`

Tests controller scanning: @GetMapping/@PostMapping detection, path variable extraction, class-level @RequestMapping prefix, @RequestMapping with explicit method, multiple controllers.

#### [NEW] `iris-framework/src/test/java/com/iris/framework/web/servlet/RequestMappingHandlerAdapterTest.java`

Tests parameter resolution: @RequestParam, @PathVariable, type conversion (String→int), default values, required param validation, HttpServletRequest/Response injection, void return handling.

#### [NEW] `iris-framework/src/test/java/com/iris/framework/web/servlet/DispatcherServletTest.java`

Tests full dispatch with mock objects: GET/POST dispatch, 404 for unknown paths, path variable resolution, request param resolution, class-level prefix, method mismatch rejection.

#### [NEW] `iris-boot-core/src/test/java/com/iris/boot/integration/DispatcherServletMvcIntegrationTest.java`

End-to-end integration test with embedded Tomcat and Java HttpClient: GET/POST over HTTP, path variables, query params, default params, class-level prefix, nested path variables.

---

## Summary

| What | File(s) |
|---|---|
| **Annotations** | `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@RequestParam`, `@PathVariable`, `RequestMethod` |
| **MVC core** | `HandlerMapping`, `HandlerAdapter`, `HandlerMethod`, `RequestMappingInfo` |
| **Implementations** | `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, `DispatcherServlet` |
| **Modified** | `ServletWebServerFactory`, `TomcatServletWebServerFactory`, `ServletWebServerApplicationContext`, `build.gradle` |
| **Tests** | 4 unit test classes + 1 integration test class (254 total tests pass) |
| **Pattern** | Front Controller → Strategy (HandlerMapping, HandlerAdapter) → Reflection-based invocation |

**Next chapter:** [Chapter 11: JSON Response Body](ch11_json_response_body.md) — Add `@ResponseBody` and `@RestController` annotations, integrate Jackson's `ObjectMapper` for automatic JSON serialization, and enhance the `HandlerAdapter` to detect `@ResponseBody` and write `Content-Type: application/json`.
