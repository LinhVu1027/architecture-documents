# Chapter 4: Handler Method Invocation

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `SimpleDispatcherServlet.doDispatch()` finds the handler AND invokes it AND writes the response — all in one method with inline `Method.invoke()` | Handler invocation is hardwired into the dispatcher. There's no way to change how handlers are called (e.g., add argument resolution, support different handler types) without modifying `doDispatch()` itself | Extract invocation into a `HandlerAdapter` strategy that the dispatcher delegates to — separating "find the handler" from "call the handler" |

---

## 4.1 The Integration Point

The integration point is `SimpleDispatcherServlet.doDispatch()` — the same central dispatch method from ch03. Right now it contains inline reflection invocation and response writing. We need to replace that with a delegation to `HandlerAdapter`.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Replace inline `invokeHandler()` + response writing with `HandlerAdapter.handle()` delegation

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

    // Step 2: Find a HandlerAdapter that supports this handler (ch04)
    HandlerAdapter adapter = getHandlerAdapter(handler);

    // Step 3: Delegate invocation + response writing to the adapter (ch04)
    adapter.handle(request, response, handler);
}
```

Compare this with the previous version from ch03:

```java
// BEFORE (ch03) — invocation baked into the dispatcher:
Object result = invokeHandler(handler);
response.setContentType("text/plain");
response.setCharacterEncoding("UTF-8");
if (result != null) {
    response.getWriter().write(result.toString());
}
response.getWriter().flush();

// AFTER (ch04) — delegated to an adapter:
HandlerAdapter adapter = getHandlerAdapter(handler);
adapter.handle(request, response, handler);
```

Two key decisions here:

1. **The adapter takes raw `Object handler`, not `HandlerMethod`.** This mirrors the real framework where `HandlerAdapter.supports(Object)` can accept any handler type — the dispatcher doesn't know or care what the handler is. This is what makes the pattern extensible.

2. **`getHandlerAdapter()` throws `ServletException` if no adapter matches**, just like real Spring's `DispatcherServlet.getHandlerAdapter()` (line 1185). This fails fast rather than silently producing a null.

This connects **DispatcherServlet** (request lifecycle) to **HandlerAdapter** (invocation strategy). To make it work, we need to build:
- **`HandlerAdapter` interface** — the strategy contract with `supports()` and `handle()`
- **`SimpleHandlerAdapter`** — the concrete adapter that invokes `HandlerMethod` via reflection and writes the response
- **`initHandlerAdapter()`** — initialization in `initStrategies()`
- **`getHandlerAdapter()`** — lookup method that matches handler to adapter

We also need to add the new `initHandlerAdapter()` call to `initStrategies()`:

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Add `handlerAdapter` field, `initHandlerAdapter()` call, and `getHandlerAdapter()` method

```java
private HandlerAdapter handlerAdapter;

protected void initStrategies() {
    initHandlerMapping();
    initHandlerAdapter();
    // Future features will initialize:
    // - ExceptionResolvers (ch12)
    // - ViewResolvers (ch13)
}

private void initHandlerAdapter() {
    handlerAdapter = new SimpleHandlerAdapter();
}

private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (handlerAdapter != null && handlerAdapter.supports(handler)) {
        return handlerAdapter;
    }
    throw new ServletException(
            "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                    + "configuration include a HandlerAdapter that supports this handler?");
}
```

And remove the old `invokeHandler()` method entirely — its logic now lives in `SimpleHandlerAdapter`.

## 4.2 The HandlerAdapter Interface

The strategy contract that decouples the dispatcher from handler invocation.

**New file:** `src/main/java/com/simplespringmvc/adapter/HandlerAdapter.java`

```java
package com.simplespringmvc.adapter;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public interface HandlerAdapter {

    /**
     * Can this adapter handle the given handler?
     */
    boolean supports(Object handler);

    /**
     * Invoke the handler to process the request.
     */
    void handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception;
}
```

## 4.3 The SimpleHandlerAdapter

The concrete adapter that knows how to invoke `HandlerMethod` instances via reflection and write the result to the response.

**New file:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;

public class SimpleHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        Object result = invokeHandlerMethod(handlerMethod);
        writeResponse(response, result);
    }

    private Object invokeHandlerMethod(HandlerMethod handlerMethod) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean());
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    private void writeResponse(HttpServletResponse response, Object result) throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        if (result != null) {
            response.getWriter().write(result.toString());
        }
        response.getWriter().flush();
    }
}
```

Notice how `handle()` splits into two clear phases: **invoke** and **write**. This separation maps directly to what the real framework does with `InvocableHandlerMethod.invokeForRequest()` and `HandlerMethodReturnValueHandler.handleReturnValue()`. In ch05 and ch06, we'll expand each phase independently — argument resolvers before invocation, return value handlers after.

## 4.4 Try It Yourself

<details>
<summary>Challenge: What happens if you register a handler that isn't a HandlerMethod?</summary>

Try modifying the test to register a plain `String` as a handler and call `getHandlerAdapter()` on it. What should happen?

The answer: `getHandlerAdapter()` iterates all adapters and calls `supports()`. Since `SimpleHandlerAdapter.supports()` checks `handler instanceof HandlerMethod`, it returns `false` for a `String`. No adapter matches, so `getHandlerAdapter()` throws `ServletException`. This is exactly what the real DispatcherServlet does at line 1193:

```java
throw new ServletException("No adapter for handler [" + handler +
        "]: The DispatcherServlet configuration needs to include a " +
        "HandlerAdapter that supports this handler");
```

</details>

<details>
<summary>Challenge: Implement a second HandlerAdapter for a different handler type</summary>

Create a `FunctionHandlerAdapter` that supports `java.util.function.Function<HttpServletRequest, String>`:

```java
public class FunctionHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return handler instanceof Function;
    }

    @Override
    @SuppressWarnings("unchecked")
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       Object handler) throws Exception {
        Function<HttpServletRequest, String> function =
                (Function<HttpServletRequest, String>) handler;
        String result = function.apply(request);
        response.setContentType("text/plain");
        response.getWriter().write(result);
        response.getWriter().flush();
    }
}
```

This illustrates why `HandlerAdapter` exists — you can support entirely new handler types without touching the dispatcher. You'd just need to register the adapter and register a `Function` handler in the mapping.

</details>

## 4.5 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/adapter/SimpleHandlerAdapterTest.java`

```java
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
}

@Test
@DisplayName("should invoke handler method and write result to response")
void shouldInvokeHandlerAndWriteResult() throws Exception {
    // ... creates HandlerMethod, calls adapter.handle(), checks response body
    assertThat(responseBody.toString()).isEqualTo("hello world");
}

@Test
@DisplayName("should unwrap InvocationTargetException and throw cause")
void shouldUnwrapInvocationTargetException() throws Exception {
    // ... handler that throws IllegalArgumentException
    assertThatThrownBy(() -> adapter.handle(request, response, handlerMethod))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("test error");
}
```

### Updated Tests

**Modified file:** `src/test/java/com/simplespringmvc/servlet/SimpleDispatcherServletTest.java`
**Change:** Added test for handler adapter initialization

```java
@Test
@DisplayName("should initialize handler adapter during init")
void shouldInitializeHandlerAdapter() throws ServletException {
    assertThat(servlet.getHandlerAdapter()).isNotNull();
    assertThat(servlet.getHandlerAdapter()).isInstanceOf(SimpleHandlerAdapter.class);
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/HandlerAdapterIntegrationTest.java`

Tests the full HTTP stack: Tomcat → DispatcherServlet → HandlerMapping → HandlerAdapter → Controller → Response. Verifies handler invocation, content type, error handling (500 for handler exceptions, 404 for missing paths).

**Run:** `./gradlew test` — expected: all tests pass (including ch01–ch03 tests)

---

## 4.6 Why This Works

> ★ **Insight** ─────────────────────────────────────
> **The Adapter pattern makes the framework "indefinitely extensible"**
> - The DispatcherServlet never calls `method.invoke()` directly. It says "I have a handler — who knows how to call it?" and iterates adapters. This means you can add entirely new handler types (Servlets, Functions, WebSocket handlers) by writing a new `HandlerAdapter` — zero changes to the dispatcher.
> - In the real framework, Spring ships three adapters: `RequestMappingHandlerAdapter` (for `@RequestMapping` methods), `SimpleControllerHandlerAdapter` (for the legacy `Controller` interface), and `HttpRequestHandlerAdapter` (for `HttpRequestHandler`). All three coexist because the dispatcher treats them uniformly.
> - This pattern only pays off when you expect multiple handler types. If your framework will only ever have one handler type, a direct call is simpler. Spring needed it because it evolved through multiple handler paradigms.
> ─────────────────────────────────────────────────

> ★ **Insight** ─────────────────────────────────────
> **`InvocationTargetException` unwrapping is critical for exception handling**
> - Java reflection wraps every exception thrown by `method.invoke()` in `InvocationTargetException`. If you don't unwrap it, a controller throwing `IllegalArgumentException` would appear as `InvocationTargetException` to the caller. This breaks `@ExceptionHandler` matching (ch12) because exception resolvers match on the *cause* type, not the wrapper.
> - The real `InvocableHandlerMethod.doInvoke()` (line 260-275) handles four cases: `RuntimeException`, `Error`, checked `Exception`, and `Throwable`. We simplify to just `Exception` vs everything else.
> ─────────────────────────────────────────────────

## 4.7 What We Enhanced

| Aspect | Before (ch03) | Current (ch04) | Real Framework |
|--------|---------------|----------------|----------------|
| Handler invocation | Inline `invokeHandler()` in `doDispatch()` — dispatcher directly calls `method.invoke()` | Delegated to `SimpleHandlerAdapter.handle()` — dispatcher asks "who can invoke this?" | `RequestMappingHandlerAdapter` with `InvocableHandlerMethod`, argument resolvers, return value handlers, `WebDataBinderFactory` (`RequestMappingHandlerAdapter.java:885`) |
| Response writing | Inline in `doDispatch()` — hardcoded text/plain | Encapsulated in `SimpleHandlerAdapter.writeResponse()` — adapter owns the response format | `HandlerMethodReturnValueHandler` chain — `RequestResponseBodyMethodProcessor` for JSON, `ViewNameMethodReturnValueHandler` for views (`RequestMappingHandlerAdapter.java:730-776`) |
| Strategy initialization | Only `initHandlerMapping()` | Added `initHandlerAdapter()` | `initStrategies()` initializes 9 strategy types including `initHandlerAdapters()` (`DispatcherServlet.java:441-445`) |
| Extensibility | Cannot support new handler types without editing `doDispatch()` | New handler types need only a new `HandlerAdapter` implementation | Three built-in adapters + custom adapters via bean registration (`DispatcherServlet.java:552-568`) |

## 4.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `HandlerAdapter` | `HandlerAdapter` | `HandlerAdapter.java:49` | Real version returns `ModelAndView`; ours returns void |
| `SimpleHandlerAdapter` | `RequestMappingHandlerAdapter` | `RequestMappingHandlerAdapter.java:885` | Real version creates `InvocableHandlerMethod`, sets argument resolvers + return value handlers, manages `ModelAndViewContainer` |
| `SimpleHandlerAdapter.invokeHandlerMethod()` | `InvocableHandlerMethod.doInvoke()` | `InvocableHandlerMethod.java:243` | Real version handles Kotlin coroutines, bridged methods, and formats detailed error messages |
| `SimpleHandlerAdapter.writeResponse()` | `HandlerMethodReturnValueHandler` chain | `ServletInvocableHandlerMethod.java:114` | Real version iterates return value handlers — JSON, views, redirect, streaming |
| `getHandlerAdapter()` | `DispatcherServlet.getHandlerAdapter()` | `DispatcherServlet.java:1185` | Real version iterates a `List<HandlerAdapter>` and returns first match |
| `initHandlerAdapter()` | `DispatcherServlet.initHandlerAdapters()` | `DispatcherServlet.java:552` | Real version discovers adapters from `ApplicationContext` via `BeanFactoryUtils.beansOfTypeIncludingAncestors()` |

## 4.9 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerAdapter.java` [NEW]

```java
package com.simplespringmvc.adapter;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface that allows the DispatcherServlet to invoke handlers
 * without knowing their specific type or calling convention.
 *
 * Maps to: {@code org.springframework.web.servlet.HandlerAdapter}
 *
 * The real HandlerAdapter is a Service Provider Interface (SPI) that decouples
 * the DispatcherServlet from any specific handler type. Spring ships three:
 * <ul>
 *   <li>{@code RequestMappingHandlerAdapter} — for @RequestMapping methods</li>
 *   <li>{@code SimpleControllerHandlerAdapter} — for the Controller interface</li>
 *   <li>{@code HttpRequestHandlerAdapter} — for the HttpRequestHandler interface</li>
 * </ul>
 *
 * The DispatcherServlet iterates all registered HandlerAdapters and asks each
 * one {@code supports(handler)?}. The first one that says "yes" handles the
 * request. This pattern lets Spring MVC support new handler types without
 * changing the dispatcher.
 *
 * Simplifications:
 * <ul>
 *   <li>Returns void instead of ModelAndView — ch13 will add view resolution</li>
 *   <li>Takes our HandlerMethod directly, not raw Object — we only have one handler type</li>
 *   <li>No async support (DeferredResult, Callable)</li>
 * </ul>
 */
public interface HandlerAdapter {

    /**
     * Given a handler instance, return whether this adapter can support it.
     *
     * Maps to: {@code HandlerAdapter.supports()} (line 62)
     * Real version takes Object so it can check instanceof for any handler type.
     * We take Object too, for the same reason — the DispatcherServlet shouldn't
     * know the concrete handler type.
     *
     * @param handler the handler to check
     * @return true if this adapter can invoke the handler
     */
    boolean supports(Object handler);

    /**
     * Use the given handler to handle this request.
     *
     * Maps to: {@code HandlerAdapter.handle()} (line 76)
     * Real version returns ModelAndView (null if response was handled directly).
     * We return void for now — ch06 adds @ResponseBody, ch13 adds view resolution.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the handler to invoke (previously validated via supports())
     * @throws Exception if handler invocation fails
     */
    void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;

/**
 * The default HandlerAdapter that knows how to invoke {@link HandlerMethod}
 * instances via reflection.
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
 * We collapse that chain into a single class:
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handle() invokes the method via reflection
 *   3. Writes the String return value directly to the response
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No InvocableHandlerMethod wrapper — we call Method.invoke() directly</li>
 *   <li>No argument resolvers — only supports no-arg methods (ch05 adds @RequestParam)</li>
 *   <li>No return value handlers — always writes toString() as text/plain (ch06 adds @ResponseBody)</li>
 *   <li>No ModelAndView return — response is written directly</li>
 *   <li>No WebDataBinderFactory</li>
 *   <li>No ModelFactory</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

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
     * Invoke the handler method via reflection and write the result to the response.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.handleInternal()} (line 831)
     * → {@code invokeHandlerMethod()} (line 885)
     * → {@code ServletInvocableHandlerMethod.invokeAndHandle()} (line 114)
     * → {@code InvocableHandlerMethod.invokeForRequest()} (line 171)
     * → {@code InvocableHandlerMethod.doInvoke()} (line 243)
     *
     * The real chain creates an InvocableHandlerMethod, configures argument
     * resolvers and return value handlers, then invokes. We do the simplest
     * possible version: call method.invoke(bean) with no arguments, and write
     * the result as plain text.
     *
     * @param request  current HTTP request (unused for now — ch05 will use it for @RequestParam)
     * @param response current HTTP response
     * @param handler  the HandlerMethod to invoke
     */
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // Invoke the controller method via reflection.
        // Maps to: InvocableHandlerMethod.doInvoke() (line 243)
        // Real version handles Kotlin coroutines, bridged methods, etc.
        // We just call Method.invoke().
        Object result = invokeHandlerMethod(handlerMethod);

        // Write the result to the response.
        // Maps to: HandlerMethodReturnValueHandler pattern
        // The real framework uses ReturnValueHandlers to decide how to write
        // the response (JSON, view name, redirect, etc.).
        // We write everything as text/plain for now.
        writeResponse(response, result);
    }

    /**
     * Invoke the handler method and return the result.
     *
     * Maps to: {@code InvocableHandlerMethod.doInvoke()} (line 243)
     * The real version uses getBridgedMethod() for generic type bridge methods,
     * handles Kotlin suspend functions, and formats detailed error messages.
     */
    private Object invokeHandlerMethod(HandlerMethod handlerMethod) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean());
        } catch (InvocationTargetException ex) {
            // Unwrap the real exception from the reflection wrapper.
            // Maps to: InvocableHandlerMethod.doInvoke() (line 260-275)
            // The real version unwraps RuntimeException, Error, and Exception
            // separately so they preserve their original type for
            // @ExceptionHandler matching (ch12).
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
}
```

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

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
    private HandlerAdapter handlerAdapter;

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
     */
    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    /**
     * Initialize all strategy beans from the container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * Real version initializes 9 strategy types. We initialize them
     * incrementally as features are added.
     */
    protected void initStrategies() {
        initHandlerMapping();
        initHandlerAdapter();
        // Future features will initialize:
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
     * Create the handler adapter that knows how to invoke handler methods.
     *
     * Maps to: {@code DispatcherServlet.initHandlerAdapters()} (line 552)
     * Real version finds all HandlerAdapter beans in the ApplicationContext
     * (or falls back to defaults from DispatcherServlet.properties).
     * The default adapters include:
     * <ul>
     *   <li>HttpRequestHandlerAdapter</li>
     *   <li>SimpleControllerHandlerAdapter</li>
     *   <li>RequestMappingHandlerAdapter</li>
     * </ul>
     *
     * We create just one adapter directly — the one that handles our
     * HandlerMethod type via reflection.
     */
    private void initHandlerAdapter() {
        handlerAdapter = new SimpleHandlerAdapter();
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
     *   4. ha = getHandlerAdapter(handler)             ← ch04 ✓
     *   5. ha.handle(request, response, handler)       ← ch04 ✓
     *   6. mappedHandler.applyPostHandle()             ← ch11
     *   7. processDispatchResult(mv, exception)        ← ch12, ch13
     * </pre>
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

        // Step 2: Find a HandlerAdapter that supports this handler (ch04)
        HandlerAdapter adapter = getHandlerAdapter(handler);

        // Step 3: Delegate invocation + response writing to the adapter (ch04)
        adapter.handle(request, response, handler);
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
     * Find a HandlerAdapter that can handle the given handler.
     *
     * Maps to: {@code DispatcherServlet.getHandlerAdapter()} (line 1185)
     * Real version iterates a list of HandlerAdapters and returns the first
     * one where {@code adapter.supports(handler)} is true.
     *
     * We have just one adapter, so we check it directly. The real framework
     * throws ServletException if no adapter supports the handler — we do the same.
     */
    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
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

    /**
     * Provides access to the handler adapter for testing and inspection.
     */
    public HandlerAdapter getHandlerAdapter() {
        return handlerAdapter;
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/adapter/SimpleHandlerAdapterTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

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

    // ─── Test controllers ────────────────────────────────────────────

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

    // ─── supports() ─────────────────────────────────────────────────

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

    // ─── handle() ───────────────────────────────────────────────────

    @Nested
    @DisplayName("handle()")
    class Handle {

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
}
```

#### File: `src/test/java/com/simplespringmvc/servlet/SimpleDispatcherServletTest.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
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
        @DisplayName("should dispatch to matched handler via adapter and return its result")
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
        @DisplayName("should set content type to text/plain via adapter")
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
        @DisplayName("should set character encoding to UTF-8 via adapter")
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

        @Test
        @DisplayName("should initialize handler adapter during init")
        void shouldInitializeHandlerAdapter() throws ServletException {
            assertThat(servlet.getHandlerAdapter()).isNotNull();
            assertThat(servlet.getHandlerAdapter()).isInstanceOf(SimpleHandlerAdapter.class);
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/HandlerAdapterIntegrationTest.java` [NEW]

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
 * Integration tests for the handler adapter feature (ch04).
 *
 * Verifies that the HandlerAdapter pattern correctly invokes handlers
 * and writes responses end-to-end through a real HTTP stack.
 * These tests complement the ch03 integration tests by confirming the
 * adapter sits between DispatcherServlet and handler invocation.
 */
class HandlerAdapterIntegrationTest {

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    static class StatusController {

        @RequestMapping(path = "/status", method = "GET")
        public String status() {
            return "OK";
        }

        @RequestMapping(path = "/version", method = "GET")
        public String version() {
            return "1.0.0";
        }
    }

    @Controller
    static class ErrorController {

        @RequestMapping(path = "/fail", method = "GET")
        public String fail() {
            throw new RuntimeException("handler error");
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new StatusController());
        container.registerBean(new ErrorController());

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

    // ─── Adapter invocation ─────────────────────────────────────────

    @Test
    @DisplayName("should invoke handler via adapter and return result")
    void shouldInvokeHandlerViaAdapter() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/status")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("OK");
    }

    @Test
    @DisplayName("should invoke different handlers for different endpoints")
    void shouldInvokeDifferentHandlers() throws Exception {
        HttpResponse<String> statusResponse = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/status")).GET().build(),
                HttpResponse.BodyHandlers.ofString());
        HttpResponse<String> versionResponse = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/version")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(statusResponse.body()).isEqualTo("OK");
        assertThat(versionResponse.body()).isEqualTo("1.0.0");
    }

    @Test
    @DisplayName("should set content type to text/plain on response")
    void shouldSetContentType() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/status")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.headers().firstValue("Content-Type").orElse(""))
                .contains("text/plain");
    }

    // ─── Error handling ─────────────────────────────────────────────

    @Test
    @DisplayName("should return 500 when handler throws exception")
    void shouldReturn500_WhenHandlerThrows() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/fail")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(500);
    }

    @Test
    @DisplayName("should return 404 for unregistered path")
    void shouldReturn404_WhenPathNotFound() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/nonexistent")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(404);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **HandlerAdapter** | Strategy interface that decouples the DispatcherServlet from handler invocation — the dispatcher asks "who can call this handler?" |
| **supports()** | Type-check method that lets the dispatcher find the right adapter for a handler type |
| **Strategy pattern** | Define a family of algorithms, encapsulate each one, and make them interchangeable — here, "how to invoke a handler" is the algorithm |
| **InvocationTargetException unwrapping** | Java reflection wraps handler exceptions — unwrapping preserves the original type for exception handling |

**Next: Chapter 5 — @RequestParam Argument Resolution** — Our handlers can only be no-arg methods. We'll introduce the argument resolver pattern to extract query parameters from the request and pass them as method arguments — the first step toward `@RequestParam String name`.
