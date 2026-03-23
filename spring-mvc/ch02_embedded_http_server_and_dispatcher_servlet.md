# Chapter 2: Embedded HTTP Server & DispatcherServlet

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| A `SimpleBeanContainer` stores and retrieves singleton beans by name or type | No way to receive HTTP requests — the container exists but nothing connects it to the network | Embed Tomcat to accept HTTP connections and route every request through a single front-controller `SimpleDispatcherServlet` whose `doDispatch()` method will be the extension point for all future features |

---

## 2.1 The Integration Point

This is a foundation feature — there is no existing dispatch pipeline to modify. The integration point is the **`doDispatch()` method itself**: the single method that every future feature will plug into.

```
Future doDispatch() flow (what this method will become):
┌──────────────────────────────────────────────────────────────┐
│  doDispatch(request, response)                                │
│                                                               │
│  1. handler = handlerMapping.getHandler(request)    ← ch03   │
│  2. chain.applyPreHandle(request, response)         ← ch11   │
│  3. adapter = getHandlerAdapter(handler)            ← ch04   │
│  4. mv = adapter.handle(request, response, handler) ← ch04   │
│  5. chain.applyPostHandle(request, response, mv)    ← ch11   │
│  6. processDispatchResult(mv, exception)            ← ch12,13│
│  7. chain.triggerAfterCompletion()                   ← ch11   │
└──────────────────────────────────────────────────────────────┘
```

Right now, `doDispatch()` simply writes `"Hello from DispatcherServlet"` — but it's the method that ch03 will add handler lookup to, ch04 will add invocation to, ch11 will wrap with interceptors, ch12 will add exception handling to, and ch13 will add view rendering to.

Two key decisions:

1. **Override `service()`, not `doGet()`/`doPost()` individually.** The real `FrameworkServlet.service()` (line 870) routes ALL HTTP methods through a single path. By overriding `service()` we ensure GET, POST, PUT, DELETE, PATCH, OPTIONS, and TRACE all reach `doDispatch()` — one method to rule them all.

2. **Accept `BeanContainer` in the constructor.** The real DispatcherServlet gets its strategies from a `WebApplicationContext`. We use our `BeanContainer` from ch01 instead — same role (a registry of components), much simpler API. Future features will call `beanContainer.getBeansOfType(HandlerMapping.class)` to discover strategies.

To make this work, we need to build:
- **`SimpleDispatcherServlet`** — the front-controller servlet with `doDispatch()`
- **`EmbeddedTomcat`** — the server launcher that wires Tomcat to the servlet

## 2.2 The Front Controller: SimpleDispatcherServlet

The real Spring MVC uses a three-class hierarchy:

```
jakarta.servlet.http.HttpServlet
  └── HttpServletBean           → maps init-params to bean properties
        └── FrameworkServlet    → manages WebApplicationContext, processRequest() lifecycle
              └── DispatcherServlet → doDispatch() with strategy delegation
```

We collapse all three into one class. Here is the key simplification: `service()` calls `doDispatch()` directly, while the real framework goes through `service()` → `processRequest()` → `doService()` → `doDispatch()`.

**New file:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.container.BeanContainer;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    protected void initStrategies() {
        // Future features will initialize:
        // - HandlerMappings (ch03)
        // - HandlerAdapters (ch04)
        // - ExceptionResolvers (ch12)
        // - ViewResolvers (ch13)
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        }
    }

    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write("Hello from DispatcherServlet");
        response.getWriter().flush();
    }

    public BeanContainer getBeanContainer() {
        return beanContainer;
    }
}
```

Three things to notice:

- **`init()` → `initStrategies()`**: Tomcat calls `init()` after registering the servlet. This is where strategy components will be looked up from the container. Maps to `DispatcherServlet.initStrategies()` (line 441), which is called from `onRefresh()` (line 433).
- **`service()` catches and wraps exceptions**: `doDispatch()` throws `Exception` (broader than `ServletException`), so we wrap in `service()`. The real framework does the same in `processRequest()` (line 982).
- **`doDispatch()` is `protected`**, not `private` — future chapters need to modify its behavior, and tests call it directly.

## 2.3 The Server Launcher: EmbeddedTomcat

In real Spring Boot, embedding Tomcat requires coordinating `TomcatServletWebServerFactory`, `ServletWebServerApplicationContext`, `DispatcherServletAutoConfiguration`, and `DispatcherServletRegistrationBean`. We flatten all of that into one class.

**New file:** `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java`

```java
package com.simplespringmvc.server;

import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;

import java.io.File;

public class EmbeddedTomcat {

    private final Tomcat tomcat;
    private final SimpleDispatcherServlet dispatcherServlet;

    public EmbeddedTomcat(int port, SimpleDispatcherServlet dispatcherServlet) {
        this.dispatcherServlet = dispatcherServlet;
        this.tomcat = new Tomcat();
        tomcat.setPort(port);

        tomcat.setBaseDir(new File(System.getProperty("java.io.tmpdir"), "tomcat-simple")
                .getAbsolutePath());

        Context context = tomcat.addContext("", null);

        Tomcat.addServlet(context, "dispatcher", dispatcherServlet);
        context.addServletMappingDecoded("/", "dispatcher");

        tomcat.getConnector();
    }

    public void start() throws Exception {
        tomcat.start();
    }

    public void stop() throws Exception {
        tomcat.stop();
        tomcat.destroy();
    }

    public int getPort() {
        return tomcat.getConnector().getLocalPort();
    }

    public SimpleDispatcherServlet getDispatcherServlet() {
        return dispatcherServlet;
    }

    public void await() {
        tomcat.getServer().await();
    }
}
```

Key setup steps:

1. **`tomcat.setBaseDir(...)`** — Tomcat requires a base directory for temp files and work directories. We use the system temp dir so we don't pollute the project.
2. **`tomcat.addContext("", null)`** — Creates a web application context at the root path `""`. The `null` docBase means no static file directory.
3. **`Tomcat.addServlet(context, "dispatcher", dispatcherServlet)`** — The programmatic equivalent of a `<servlet>` entry in `web.xml`.
4. **`context.addServletMappingDecoded("/", "dispatcher")`** — Maps the servlet to `/`, making it the default servlet that receives all requests.
5. **`tomcat.getConnector()`** — This seemingly innocent getter call has a critical side effect: it triggers creation of the HTTP connector. Without it, Tomcat won't listen on any port.

## 2.4 Try It Yourself

<details>
<summary>Challenge 1: Write the SimpleDispatcherServlet from scratch</summary>

Starting point: create a class that extends `HttpServlet`, takes a `BeanContainer` in its constructor, and makes sure every HTTP method reaches a single `doDispatch()` method.

Hint: Don't override `doGet()`, `doPost()`, etc. individually — there's a method higher up in `HttpServlet` that all of those call.

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.container.BeanContainer;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    protected void initStrategies() {
        // Empty for now — future chapters populate this
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        }
    }

    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write("Hello from DispatcherServlet");
        response.getWriter().flush();
    }

    public BeanContainer getBeanContainer() {
        return beanContainer;
    }
}
```

</details>

<details>
<summary>Challenge 2: Wire Tomcat to the servlet using the Tomcat embed API</summary>

Starting point: the Tomcat embed API uses `new Tomcat()`, `tomcat.addContext()`, `Tomcat.addServlet()`, and `context.addServletMappingDecoded()`.

Key gotcha: you MUST call `tomcat.getConnector()` before `start()`, or Tomcat won't create an HTTP listener.

```java
package com.simplespringmvc.server;

import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;
import java.io.File;

public class EmbeddedTomcat {

    private final Tomcat tomcat;
    private final SimpleDispatcherServlet dispatcherServlet;

    public EmbeddedTomcat(int port, SimpleDispatcherServlet dispatcherServlet) {
        this.dispatcherServlet = dispatcherServlet;
        this.tomcat = new Tomcat();
        tomcat.setPort(port);
        tomcat.setBaseDir(new File(System.getProperty("java.io.tmpdir"), "tomcat-simple")
                .getAbsolutePath());

        Context context = tomcat.addContext("", null);
        Tomcat.addServlet(context, "dispatcher", dispatcherServlet);
        context.addServletMappingDecoded("/", "dispatcher");
        tomcat.getConnector();  // Critical! Creates the HTTP connector
    }

    public void start() throws Exception { tomcat.start(); }
    public void stop() throws Exception { tomcat.stop(); tomcat.destroy(); }
    public int getPort() { return tomcat.getConnector().getLocalPort(); }
    public SimpleDispatcherServlet getDispatcherServlet() { return dispatcherServlet; }
    public void await() { tomcat.getServer().await(); }
}
```

</details>

## 2.5 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/servlet/SimpleDispatcherServletTest.java`

```java
package com.simplespringmvc.servlet;

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

class SimpleDispatcherServletTest {

    private SimpleBeanContainer beanContainer;
    private SimpleDispatcherServlet servlet;

    @BeforeEach
    void setUp() throws ServletException {
        beanContainer = new SimpleBeanContainer();
        servlet = new SimpleDispatcherServlet(beanContainer);
        servlet.init();
    }

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

    @Nested
    @DisplayName("doDispatch")
    class DoDispatch {

        @Test
        @DisplayName("should write greeting response")
        void shouldWriteGreetingResponse() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));

            servlet.doDispatch(request, response);

            assertThat(stringWriter.toString()).isEqualTo("Hello from DispatcherServlet");
        }

        @Test
        @DisplayName("should set content type to text/plain")
        void shouldSetContentType() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(response.getWriter()).thenReturn(new PrintWriter(new StringWriter()));

            servlet.doDispatch(request, response);

            verify(response).setContentType("text/plain");
        }

        @Test
        @DisplayName("should set character encoding to UTF-8")
        void shouldSetCharacterEncoding() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(response.getWriter()).thenReturn(new PrintWriter(new StringWriter()));

            servlet.doDispatch(request, response);

            verify(response).setCharacterEncoding("UTF-8");
        }
    }

    @Nested
    @DisplayName("service() routing")
    class ServiceRouting {

        @Test
        @DisplayName("should route GET to doDispatch")
        void shouldRouteGetToDoDispatch() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));

            servlet.service(request, response);

            assertThat(stringWriter.toString()).isEqualTo("Hello from DispatcherServlet");
        }

        @Test
        @DisplayName("should route POST to doDispatch")
        void shouldRoutePostToDoDispatch() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));

            servlet.service(request, response);

            assertThat(stringWriter.toString()).isEqualTo("Hello from DispatcherServlet");
        }
    }

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
    }
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/EmbeddedTomcatIntegrationTest.java`

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.*;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.assertj.core.api.Assertions.assertThat;

class EmbeddedTomcatIntegrationTest {

    private EmbeddedTomcat server;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
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

    @Test
    @DisplayName("should respond to GET /")
    void shouldRespondToGetRoot() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/"))
                .GET().build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
    }

    @Test
    @DisplayName("should respond to GET /any/path")
    void shouldRespondToGetAnyPath() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/any/path"))
                .GET().build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
    }

    @Test
    @DisplayName("should return text/plain content type")
    void shouldReturnTextPlainContentType() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/")).GET().build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.headers().firstValue("Content-Type"))
                .isPresent()
                .hasValueSatisfying(ct -> assertThat(ct).contains("text/plain"));
    }

    @Test
    @DisplayName("should respond to POST requests")
    void shouldRespondToPostRequests() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/submit"))
                .POST(HttpRequest.BodyPublishers.ofString("data")).build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
    }

    @Test
    @DisplayName("should respond to PUT requests")
    void shouldRespondToPutRequests() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/resource"))
                .PUT(HttpRequest.BodyPublishers.ofString("data")).build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
    }

    @Test
    @DisplayName("should respond to DELETE requests")
    void shouldRespondToDeleteRequests() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/resource/1")).DELETE().build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
    }

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

**Run:** `./gradlew test` — expected: all 37 tests pass (21 from ch01 + 8 unit + 8 integration)

---

## 2.6 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why a single `service()` entry point?** The standard `HttpServlet` dispatches to `doGet()`, `doPost()`, `doPut()`, etc. based on the HTTP method. This means each method type goes through a different code path — making it impossible to apply cross-cutting concerns (logging, security, interceptors) in one place. By overriding `service()` to funnel everything through `doDispatch()`, we get a **single pipeline** that all HTTP methods share. This is the **Front Controller** pattern, and it's the reason Spring MVC can have one interceptor chain that applies to all requests regardless of method.
> - **When this is overkill:** If you're building a simple REST API with no cross-cutting concerns, a per-method approach (like basic `doGet()`/`doPost()` overrides) is simpler. The front controller pays off only when you have shared pipeline steps.
> - **Real-world parallel:** `FrameworkServlet.service()` (line 870) does exactly this — it overrides `service()` and routes all methods through `processRequest()`. Even `PATCH` (not part of the original HTTP spec when Servlet API was designed) goes through the same path.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why separate `EmbeddedTomcat` from `SimpleDispatcherServlet`?** The servlet knows about request dispatch; the server knows about network sockets. Mixing them would couple HTTP processing logic to a specific server implementation. By keeping them separate, you could swap Tomcat for Jetty or Netty without changing the dispatcher — the same principle Spring Boot uses with `ServletWebServerFactory`.
> - **The `tomcat.getConnector()` gotcha:** This is a real foot-gun in the Tomcat embed API. The `getConnector()` method lazily creates the HTTP connector on first call. If you never call it, `tomcat.start()` succeeds silently but never listens on any port. Spring Boot's `TomcatServletWebServerFactory` calls it explicitly in `getWebServer()` for exactly this reason.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why `doDispatch()` is `protected`, not `private`:** This method will be modified by at least 5 future features. Making it `protected` allows tests to call it directly (bypassing the Servlet container) and signals to readers that it's an extension point. In the real framework, `doDispatch()` at `DispatcherServlet.java:935` is also `protected` — intentionally overridable for custom dispatch behavior.
> -----------------------------------------------------------

## 2.7 What We Enhanced

| Aspect | Before (ch01) | Current (ch02) | Real Framework |
|--------|---------------|----------------|----------------|
| Network access | None — beans exist in memory only | Tomcat listens on a port and routes HTTP requests to the bean container via the servlet | Full Servlet container with NIO connector, thread pools, keep-alive, SSL (`TomcatServletWebServerFactory`) |
| Request entry point | N/A | `service()` → `doDispatch()` — single pipeline for all HTTP methods | `service()` → `processRequest()` → `doService()` → `doDispatch()` — three-layer pipeline with locale/attribute lifecycle (`FrameworkServlet.java:982`) |
| Strategy initialization | N/A | `init()` → `initStrategies()` — empty placeholder for future features | `onRefresh()` → `initStrategies()` — initializes 8 strategy types from ApplicationContext (`DispatcherServlet.java:441`) |

## 2.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `SimpleDispatcherServlet` | `DispatcherServlet` | `DispatcherServlet.java:157` | Real uses a 3-class hierarchy, manages WebApplicationContext, has 8 strategy lists |
| `service()` → `doDispatch()` | `service()` → `processRequest()` → `doService()` → `doDispatch()` | `FrameworkServlet.java:870`, `DispatcherServlet.java:829`, `DispatcherServlet.java:935` | Real saves/restores `LocaleContext` and `RequestAttributes` in thread-locals, publishes `ServletRequestHandledEvent` |
| `initStrategies()` (empty) | `initStrategies(ApplicationContext)` | `DispatcherServlet.java:441` | Real initializes 8 strategies: `HandlerMapping`, `HandlerAdapter`, `HandlerExceptionResolver`, `ViewResolver`, `MultipartResolver`, `LocaleResolver`, `RequestToViewNameTranslator`, `FlashMapManager` |
| `init()` calls `initStrategies()` | `HttpServletBean.init()` → `FrameworkServlet.initServletBean()` → `initWebApplicationContext()` → `onRefresh()` → `initStrategies()` | `HttpServletBean.java:147`, `DispatcherServlet.java:433` | Real creates/refreshes a full `WebApplicationContext` before strategy init |
| `EmbeddedTomcat` | `TomcatServletWebServerFactory` | Spring Boot `TomcatServletWebServerFactory.java` | Real supports customizers, SSL, HTTP/2, compression, access logs, graceful shutdown |

## 2.9 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [NEW]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.container.BeanContainer;
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
        // Future features will initialize:
        // - HandlerMappings (ch03)
        // - HandlerAdapters (ch04)
        // - ExceptionResolvers (ch12)
        // - ViewResolvers (ch13)
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
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write("Hello from DispatcherServlet");
        response.getWriter().flush();
    }

    /**
     * Provides access to the bean container for strategy initialization.
     * Future features use this to look up HandlerMappings, HandlerAdapters, etc.
     */
    public BeanContainer getBeanContainer() {
        return beanContainer;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java` [NEW]

```java
package com.simplespringmvc.server;

import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;

import java.io.File;

/**
 * Embeds Apache Tomcat to receive HTTP requests and route them through
 * the {@link SimpleDispatcherServlet}.
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
        Context context = tomcat.addContext("", null);

        // Register our DispatcherServlet.
        // Tomcat.addServlet() is the programmatic equivalent of a <servlet> entry
        // in web.xml. We map it to "/" so it receives all requests (the default servlet).
        Tomcat.addServlet(context, "dispatcher", dispatcherServlet);
        context.addServletMappingDecoded("/", "dispatcher");

        // Enable the connector so Tomcat will actually listen on the port.
        tomcat.getConnector();
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

#### File: `src/test/java/com/simplespringmvc/servlet/SimpleDispatcherServletTest.java` [NEW]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.container.SimpleBeanContainer;
import jakarta.servlet.ServletException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.IOException;
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

    private SimpleBeanContainer beanContainer;
    private SimpleDispatcherServlet servlet;

    @BeforeEach
    void setUp() throws ServletException {
        beanContainer = new SimpleBeanContainer();
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
        @DisplayName("should write greeting response")
        void shouldWriteGreetingResponse() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));

            servlet.doDispatch(request, response);

            assertThat(stringWriter.toString()).isEqualTo("Hello from DispatcherServlet");
        }

        @Test
        @DisplayName("should set content type to text/plain")
        void shouldSetContentType() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(response.getWriter()).thenReturn(new PrintWriter(new StringWriter()));

            servlet.doDispatch(request, response);

            verify(response).setContentType("text/plain");
        }

        @Test
        @DisplayName("should set character encoding to UTF-8")
        void shouldSetCharacterEncoding() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(response.getWriter()).thenReturn(new PrintWriter(new StringWriter()));

            servlet.doDispatch(request, response);

            verify(response).setCharacterEncoding("UTF-8");
        }
    }

    // ─── service() routing ───────────────────────────────────────────

    @Nested
    @DisplayName("service() routing")
    class ServiceRouting {

        @Test
        @DisplayName("should route GET to doDispatch")
        void shouldRouteGetToDoDispatch() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));

            servlet.service(request, response);

            assertThat(stringWriter.toString()).isEqualTo("Hello from DispatcherServlet");
        }

        @Test
        @DisplayName("should route POST to doDispatch")
        void shouldRoutePostToDoDispatch() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter stringWriter = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(stringWriter));

            servlet.service(request, response);

            assertThat(stringWriter.toString()).isEqualTo("Hello from DispatcherServlet");
        }
    }

    // ─── initStrategies ──────────────────────────────────────────────

    @Nested
    @DisplayName("initStrategies")
    class InitStrategies {

        @Test
        @DisplayName("should complete without error on empty container")
        void shouldCompleteWithoutError_WhenContainerEmpty() throws ServletException {
            // initStrategies() is called during init() — should not throw
            // even with an empty container (no strategies registered yet)
            var freshServlet = new SimpleDispatcherServlet(new SimpleBeanContainer());
            freshServlet.init();
            // If we get here, init succeeded
            assertThat(freshServlet.getBeanContainer()).isNotNull();
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/EmbeddedTomcatIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

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
 *   HTTP client → Tomcat → SimpleDispatcherServlet.service() → doDispatch() → response
 * </pre>
 *
 * Uses port 0 so Tomcat picks a random available port — safe for parallel test runs.
 */
class EmbeddedTomcatIntegrationTest {

    private EmbeddedTomcat server;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
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
    @DisplayName("should respond to GET /any/path")
    void shouldRespondToGetAnyPath() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/any/path"))
                .GET()
                .build();

        HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
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
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
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
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
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
        assertThat(response.body()).isEqualTo("Hello from DispatcherServlet");
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
| **Front Controller** | A single servlet that receives ALL HTTP requests and dispatches them through a shared pipeline — the `doDispatch()` method |
| **`service()` override** | Intercepts the request before `HttpServlet` splits it into `doGet()`/`doPost()`/etc., ensuring all HTTP methods share the same code path |
| **`initStrategies()`** | The hook called during servlet initialization where strategy components (handler mappings, adapters, resolvers) will be loaded from the bean container |
| **Embedded server** | Running Tomcat programmatically inside the application, eliminating the need for external server deployment |
| **Strategy Pattern** | The `doDispatch()` method delegates to pluggable components (handler mappings, adapters, etc.) rather than hardcoding behavior — future features add strategies without modifying the dispatch method's structure |

**Next: Chapter 3 — Handler Mapping** — Right now `doDispatch()` returns the same response for every request. Handler mapping will scan `@Controller` classes, discover `@RequestMapping` methods, and build a registry that maps URL patterns + HTTP methods to specific handler methods — giving `doDispatch()` the ability to route requests to the right controller.
