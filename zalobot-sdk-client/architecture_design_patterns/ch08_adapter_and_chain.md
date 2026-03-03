# Chapter 8: Adapter & Chain of Responsibility — The Request Pipeline

## What You'll Learn
- How the **Adapter** pattern unifies different handler types behind one interface
- How **Chain of Responsibility** enables interceptor pipelines
- How `DispatcherServlet` orchestrates these patterns for HTTP request handling
- Why combining Adapter + Chain + Strategy creates a powerful extensible architecture

---

## 8.1 The Problem: Multiple Handler Types

In a web framework, handlers come in different shapes:

```java
// Style 1: Controller interface
public class OldController implements Controller {
    ModelAndView handleRequest(Request req, Response resp) { ... }
}

// Style 2: Annotated methods
@GetMapping("/users/{id}")
public User getUser(@PathVariable long id) { ... }

// Style 3: Functional handlers
RouterFunction<ServerResponse> route = route()
    .GET("/users/{id}", req -> ok().body(findUser(req)))
    .build();
```

The dispatcher needs to call all of these, but they have different method signatures. Direct `if/else` checking the handler type is fragile and violates Open-Closed.

## 8.2 Adapter Pattern: Normalize Different Interfaces

```
  Dispatcher                     Adapters                   Handlers
     │                             │                          │
     │  "Can you handle this?"     │                          │
     │────────────────────────────►│ supports(handler)?       │
     │                             │                          │
     │  "Handle this request"      │                          │
     │────────────────────────────►│ handle(req, resp, handler)│
     │                             │─────────────────────────►│
     │                             │                          │ actual execution
     │                             │◄─────────────────────────│
     │◄────────────────────────────│ returns ModelAndView     │
     │                             │                          │
```

> ★ **Insight** ─────────────────────────────────────
> - Adapter vs Strategy: Both involve pluggable implementations, but Adapter's purpose is **normalization** (make incompatible interfaces work together), while Strategy's purpose is **interchangeability** (swap algorithms). `HandlerAdapter` is truly an Adapter — it translates different handler APIs to a common interface.
> - Spring's Adapter pattern here follows the **Object Adapter** variant (composition, not inheritance). Each adapter holds no state — it just knows how to delegate to a specific handler type.
> - **When to use Adapter:** When you need to integrate third-party code or legacy interfaces that you can't modify. The adapter sits between your expected interface and the actual implementation.
> ─────────────────────────────────────────────────────

## 8.3 Chain of Responsibility: Interceptor Pipeline

Before and after the handler executes, interceptors process the request:

```
  Request
    │
    ▼
  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │ SecurityFilter   │────►│ LoggingFilter    │────►│ CompressionFilter│
  │                 │     │                 │     │                 │
  │ preHandle()     │     │ preHandle()     │     │ preHandle()     │
  │ Can reject! ✕   │     │ Always passes ✓ │     │ Always passes ✓ │
  └─────────────────┘     └─────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
                                                  ┌─────────────┐
                                                  │   Handler    │
                                                  │  executes    │
                                                  └──────┬──────┘
                                                         │
    ┌─────────────────┐     ┌─────────────────┐         │
    │ CompressionFilter│◄───│ LoggingFilter    │◄────────┘
    │ postHandle()    │     │ postHandle()    │
    │ (reverse order) │     │ (reverse order) │
    └─────────────────┘     └─────────────────┘
```

## 8.4 Building the Simplified Web Framework

### Step 1: Request/Response Abstractions

```java
// === SimpleHttpRequest.java ===
import java.util.HashMap;
import java.util.Map;

public class SimpleHttpRequest {
    private final String method;  // GET, POST, etc.
    private final String path;    // /users/42
    private final Map<String, String> params = new HashMap<>();

    public SimpleHttpRequest(String method, String path) {
        this.method = method;
        this.path = path;
    }

    public String getMethod() { return method; }
    public String getPath() { return path; }

    public void setParam(String key, String value) { params.put(key, value); }
    public String getParam(String key) { return params.get(key); }
}
```

```java
// === SimpleHttpResponse.java ===
public class SimpleHttpResponse {
    private int status = 200;
    private String body = "";

    public int getStatus() { return status; }
    public void setStatus(int status) { this.status = status; }

    public String getBody() { return body; }
    public void setBody(String body) { this.body = body; }
}
```

```java
// === SimpleModelAndView.java ===
import java.util.HashMap;
import java.util.Map;

public class SimpleModelAndView {
    private String viewName;
    private final Map<String, Object> model = new HashMap<>();

    public SimpleModelAndView(String viewName) { this.viewName = viewName; }

    public String getViewName() { return viewName; }
    public Map<String, Object> getModel() { return model; }

    public SimpleModelAndView addAttribute(String key, Object value) {
        model.put(key, value);
        return this;
    }
}
```

### Step 2: Handler Adapter (Adapter Pattern)

```java
// === SimpleHandlerAdapter.java ===
/**
 * Adapter pattern: normalizes different handler types to one interface.
 * Each adapter knows how to invoke a specific handler type.
 */
public interface SimpleHandlerAdapter {
    /**
     * Does this adapter support the given handler?
     */
    boolean supports(Object handler);

    /**
     * Invoke the handler and return a ModelAndView.
     */
    SimpleModelAndView handle(SimpleHttpRequest request, SimpleHttpResponse response,
                               Object handler) throws Exception;
}
```

```java
// === ControllerHandlerAdapter.java ===
/**
 * Adapter for handlers that implement the SimpleController interface.
 */
public class ControllerHandlerAdapter implements SimpleHandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof SimpleController;
    }

    @Override
    public SimpleModelAndView handle(SimpleHttpRequest request, SimpleHttpResponse response,
                                      Object handler) throws Exception {
        return ((SimpleController) handler).handleRequest(request, response);
    }
}

// === SimpleController.java ===
public interface SimpleController {
    SimpleModelAndView handleRequest(SimpleHttpRequest request, SimpleHttpResponse response)
        throws Exception;
}
```

```java
// === FunctionalHandlerAdapter.java ===
import java.util.function.BiFunction;

/**
 * Adapter for lambda/functional handlers.
 */
public class FunctionalHandlerAdapter implements SimpleHandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof BiFunction;
    }

    @Override
    @SuppressWarnings("unchecked")
    public SimpleModelAndView handle(SimpleHttpRequest request, SimpleHttpResponse response,
                                      Object handler) throws Exception {
        BiFunction<SimpleHttpRequest, SimpleHttpResponse, SimpleModelAndView> fn =
            (BiFunction<SimpleHttpRequest, SimpleHttpResponse, SimpleModelAndView>) handler;
        return fn.apply(request, response);
    }
}
```

### Step 3: Handler Mapping (Strategy for URL Matching)

```java
// === SimpleHandlerMapping.java ===
/**
 * Strategy for mapping requests to handlers.
 */
public interface SimpleHandlerMapping {
    /**
     * Find a handler for the given request.
     * Returns null if no match.
     */
    Object getHandler(SimpleHttpRequest request);
}
```

```java
// === SimpleUrlHandlerMapping.java ===
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Maps URL paths to handler objects.
 */
public class SimpleUrlHandlerMapping implements SimpleHandlerMapping {

    private final Map<String, Object> urlMap = new LinkedHashMap<>();

    public void registerHandler(String path, Object handler) {
        urlMap.put(path, handler);
    }

    @Override
    public Object getHandler(SimpleHttpRequest request) {
        return urlMap.get(request.getPath());
    }
}
```

### Step 4: Handler Interceptor (Chain of Responsibility)

```java
// === SimpleHandlerInterceptor.java ===
/**
 * Chain of Responsibility: each interceptor can process or reject the request.
 */
public interface SimpleHandlerInterceptor {
    /**
     * Called before the handler executes.
     * @return true to continue, false to abort the chain
     */
    default boolean preHandle(SimpleHttpRequest request, SimpleHttpResponse response,
                               Object handler) throws Exception {
        return true;
    }

    /**
     * Called after the handler executes (but before view rendering).
     */
    default void postHandle(SimpleHttpRequest request, SimpleHttpResponse response,
                             Object handler, SimpleModelAndView mv) throws Exception {
    }

    /**
     * Called after everything completes (even on error). Always called.
     */
    default void afterCompletion(SimpleHttpRequest request, SimpleHttpResponse response,
                                  Object handler, Exception ex) throws Exception {
    }
}
```

```java
// === SimpleHandlerExecutionChain.java ===
import java.util.ArrayList;
import java.util.List;

/**
 * Bundles a handler with its interceptor chain.
 * Manages the chain execution lifecycle.
 */
public class SimpleHandlerExecutionChain {

    private final Object handler;
    private final List<SimpleHandlerInterceptor> interceptors;
    private int interceptorIndex = -1;  // tracks which interceptors ran (for afterCompletion)

    public SimpleHandlerExecutionChain(Object handler, List<SimpleHandlerInterceptor> interceptors) {
        this.handler = handler;
        this.interceptors = interceptors != null ? interceptors : new ArrayList<>();
    }

    public Object getHandler() { return handler; }

    /**
     * Apply preHandle in forward order. Any interceptor can abort.
     */
    public boolean applyPreHandle(SimpleHttpRequest request, SimpleHttpResponse response)
            throws Exception {
        for (int i = 0; i < interceptors.size(); i++) {
            SimpleHandlerInterceptor interceptor = interceptors.get(i);
            if (!interceptor.preHandle(request, response, handler)) {
                // Trigger afterCompletion for already-applied interceptors
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
        return true;
    }

    /**
     * Apply postHandle in REVERSE order.
     */
    public void applyPostHandle(SimpleHttpRequest request, SimpleHttpResponse response,
                                 SimpleModelAndView mv) throws Exception {
        for (int i = interceptors.size() - 1; i >= 0; i--) {
            interceptors.get(i).postHandle(request, response, handler, mv);
        }
    }

    /**
     * Always called — even on error. Reverse order, only for interceptors that ran.
     */
    public void triggerAfterCompletion(SimpleHttpRequest request, SimpleHttpResponse response,
                                        Exception ex) throws Exception {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            interceptors.get(i).afterCompletion(request, response, handler, ex);
        }
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - The `interceptorIndex` tracking is critical for correctness: if interceptor #2 (of 5) rejects the request, only interceptors #0 and #1 should receive `afterCompletion()`. Real Spring does exactly this at `HandlerExecutionChain.java:48`.
> - `postHandle()` runs in REVERSE order (like unwinding a stack), while `preHandle()` runs in FORWARD order. This ensures symmetric wrapping: the first interceptor to wrap is the last to unwrap.
> - This is NOT exactly the GoF Chain of Responsibility (where only ONE handler processes). Here, ALL interceptors run unless one rejects. It's closer to a **Filter Chain** or **Pipeline** pattern.
> ─────────────────────────────────────────────────────

### Step 5: The Dispatcher (Facade over everything)

```java
// === SimpleDispatcherServlet.java ===
import java.util.ArrayList;
import java.util.List;

/**
 * Front Controller + Facade: orchestrates the entire request pipeline.
 * Combines HandlerMapping (Strategy), HandlerAdapter (Adapter),
 * HandlerInterceptor (Chain of Responsibility).
 */
public class SimpleDispatcherServlet {

    private final List<SimpleHandlerMapping> handlerMappings = new ArrayList<>();
    private final List<SimpleHandlerAdapter> handlerAdapters = new ArrayList<>();
    private final List<SimpleHandlerInterceptor> interceptors = new ArrayList<>();

    public void addHandlerMapping(SimpleHandlerMapping mapping) {
        handlerMappings.add(mapping);
    }

    public void addHandlerAdapter(SimpleHandlerAdapter adapter) {
        handlerAdapters.add(adapter);
    }

    public void addInterceptor(SimpleHandlerInterceptor interceptor) {
        interceptors.add(interceptor);
    }

    /**
     * The core dispatch algorithm — simplified DispatcherServlet.doDispatch().
     */
    public void doDispatch(SimpleHttpRequest request, SimpleHttpResponse response) {
        SimpleHandlerExecutionChain chain = null;
        Exception dispatchException = null;
        SimpleModelAndView mv = null;

        try {
            // ① Find the handler (iterate HandlerMappings — Chain of Responsibility)
            Object handler = null;
            for (SimpleHandlerMapping mapping : handlerMappings) {
                handler = mapping.getHandler(request);
                if (handler != null) break;  // first match wins
            }

            if (handler == null) {
                response.setStatus(404);
                response.setBody("No handler found for: " + request.getPath());
                return;
            }

            // ② Build execution chain (handler + interceptors)
            chain = new SimpleHandlerExecutionChain(handler, new ArrayList<>(interceptors));

            // ③ Apply pre-handle interceptors
            if (!chain.applyPreHandle(request, response)) {
                return;  // interceptor rejected the request
            }

            // ④ Find the right adapter for this handler type
            SimpleHandlerAdapter adapter = getHandlerAdapter(handler);

            // ⑤ Invoke handler through adapter
            mv = adapter.handle(request, response, handler);

            // ⑥ Apply post-handle interceptors (reverse order)
            chain.applyPostHandle(request, response, mv);

            // ⑦ Render the result
            if (mv != null) {
                response.setBody("View: " + mv.getViewName() + ", Model: " + mv.getModel());
            }

        } catch (Exception ex) {
            dispatchException = ex;
            response.setStatus(500);
            response.setBody("Error: " + ex.getMessage());
        } finally {
            // ⑧ Always trigger afterCompletion
            if (chain != null) {
                try {
                    chain.triggerAfterCompletion(request, response, dispatchException);
                } catch (Exception ignored) {}
            }
        }
    }

    private SimpleHandlerAdapter getHandlerAdapter(Object handler) {
        for (SimpleHandlerAdapter adapter : handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
        throw new RuntimeException("No adapter for handler: " + handler.getClass().getName());
    }
}
```

## 8.5 Usage

```java
// === SecurityInterceptor.java ===
public class SecurityInterceptor implements SimpleHandlerInterceptor {
    @Override
    public boolean preHandle(SimpleHttpRequest request, SimpleHttpResponse response,
                              Object handler) {
        String token = request.getParam("token");
        if (token == null || !token.equals("valid")) {
            response.setStatus(401);
            response.setBody("Unauthorized");
            System.out.println("[SECURITY] Blocked: " + request.getPath());
            return false;  // abort chain!
        }
        System.out.println("[SECURITY] Allowed: " + request.getPath());
        return true;
    }
}
```

```java
// === RequestLogInterceptor.java ===
public class RequestLogInterceptor implements SimpleHandlerInterceptor {
    @Override
    public boolean preHandle(SimpleHttpRequest request, SimpleHttpResponse response,
                              Object handler) {
        System.out.println("[LOG] " + request.getMethod() + " " + request.getPath());
        return true;
    }

    @Override
    public void afterCompletion(SimpleHttpRequest request, SimpleHttpResponse response,
                                 Object handler, Exception ex) {
        System.out.println("[LOG] Completed: " + response.getStatus());
    }
}
```

```java
// === UserController.java ===
public class UserController implements SimpleController {
    @Override
    public SimpleModelAndView handleRequest(SimpleHttpRequest request,
                                             SimpleHttpResponse response) {
        return new SimpleModelAndView("user-detail")
            .addAttribute("name", "Alice")
            .addAttribute("path", request.getPath());
    }
}
```

```java
// === AdapterChainDemo.java ===
import java.util.function.BiFunction;

public class AdapterChainDemo {
    public static void main(String[] args) {
        // Set up dispatcher
        SimpleDispatcherServlet dispatcher = new SimpleDispatcherServlet();

        // Register handler mappings
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.registerHandler("/users", new UserController());
        mapping.registerHandler("/health", (BiFunction<SimpleHttpRequest, SimpleHttpResponse, SimpleModelAndView>)
            (req, resp) -> new SimpleModelAndView("health").addAttribute("status", "UP"));
        dispatcher.addHandlerMapping(mapping);

        // Register adapters for different handler types
        dispatcher.addHandlerAdapter(new ControllerHandlerAdapter());
        dispatcher.addHandlerAdapter(new FunctionalHandlerAdapter());

        // Register interceptors
        dispatcher.addInterceptor(new RequestLogInterceptor());
        dispatcher.addInterceptor(new SecurityInterceptor());

        // ── Scenario 1: Valid request ──
        System.out.println("=== Valid Request ===");
        SimpleHttpRequest req1 = new SimpleHttpRequest("GET", "/users");
        req1.setParam("token", "valid");
        SimpleHttpResponse resp1 = new SimpleHttpResponse();
        dispatcher.doDispatch(req1, resp1);
        System.out.println("Response: " + resp1.getBody());

        // ── Scenario 2: Unauthorized ──
        System.out.println("\n=== Unauthorized ===");
        SimpleHttpRequest req2 = new SimpleHttpRequest("GET", "/users");
        SimpleHttpResponse resp2 = new SimpleHttpResponse();
        dispatcher.doDispatch(req2, resp2);
        System.out.println("Response: " + resp2.getStatus() + " " + resp2.getBody());

        // ── Scenario 3: Functional handler ──
        System.out.println("\n=== Health Check ===");
        SimpleHttpRequest req3 = new SimpleHttpRequest("GET", "/health");
        req3.setParam("token", "valid");
        SimpleHttpResponse resp3 = new SimpleHttpResponse();
        dispatcher.doDispatch(req3, resp3);
        System.out.println("Response: " + resp3.getBody());
    }
}
```

**Output:**
```
=== Valid Request ===
[LOG] GET /users
[SECURITY] Allowed: /users
Response: View: user-detail, Model: {name=Alice, path=/users}
[LOG] Completed: 200

=== Unauthorized ===
[LOG] GET /users
[SECURITY] Blocked: /users
[LOG] Completed: 401
Response: 401 Unauthorized

=== Health Check ===
[LOG] GET /health
[SECURITY] Allowed: /health
Response: View: health, Model: {status=UP}
[LOG] Completed: 200
```

## 8.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleDispatcherServlet.doDispatch()` | `DispatcherServlet.doDispatch()` | `:935-1004` |
| `SimpleHandlerAdapter` | `HandlerAdapter` | `:62-76` |
| `SimpleHandlerMapping` | `HandlerMapping` | `:160-179` |
| `SimpleHandlerInterceptor` | `HandlerInterceptor` | `:102-156` |
| `SimpleHandlerExecutionChain` | `HandlerExecutionChain` | `:44-164` |
| `ControllerHandlerAdapter` | `SimpleControllerHandlerAdapter` | `spring-webmvc` |
| `SimpleUrlHandlerMapping` | `SimpleUrlHandlerMapping` | `spring-webmvc` |
| `SecurityInterceptor` | Custom `HandlerInterceptor` | User-defined |

**What real Spring adds:**
- `RequestMappingHandlerMapping` for `@GetMapping` / `@PostMapping` annotation scanning
- `RequestMappingHandlerAdapter` for invoking annotated controller methods with argument resolution
- `HandlerMethodArgumentResolver` composite for `@PathVariable`, `@RequestBody`, etc.
- `HandlerMethodReturnValueHandler` for converting return types to responses
- Async support via `WebAsyncManager`
- View resolution via `ViewResolver` chain
- Exception handling via `HandlerExceptionResolver`

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Adapter** | Normalize different handler types to one `handle()` interface |
| **Chain of Responsibility** | Interceptors process request in order; any can reject |
| **Handler Mapping** | Strategy for URL → handler resolution |
| **Execution Chain** | Bundles handler with interceptors; manages lifecycle |
| **Front Controller** | DispatcherServlet is the single entry point for all requests |

**Next: Chapter 9** — We build `SimpleBeanDefinitionBuilder` and composites using **Builder** and **Composite** patterns.
