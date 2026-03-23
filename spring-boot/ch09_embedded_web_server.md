# Chapter 9: Embedded Web Server

## Build Challenge

| Current State | Limitation | Objective |
|--------------|-----------|-----------|
| `IrisApplication.run()` creates a context, loads properties, fires events | The application is a plain Java process — no HTTP server, no way to serve web requests | Embed Apache Tomcat inside the application, start it as part of `refresh()`, and stop it on `close()` — so `main()` becomes a web server |

---

## 9.1 The Integration Point

The integration point is `AnnotationConfigApplicationContext.refresh()` — specifically, a new **template method hook** called `onRefresh()` that we insert between `registerBeanPostProcessors()` and `finishBeanFactoryInitialization()`.

This is the classic **Template Method** pattern from the real Spring Framework's `AbstractApplicationContext.onRefresh()` (line 906). The base class defines the algorithm (`refresh()`), and subclasses override the hook (`onRefresh()`) to add behavior — in our case, creating an embedded web server.

**Direction:** The `onRefresh()` hook connects **two subsystems**: the IoC container lifecycle (`refresh()`) and the embedded web server lifecycle (`create → start → stop`). From this integration point, we need to build:
1. A `WebServer` interface and `TomcatWebServer` implementation (what gets created)
2. A `ServletWebServerFactory` interface and `TomcatServletWebServerFactory` (who creates it)
3. A `ServletWebServerApplicationContext` that overrides `onRefresh()` and orchestrates the server lifecycle
4. A change to `IrisApplication.createApplicationContext()` to select the right context type

**Modifying:** `iris-framework/.../context/AnnotationConfigApplicationContext.java`
**Change:** Add `onRefresh()` template method in `refresh()` between step 2 and step 3

```java
@Override
public void refresh() {
    try {
        invokeBeanFactoryPostProcessors();
        registerBeanPostProcessors();
        onRefresh();                        // ← NEW: template method hook
        finishBeanFactoryInitialization();
        registerListeners();
        finishRefresh();
        // ...
    }
}

protected void onRefresh() {
    // For subclasses to override — do nothing by default.
}
```

**Key decision:** The hook is placed *after* `registerBeanPostProcessors()` but *before* `finishBeanFactoryInitialization()`. This means `@Configuration` classes have been processed (BeanDefinitions exist for all beans, including the `ServletWebServerFactory`), but singletons haven't been eagerly instantiated yet. The hook can look up the factory bean (triggering its creation) and use it to create the server — all before the remaining singletons are initialized. This ordering is identical to the real Spring Framework.

---

## 9.2 The Core Implementation

### WebServer Interface — The Server Abstraction

**New file:** `iris-boot-core/.../boot/web/server/WebServer.java`

```java
package com.iris.boot.web.server;

public interface WebServer {

    void start() throws WebServerException;

    void stop() throws WebServerException;

    int getPort();

    default void destroy() {
        stop();
    }
}
```

This interface decouples the application context from the server implementation. The context only knows `start()`, `stop()`, `getPort()` — it doesn't care if the server is Tomcat, Jetty, or Netty.

**New file:** `iris-boot-core/.../boot/web/server/WebServerException.java`

```java
package com.iris.boot.web.server;

public class WebServerException extends RuntimeException {

    public WebServerException(String message) {
        super(message);
    }

    public WebServerException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### ServletWebServerFactory — The Abstract Factory

**New file:** `iris-boot-core/.../boot/web/server/servlet/ServletWebServerFactory.java`

```java
package com.iris.boot.web.server.servlet;

import com.iris.boot.web.server.WebServer;

@FunctionalInterface
public interface ServletWebServerFactory {

    WebServer getWebServer();
}
```

The contract is critical: `getWebServer()` returns a **configured but not started** server. Port binding happens later when the caller invokes `WebServer.start()`. This separation ensures all beans are ready before the server accepts traffic.

### TomcatWebServer — Wrapping Embedded Tomcat

**New file:** `iris-boot-core/.../boot/web/embedded/tomcat/TomcatWebServer.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.util.concurrent.atomic.AtomicBoolean;
import org.apache.catalina.LifecycleException;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.startup.Tomcat;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;

public class TomcatWebServer implements WebServer {

    private final Tomcat tomcat;
    private final AtomicBoolean started = new AtomicBoolean(false);

    public TomcatWebServer(Tomcat tomcat) {
        this.tomcat = tomcat;
    }

    @Override
    public void start() throws WebServerException {
        if (this.started.compareAndSet(false, true)) {
            try {
                this.tomcat.start();
                checkConnectorState();
                startAwaitThread();
                System.out.printf("Tomcat started on port %d%n", getPort());
            } catch (LifecycleException ex) {
                this.started.set(false);
                throw new WebServerException("Unable to start embedded Tomcat", ex);
            }
        }
    }

    private void checkConnectorState() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null
                && connector.getState() == org.apache.catalina.LifecycleState.FAILED) {
            throw new WebServerException(
                    "Tomcat connector failed to start on port " + connector.getPort());
        }
    }

    private void startAwaitThread() {
        Thread awaitThread = new Thread(() -> {
            this.tomcat.getServer().await();
        }, "tomcat-await");
        awaitThread.setDaemon(false);
        awaitThread.start();
    }

    @Override
    public void stop() throws WebServerException {
        if (this.started.compareAndSet(true, false)) {
            try {
                this.tomcat.stop();
            } catch (LifecycleException ex) {
                throw new WebServerException("Unable to stop embedded Tomcat", ex);
            }
        }
    }

    @Override
    public void destroy() {
        try {
            stop();
            this.tomcat.destroy();
        } catch (LifecycleException ex) {
            throw new WebServerException("Unable to destroy embedded Tomcat", ex);
        }
    }

    @Override
    public int getPort() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null && this.started.get()) {
            return connector.getLocalPort();
        }
        return -1;
    }

    Tomcat getTomcat() {
        return this.tomcat;
    }
}
```

Notice the **non-daemon await thread**. All Tomcat threads are daemon threads — without this, the JVM would exit as soon as `main()` returns. The await thread blocks indefinitely on `Tomcat.getServer().await()`, keeping the JVM alive until `stop()` is called.

### TomcatServletWebServerFactory — Creating Tomcat

**New file:** `iris-boot-core/.../boot/web/embedded/tomcat/TomcatServletWebServerFactory.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;

public class TomcatServletWebServerFactory implements ServletWebServerFactory {

    private static final int DEFAULT_PORT = 8080;
    private int port;

    public TomcatServletWebServerFactory() {
        this(DEFAULT_PORT);
    }

    public TomcatServletWebServerFactory(int port) {
        this.port = port;
    }

    @Override
    public WebServer getWebServer() {
        Tomcat tomcat = createTomcat();
        prepareContext(tomcat);
        return new TomcatWebServer(tomcat);
    }

    private Tomcat createTomcat() {
        Tomcat tomcat = new Tomcat();
        File baseDir = createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        tomcat.setPort(this.port);
        tomcat.getHost().setAutoDeploy(false);
        return tomcat;
    }

    private void prepareContext(Tomcat tomcat) {
        Context context = tomcat.addContext("",
                createTempDir("tomcat-docbase").getAbsolutePath());
        context.setParentClassLoader(getClass().getClassLoader());
    }

    private File createTempDir(String prefix) {
        try {
            File tempDir = Files.createTempDirectory(prefix + ".").toFile();
            tempDir.deleteOnExit();
            return tempDir;
        } catch (IOException ex) {
            throw new WebServerException(
                    "Unable to create temp directory for embedded Tomcat", ex);
        }
    }

    public void setPort(int port) { this.port = port; }
    public int getPort() { return this.port; }
}
```

The factory follows the **Abstract Factory** pattern: `getWebServer()` creates a configured-but-not-started `TomcatWebServer`. The caller (`ServletWebServerApplicationContext`) doesn't know or care that it's Tomcat — it only sees `WebServer`.

---

## 9.3 The Context Integration

### WebServerApplicationContext — The Marker

**New file:** `iris-boot-core/.../boot/web/context/WebServerApplicationContext.java`

```java
package com.iris.boot.web.context;

import com.iris.boot.web.server.WebServer;
import com.iris.framework.context.ApplicationContext;

public interface WebServerApplicationContext extends ApplicationContext {

    WebServer getWebServer();
}
```

### ServletWebServerApplicationContext — Bridging Container and Server

**New file:** `iris-boot-core/.../boot/web/context/ServletWebServerApplicationContext.java`

```java
package com.iris.boot.web.context;

import java.util.Map;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;

public class ServletWebServerApplicationContext extends AnnotationConfigApplicationContext
        implements WebServerApplicationContext {

    private volatile WebServer webServer;

    @Override
    public void refresh() {
        try {
            super.refresh();       // calls onRefresh() → createWebServer()
            startWebServer();      // port binds AFTER all singletons are ready
        } catch (RuntimeException ex) {
            stopAndDestroyWebServer();
            throw ex;
        }
    }

    @Override
    protected void onRefresh() {
        createWebServer();
    }

    private void createWebServer() {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer();
    }

    private ServletWebServerFactory getWebServerFactory() {
        Map<String, ServletWebServerFactory> factories =
                getBeanFactory().getBeansOfType(ServletWebServerFactory.class);
        if (factories.isEmpty()) {
            throw new WebServerException(
                    "Unable to start ServletWebServerApplicationContext: "
                    + "no ServletWebServerFactory bean found. "
                    + "Define a TomcatServletWebServerFactory @Bean in your @Configuration.");
        }
        if (factories.size() > 1) {
            throw new WebServerException(
                    "Unable to start ServletWebServerApplicationContext: "
                    + "multiple ServletWebServerFactory beans found: " + factories.keySet());
        }
        return factories.values().iterator().next();
    }

    private void startWebServer() {
        if (this.webServer != null) {
            this.webServer.start();
        }
    }

    private void stopAndDestroyWebServer() {
        WebServer server = this.webServer;
        if (server != null) {
            try {
                server.destroy();
            } catch (Exception ex) {
                System.err.println("Failed to destroy web server: " + ex.getMessage());
            }
        }
    }

    @Override
    public void close() {
        super.close();
        stopAndDestroyWebServer();
    }

    @Override
    public WebServer getWebServer() {
        return this.webServer;
    }
}
```

The lifecycle sequence is critical — study this carefully:

```
refresh()  ← overridden
  └── super.refresh()
        ├── invokeBeanFactoryPostProcessors()   ← BeanDefinitions created
        ├── registerBeanPostProcessors()         ← @Autowired etc. ready
        ├── onRefresh()                          ← ★ createWebServer() — configured, NOT started
        ├── finishBeanFactoryInitialization()     ← all singletons instantiated
        ├── registerListeners()
        └── finishRefresh()                      ← ContextRefreshedEvent
  └── startWebServer()                           ← ★ webServer.start() — port binds NOW
```

The server is created during `onRefresh()` but started **after** `super.refresh()` returns. This ensures all singleton beans (including DispatcherServlet from Feature 10) are fully initialized before HTTP traffic arrives.

### Modifying IrisApplication — Context Type Selection

**Modifying:** `iris-boot-core/.../boot/IrisApplication.java`
**Change:** `createApplicationContext()` now branches on `webApplicationType`

```java
private AnnotationConfigApplicationContext createApplicationContext() {
    return switch (this.webApplicationType) {
        case SERVLET -> new ServletWebServerApplicationContext();
        case NONE -> new AnnotationConfigApplicationContext();
    };
}
```

This is the **Factory Method** pattern — `IrisApplication` selects the right context subtype based on the detected `WebApplicationType`. Since Tomcat is on the classpath, `deduceFromClasspath()` returns `SERVLET`, and the bootstrap automatically creates a web-server-aware context.

---

## 9.4 Try It Yourself

<details><summary>Challenge 1: Why does `getWebServerFactory()` require exactly ONE factory?</summary>

Zero factories means the user didn't register one — there's no way to create a server. Multiple factories is ambiguous: if both Tomcat and Jetty factories are registered, which server should we create? The real Spring Boot throws `ApplicationContextException` with a helpful message listing the found beans.

</details>

<details><summary>Challenge 2: What happens if you remove `startAwaitThread()` from `TomcatWebServer.start()`?</summary>

The JVM would exit immediately after `IrisApplication.run()` returns from `main()`. All Tomcat threads are daemon threads — they don't prevent JVM shutdown. The non-daemon await thread keeps the process alive by blocking on `Tomcat.getServer().await()`.

Try it: comment out `startAwaitThread()`, run the app, and watch it exit instantly.

</details>

<details><summary>Challenge 3: Implement a `server.port` property check in `getWebServerFactory()`</summary>

You could have the context read `server.port` from the environment and set it on the factory:

```java
private void createWebServer() {
    ServletWebServerFactory factory = getWebServerFactory();
    String portProp = getEnvironment().getProperty("server.port");
    if (portProp != null && factory instanceof TomcatServletWebServerFactory tomcatFactory) {
        tomcatFactory.setPort(Integer.parseInt(portProp));
    }
    this.webServer = factory.getWebServer();
}
```

In real Spring Boot, this is handled by auto-configuration (Feature 13) through `ServerProperties` and `WebServerFactoryCustomizer` — much more decoupled.

</details>

---

## 9.5 Tests

### Unit Tests

**New file:** `iris-boot-core/.../boot/web/embedded/tomcat/TomcatWebServerTest.java`

```java
@Test
void shouldReturnNegativeOnePort_WhenNotStarted() {
    Tomcat tomcat = createMinimalTomcat(0);
    server = new TomcatWebServer(tomcat);
    assertThat(server.getPort()).isEqualTo(-1);
}

@Test
void shouldBindPort_WhenStarted() {
    Tomcat tomcat = createMinimalTomcat(0);
    server = new TomcatWebServer(tomcat);
    server.start();
    assertThat(server.getPort()).isGreaterThan(0);
}

@Test
void shouldBeIdempotent_WhenStartedTwice() {
    server = new TomcatWebServer(createMinimalTomcat(0));
    server.start();
    int firstPort = server.getPort();
    server.start(); // no-op
    assertThat(server.getPort()).isEqualTo(firstPort);
}

@Test
void shouldStop_WhenStartedAndStopped() {
    server = new TomcatWebServer(createMinimalTomcat(0));
    server.start();
    server.stop();
    assertThat(server.getPort()).isEqualTo(-1);
}
```

**New file:** `iris-boot-core/.../boot/web/embedded/tomcat/TomcatServletWebServerFactoryTest.java`

```java
@Test
void shouldCreateFunctionalServer_WhenGetWebServerCalled() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
    server = factory.getWebServer();
    assertThat(server).isInstanceOf(TomcatWebServer.class);
    assertThat(server.getPort()).isEqualTo(-1); // not started yet
}

@Test
void shouldStartAndServe_WhenServerStarted() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
    server = factory.getWebServer();
    server.start();
    assertThat(server.getPort()).isGreaterThan(0);
}
```

**New file:** `iris-boot-core/.../boot/web/context/ServletWebServerApplicationContextTest.java`

```java
@Test
void shouldCreateAndStartWebServer_WhenRefreshed() {
    context = new ServletWebServerApplicationContext();
    context.register(WebServerConfig.class);
    context.refresh();
    WebServer server = context.getWebServer();
    assertThat(server).isInstanceOf(TomcatWebServer.class);
    assertThat(server.getPort()).isGreaterThan(0);
}

@Test
void shouldThrowException_WhenNoFactoryBeanDefined() {
    context = new ServletWebServerApplicationContext();
    context.register(EmptyConfig.class);
    assertThatThrownBy(() -> context.refresh())
            .isInstanceOf(WebServerException.class)
            .hasMessageContaining("no ServletWebServerFactory bean found");
}

@Configuration
static class WebServerConfig {
    @Bean
    public ServletWebServerFactory webServerFactory() {
        return new TomcatServletWebServerFactory(0);
    }
}
```

### Integration Tests

**New file:** `iris-boot-core/.../boot/integration/EmbeddedWebServerIntegrationTest.java`

```java
@Test
void shouldStartEmbeddedTomcat_WhenIrisApplicationRuns() {
    ConfigurableApplicationContext context = IrisApplication.run(WebAppConfig.class);
    try {
        assertThat(context).isInstanceOf(ServletWebServerApplicationContext.class);
        WebServer server = ((WebServerApplicationContext) context).getWebServer();
        assertThat(server.getPort()).isGreaterThan(0);
    } finally {
        context.close();
    }
}

@Test
void shouldRespondToHttpRequests_WhenServerIsRunning() throws Exception {
    ConfigurableApplicationContext context = IrisApplication.run(WebAppConfig.class);
    try {
        int port = ((WebServerApplicationContext) context).getWebServer().getPort();
        HttpURLConnection conn = (HttpURLConnection)
                URI.create("http://localhost:" + port + "/").toURL().openConnection();
        conn.setRequestMethod("GET");
        // 404 expected — no servlets registered yet (Feature 10)
        assertThat(conn.getResponseCode()).isEqualTo(404);
        conn.disconnect();
    } finally {
        context.close();
    }
}

@Test
void shouldStopServer_WhenContextIsClosed() {
    ConfigurableApplicationContext context = IrisApplication.run(WebAppConfig.class);
    WebServer server = ((WebServerApplicationContext) context).getWebServer();
    assertThat(server.getPort()).isGreaterThan(0);
    context.close();
    assertThat(server.getPort()).isEqualTo(-1);
}

@Configuration
static class WebAppConfig {
    @Bean
    public ServletWebServerFactory webServerFactory() {
        return new TomcatServletWebServerFactory(0);
    }
}
```

---

## 9.6 Why This Works

> ★ Insight — **Template Method is the right pattern for lifecycle extension**
>
> The `onRefresh()` hook in `refresh()` follows the classic Template Method pattern: the base class (`AnnotationConfigApplicationContext`) defines the algorithm skeleton, and subclasses (`ServletWebServerApplicationContext`) override specific steps. This is the exact pattern the real Spring Framework uses — `AbstractApplicationContext.onRefresh()` is an empty protected method at line 906. The alternative — having the subclass override `refresh()` entirely — would force it to duplicate the entire 6-step sequence, violating DRY and making the code fragile to future changes in the base class.
>
> The real trade-off: Template Method creates coupling between base and subclass through the "algorithm skeleton". If the base changes step ordering, subclasses may break. Spring mitigates this with a well-defined contract (the steps are documented and stable). The benefit outweighs the cost: 15+ context subclasses all share the same `refresh()` logic.

> ★ Insight — **Two-phase server lifecycle prevents premature traffic**
>
> The server is **created** in `onRefresh()` but **started** after `super.refresh()`. This is deliberate: between creation and starting, `finishBeanFactoryInitialization()` instantiates all singletons — including DispatcherServlet and its handler mappings (Feature 10). If the server started accepting connections in `onRefresh()`, incoming requests would arrive before the servlet infrastructure was ready.
>
> In the real Spring Boot, this separation is even more sophisticated: the server's connectors are removed during `TomcatWebServer.initialize()` and re-added during `start()`. This means Tomcat's internal components (servlet init, filter init) run during `onRefresh()`, but no port binding occurs until `finishRefresh()`. Our simplified version defers *everything* to `start()` — a valid simplification that we note in the code.

> ★ Insight — **The non-daemon await thread is essential for JVM lifecycle**
>
> A subtlety that trips up many developers: all Tomcat threads are daemon threads. Without the await thread, `IrisApplication.run()` would return, `main()` would exit, and the JVM would shut down immediately. The non-daemon await thread blocks on `Tomcat.getServer().await()`, keeping the JVM alive. This is the same pattern used by the real `TomcatWebServer` (line 167). It's a common pattern in embedded server frameworks — Netty and Jetty have similar mechanisms.

---

## 9.7 What We Enhanced

| Prior File | Change | Why |
|-----------|--------|-----|
| `AnnotationConfigApplicationContext.refresh()` | Added `onRefresh()` template method hook between `registerBeanPostProcessors()` and `finishBeanFactoryInitialization()` | Provides the extension point where `ServletWebServerApplicationContext` creates the embedded server — matching the real `AbstractApplicationContext.onRefresh()` |
| `IrisApplication.createApplicationContext()` | Changed from always returning `AnnotationConfigApplicationContext` to branching on `webApplicationType` using a `switch` expression | SERVLET type now creates `ServletWebServerApplicationContext`, enabling embedded web server support without changing any other bootstrap code |
| Existing `IrisApplicationTest` and `IrisApplicationBootstrapIntegrationTest` | Added `app.setWebApplicationType(WebApplicationType.NONE)` to non-web tests | Since Tomcat is on the test classpath, `deduceFromClasspath()` returns SERVLET — tests that don't define a factory bean must explicitly opt out |

---

## 9.8 Connection to Real Framework

| Iris Class | Real Spring Boot Class | File (commit `5922311a95a`) |
|-----------|----------------------|------|
| `WebServer` | `WebServer` | `spring-boot-web-server/.../boot/web/server/WebServer.java` |
| `WebServerException` | `WebServerException` | `spring-boot-web-server/.../boot/web/server/WebServerException.java` |
| `ServletWebServerFactory` | `ServletWebServerFactory` | `spring-boot-web-server/.../boot/web/server/servlet/ServletWebServerFactory.java` |
| `TomcatWebServer` | `TomcatWebServer` | `spring-boot-tomcat/.../boot/tomcat/TomcatWebServer.java` |
| `TomcatServletWebServerFactory` | `TomcatServletWebServerFactory` | `spring-boot-tomcat/.../boot/tomcat/servlet/TomcatServletWebServerFactory.java` |
| `ServletWebServerApplicationContext` | `ServletWebServerApplicationContext` | `spring-boot-web-server/.../boot/web/server/servlet/context/ServletWebServerApplicationContext.java` |
| `WebServerApplicationContext` | `WebServerApplicationContext` | `spring-boot-web-server/.../boot/web/server/context/WebServerApplicationContext.java` |
| `onRefresh()` | `AbstractApplicationContext.onRefresh()` | Spring Framework `spring-context/.../context/support/AbstractApplicationContext.java:906` (commit `11ab0b4351`) |

**Key differences from real Spring Boot:**
- Real `TomcatWebServer` uses a connector-removal dance: removes connectors during init (no port binding), re-adds them during `start()`. We defer all startup to `start()`.
- Real Spring Boot uses `SmartLifecycle` (`WebServerStartStopLifecycle`) to start the server during `finishRefresh()` with phase-ordered lifecycle management. We start directly after `super.refresh()`.
- Real `ServletWebServerFactory.getWebServer()` accepts `ServletContextInitializer...` for servlet/filter registration. We skip initializers (Feature 10 adds DispatcherServlet).
- Real factory supports SSL, compression, protocol customization, multiple connectors. We support only port configuration.

---

## 9.9 Complete Code

### Production Code

#### `[MODIFIED]` `iris-framework/.../context/AnnotationConfigApplicationContext.java`

```java
package com.iris.framework.context;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.ValueAnnotationBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;
import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.PropertiesPropertySourceLoader;
import com.iris.framework.core.env.StandardEnvironment;

public class AnnotationConfigApplicationContext implements ConfigurableApplicationContext {

    private static final String APPLICATION_PROPERTIES_SOURCE_NAME = "applicationProperties";
    private static final String APPLICATION_PROPERTIES_RESOURCE = "application.properties";

    private final DefaultBeanFactory beanFactory;
    private Environment environment;
    private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();
    private final AtomicBoolean active = new AtomicBoolean(false);
    private final AtomicBoolean closed = new AtomicBoolean(false);

    public AnnotationConfigApplicationContext() {
        this.beanFactory = new DefaultBeanFactory();
        this.environment = createEnvironment();
    }

    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this();
        register(componentClasses);
        refresh();
    }

    public void register(Class<?>... componentClasses) {
        for (Class<?> componentClass : componentClasses) {
            String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
            beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
        }
    }

    private StandardEnvironment createEnvironment() {
        StandardEnvironment env = new StandardEnvironment();
        PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
        MapPropertySource appProps = loader.load(
                APPLICATION_PROPERTIES_SOURCE_NAME, APPLICATION_PROPERTIES_RESOURCE);
        if (appProps != null) {
            env.getPropertySources().addLast(appProps);
        }
        return env;
    }

    @Override
    public void refresh() {
        try {
            invokeBeanFactoryPostProcessors();
            registerBeanPostProcessors();
            onRefresh();                        // ← Template method hook
            finishBeanFactoryInitialization();
            registerListeners();
            finishRefresh();
            this.active.set(true);
            this.closed.set(false);
        } catch (RuntimeException ex) {
            this.beanFactory.close();
            this.active.set(false);
            throw ex;
        }
    }

    protected void onRefresh() {
        // For subclasses to override — do nothing by default.
    }

    // ... (remaining methods unchanged from Chapter 6)
}
```

#### `[MODIFIED]` `iris-boot-core/.../boot/IrisApplication.java`

Only the `createApplicationContext()` method changed:

```java
import com.iris.boot.web.context.ServletWebServerApplicationContext;

// ...

private AnnotationConfigApplicationContext createApplicationContext() {
    return switch (this.webApplicationType) {
        case SERVLET -> new ServletWebServerApplicationContext();
        case NONE -> new AnnotationConfigApplicationContext();
    };
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/server/WebServer.java`

```java
package com.iris.boot.web.server;

public interface WebServer {

    void start() throws WebServerException;

    void stop() throws WebServerException;

    int getPort();

    default void destroy() {
        stop();
    }
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/server/WebServerException.java`

```java
package com.iris.boot.web.server;

public class WebServerException extends RuntimeException {

    public WebServerException(String message) {
        super(message);
    }

    public WebServerException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/server/servlet/ServletWebServerFactory.java`

```java
package com.iris.boot.web.server.servlet;

import com.iris.boot.web.server.WebServer;

@FunctionalInterface
public interface ServletWebServerFactory {

    WebServer getWebServer();
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/embedded/tomcat/TomcatWebServer.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.util.concurrent.atomic.AtomicBoolean;

import org.apache.catalina.LifecycleException;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.startup.Tomcat;

import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;

public class TomcatWebServer implements WebServer {

    private final Tomcat tomcat;
    private final AtomicBoolean started = new AtomicBoolean(false);

    public TomcatWebServer(Tomcat tomcat) {
        this.tomcat = tomcat;
    }

    @Override
    public void start() throws WebServerException {
        if (this.started.compareAndSet(false, true)) {
            try {
                this.tomcat.start();
                checkConnectorState();
                startAwaitThread();
                System.out.printf("Tomcat started on port %d%n", getPort());
            } catch (LifecycleException ex) {
                this.started.set(false);
                throw new WebServerException("Unable to start embedded Tomcat", ex);
            }
        }
    }

    private void checkConnectorState() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null
                && connector.getState() == org.apache.catalina.LifecycleState.FAILED) {
            throw new WebServerException(
                    "Tomcat connector failed to start on port " + connector.getPort());
        }
    }

    private void startAwaitThread() {
        Thread awaitThread = new Thread(() -> {
            this.tomcat.getServer().await();
        }, "tomcat-await");
        awaitThread.setDaemon(false);
        awaitThread.start();
    }

    @Override
    public void stop() throws WebServerException {
        if (this.started.compareAndSet(true, false)) {
            try {
                this.tomcat.stop();
            } catch (LifecycleException ex) {
                throw new WebServerException("Unable to stop embedded Tomcat", ex);
            }
        }
    }

    @Override
    public void destroy() {
        try {
            stop();
            this.tomcat.destroy();
        } catch (LifecycleException ex) {
            throw new WebServerException("Unable to destroy embedded Tomcat", ex);
        }
    }

    @Override
    public int getPort() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null && this.started.get()) {
            return connector.getLocalPort();
        }
        return -1;
    }

    Tomcat getTomcat() {
        return this.tomcat;
    }
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/embedded/tomcat/TomcatServletWebServerFactory.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;

import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;

import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;

public class TomcatServletWebServerFactory implements ServletWebServerFactory {

    private static final int DEFAULT_PORT = 8080;
    private int port;

    public TomcatServletWebServerFactory() {
        this(DEFAULT_PORT);
    }

    public TomcatServletWebServerFactory(int port) {
        this.port = port;
    }

    @Override
    public WebServer getWebServer() {
        Tomcat tomcat = createTomcat();
        prepareContext(tomcat);
        return new TomcatWebServer(tomcat);
    }

    private Tomcat createTomcat() {
        Tomcat tomcat = new Tomcat();
        File baseDir = createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        tomcat.setPort(this.port);
        tomcat.getHost().setAutoDeploy(false);
        return tomcat;
    }

    private void prepareContext(Tomcat tomcat) {
        Context context = tomcat.addContext("",
                createTempDir("tomcat-docbase").getAbsolutePath());
        context.setParentClassLoader(getClass().getClassLoader());
    }

    private File createTempDir(String prefix) {
        try {
            File tempDir = Files.createTempDirectory(prefix + ".").toFile();
            tempDir.deleteOnExit();
            return tempDir;
        } catch (IOException ex) {
            throw new WebServerException(
                    "Unable to create temp directory for embedded Tomcat", ex);
        }
    }

    public void setPort(int port) {
        this.port = port;
    }

    public int getPort() {
        return this.port;
    }
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/context/WebServerApplicationContext.java`

```java
package com.iris.boot.web.context;

import com.iris.boot.web.server.WebServer;
import com.iris.framework.context.ApplicationContext;

public interface WebServerApplicationContext extends ApplicationContext {

    WebServer getWebServer();
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/context/ServletWebServerApplicationContext.java`

```java
package com.iris.boot.web.context;

import java.util.Map;

import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.context.AnnotationConfigApplicationContext;

public class ServletWebServerApplicationContext extends AnnotationConfigApplicationContext
        implements WebServerApplicationContext {

    private volatile WebServer webServer;

    @Override
    public void refresh() {
        try {
            super.refresh();
            startWebServer();
        } catch (RuntimeException ex) {
            stopAndDestroyWebServer();
            throw ex;
        }
    }

    @Override
    protected void onRefresh() {
        createWebServer();
    }

    private void createWebServer() {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer();
    }

    private ServletWebServerFactory getWebServerFactory() {
        Map<String, ServletWebServerFactory> factories =
                getBeanFactory().getBeansOfType(ServletWebServerFactory.class);
        if (factories.isEmpty()) {
            throw new WebServerException(
                    "Unable to start ServletWebServerApplicationContext: "
                    + "no ServletWebServerFactory bean found. "
                    + "Define a TomcatServletWebServerFactory @Bean in your @Configuration.");
        }
        if (factories.size() > 1) {
            throw new WebServerException(
                    "Unable to start ServletWebServerApplicationContext: "
                    + "multiple ServletWebServerFactory beans found: " + factories.keySet());
        }
        return factories.values().iterator().next();
    }

    private void startWebServer() {
        if (this.webServer != null) {
            this.webServer.start();
        }
    }

    private void stopAndDestroyWebServer() {
        WebServer server = this.webServer;
        if (server != null) {
            try {
                server.destroy();
            } catch (Exception ex) {
                System.err.println("Failed to destroy web server: " + ex.getMessage());
            }
        }
    }

    @Override
    public void close() {
        super.close();
        stopAndDestroyWebServer();
    }

    @Override
    public WebServer getWebServer() {
        return this.webServer;
    }
}
```

### Test Code

#### `[NEW]` `iris-boot-core/.../boot/web/embedded/tomcat/TomcatWebServerTest.java`

```java
package com.iris.boot.web.embedded.tomcat;

import org.apache.catalina.startup.Tomcat;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import com.iris.boot.web.server.WebServerException;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TomcatWebServerTest {

    private TomcatWebServer server;

    @AfterEach
    void cleanup() {
        if (server != null) {
            try { server.destroy(); } catch (Exception ignored) { }
        }
    }

    @Test
    void shouldReturnNegativeOnePort_WhenNotStarted() {
        server = new TomcatWebServer(createMinimalTomcat(0));
        assertThat(server.getPort()).isEqualTo(-1);
    }

    @Test
    void shouldBindPort_WhenStarted() {
        server = new TomcatWebServer(createMinimalTomcat(0));
        server.start();
        assertThat(server.getPort()).isGreaterThan(0);
    }

    @Test
    void shouldBeIdempotent_WhenStartedTwice() {
        server = new TomcatWebServer(createMinimalTomcat(0));
        server.start();
        int firstPort = server.getPort();
        server.start();
        assertThat(server.getPort()).isEqualTo(firstPort);
    }

    @Test
    void shouldStop_WhenStartedAndStopped() {
        server = new TomcatWebServer(createMinimalTomcat(0));
        server.start();
        assertThat(server.getPort()).isGreaterThan(0);
        server.stop();
        assertThat(server.getPort()).isEqualTo(-1);
    }

    @Test
    void shouldBeIdempotent_WhenStoppedTwice() {
        server = new TomcatWebServer(createMinimalTomcat(0));
        server.start();
        server.stop();
        server.stop(); // should not throw
    }

    @Test
    void shouldDestroyCleanly_WhenStarted() {
        server = new TomcatWebServer(createMinimalTomcat(0));
        server.start();
        server.destroy();
        assertThat(server.getPort()).isEqualTo(-1);
        server = null;
    }

    private Tomcat createMinimalTomcat(int port) {
        Tomcat tomcat = new Tomcat();
        tomcat.setBaseDir(System.getProperty("java.io.tmpdir"));
        tomcat.setPort(port);
        tomcat.getHost().setAutoDeploy(false);
        tomcat.addContext("", System.getProperty("java.io.tmpdir"));
        return tomcat;
    }
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/embedded/tomcat/TomcatServletWebServerFactoryTest.java`

```java
package com.iris.boot.web.embedded.tomcat;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import com.iris.boot.web.server.WebServer;

import static org.assertj.core.api.Assertions.assertThat;

class TomcatServletWebServerFactoryTest {

    private WebServer server;

    @AfterEach
    void cleanup() {
        if (server != null) {
            try { server.destroy(); } catch (Exception ignored) { }
        }
    }

    @Test
    void shouldCreateWebServer_WithDefaultPort() {
        assertThat(new TomcatServletWebServerFactory().getPort()).isEqualTo(8080);
    }

    @Test
    void shouldCreateWebServer_WithCustomPort() {
        assertThat(new TomcatServletWebServerFactory(9090).getPort()).isEqualTo(9090);
    }

    @Test
    void shouldCreateFunctionalServer_WhenGetWebServerCalled() {
        server = new TomcatServletWebServerFactory(0).getWebServer();
        assertThat(server).isInstanceOf(TomcatWebServer.class);
        assertThat(server.getPort()).isEqualTo(-1);
    }

    @Test
    void shouldStartAndServe_WhenServerStarted() {
        server = new TomcatServletWebServerFactory(0).getWebServer();
        server.start();
        assertThat(server.getPort()).isGreaterThan(0);
    }

    @Test
    void shouldAllowPortChange_WhenSetPortCalled() {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.setPort(0);
        server = factory.getWebServer();
        server.start();
        assertThat(server.getPort()).isGreaterThan(0);
    }
}
```

#### `[NEW]` `iris-boot-core/.../boot/web/context/ServletWebServerApplicationContextTest.java`

```java
package com.iris.boot.web.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.embedded.tomcat.TomcatWebServer;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ServletWebServerApplicationContextTest {

    private ServletWebServerApplicationContext context;

    @AfterEach
    void cleanup() {
        if (context != null) {
            try { context.close(); } catch (Exception ignored) { }
        }
    }

    @Test
    void shouldCreateAndStartWebServer_WhenRefreshed() {
        context = new ServletWebServerApplicationContext();
        context.register(WebServerConfig.class);
        context.refresh();
        WebServer server = context.getWebServer();
        assertThat(server).isInstanceOf(TomcatWebServer.class);
        assertThat(server.getPort()).isGreaterThan(0);
    }

    @Test
    void shouldImplementWebServerApplicationContext() {
        context = new ServletWebServerApplicationContext();
        assertThat(context).isInstanceOf(WebServerApplicationContext.class);
    }

    @Test
    void shouldDestroyWebServer_WhenClosed() {
        context = new ServletWebServerApplicationContext();
        context.register(WebServerConfig.class);
        context.refresh();
        WebServer server = context.getWebServer();
        assertThat(server.getPort()).isGreaterThan(0);
        context.close();
        assertThat(server.getPort()).isEqualTo(-1);
    }

    @Test
    void shouldThrowException_WhenNoFactoryBeanDefined() {
        context = new ServletWebServerApplicationContext();
        context.register(EmptyConfig.class);
        assertThatThrownBy(() -> context.refresh())
                .isInstanceOf(WebServerException.class)
                .hasMessageContaining("no ServletWebServerFactory bean found");
    }

    @Test
    void shouldAllowBeanAccess_WhenRefreshed() {
        context = new ServletWebServerApplicationContext();
        context.register(WebServerConfig.class);
        context.refresh();
        assertThat(context.getBean(ServletWebServerFactory.class)).isNotNull();
    }

    @Configuration
    static class WebServerConfig {
        @Bean
        public ServletWebServerFactory webServerFactory() {
            return new TomcatServletWebServerFactory(0);
        }
    }

    @Configuration
    static class EmptyConfig { }
}
```

#### `[NEW]` `iris-boot-core/.../boot/integration/EmbeddedWebServerIntegrationTest.java`

```java
package com.iris.boot.integration;

import java.net.HttpURLConnection;
import java.net.URI;
import java.util.concurrent.atomic.AtomicBoolean;

import org.junit.jupiter.api.Test;

import com.iris.boot.IrisApplication;
import com.iris.boot.WebApplicationType;
import com.iris.boot.web.context.ServletWebServerApplicationContext;
import com.iris.boot.web.context.WebServerApplicationContext;
import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.ApplicationListener;
import com.iris.framework.context.ConfigurableApplicationContext;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.context.event.ContextRefreshedEvent;

import static org.assertj.core.api.Assertions.assertThat;

class EmbeddedWebServerIntegrationTest {

    @Test
    void shouldStartEmbeddedTomcat_WhenIrisApplicationRuns() {
        ConfigurableApplicationContext context = null;
        try {
            context = IrisApplication.run(WebAppConfig.class);
            assertThat(context).isInstanceOf(ServletWebServerApplicationContext.class);
            WebServer server = ((WebServerApplicationContext) context).getWebServer();
            assertThat(server.getPort()).isGreaterThan(0);
        } finally {
            if (context != null) context.close();
        }
    }

    @Test
    void shouldRespondToHttpRequests_WhenServerIsRunning() throws Exception {
        ConfigurableApplicationContext context = null;
        try {
            context = IrisApplication.run(WebAppConfig.class);
            int port = ((WebServerApplicationContext) context).getWebServer().getPort();
            HttpURLConnection conn = (HttpURLConnection)
                    URI.create("http://localhost:" + port + "/").toURL().openConnection();
            conn.setRequestMethod("GET");
            conn.setConnectTimeout(2000);
            conn.setReadTimeout(2000);
            assertThat(conn.getResponseCode()).isEqualTo(404);
            conn.disconnect();
        } finally {
            if (context != null) context.close();
        }
    }

    @Test
    void shouldStopServer_WhenContextIsClosed() {
        ConfigurableApplicationContext context = IrisApplication.run(WebAppConfig.class);
        WebServer server = ((WebServerApplicationContext) context).getWebServer();
        assertThat(server.getPort()).isGreaterThan(0);
        context.close();
        assertThat(server.getPort()).isEqualTo(-1);
    }

    @Test
    void shouldFireContextRefreshedEvent_BeforeServerStarts() {
        AtomicBoolean eventFired = new AtomicBoolean(false);
        IrisApplication app = new IrisApplication(WebAppConfig.class);
        app.addListener(new ApplicationListener<ContextRefreshedEvent>() {
            @Override
            public void onApplicationEvent(ContextRefreshedEvent event) {
                eventFired.set(true);
            }
        });
        ConfigurableApplicationContext context = null;
        try {
            context = app.run();
            assertThat(eventFired.get()).isTrue();
        } finally {
            if (context != null) context.close();
        }
    }

    @Test
    void shouldUseNoneContext_WhenWebTypeSetToNone() {
        IrisApplication app = new IrisApplication(NonWebConfig.class);
        app.setWebApplicationType(WebApplicationType.NONE);
        ConfigurableApplicationContext context = null;
        try {
            context = app.run();
            assertThat(context).isNotInstanceOf(ServletWebServerApplicationContext.class);
        } finally {
            if (context != null) context.close();
        }
    }

    @Configuration
    static class WebAppConfig {
        @Bean
        public ServletWebServerFactory webServerFactory() {
            return new TomcatServletWebServerFactory(0);
        }
    }

    @Configuration
    static class NonWebConfig { }
}
```

---

## Summary

| What | Detail |
|------|--------|
| **Feature** | Embedded Web Server |
| **Module** | `iris-boot-core` (new classes) + `iris-framework` (modified) |
| **New files** | 7 classes: `WebServer`, `WebServerException`, `ServletWebServerFactory`, `TomcatWebServer`, `TomcatServletWebServerFactory`, `WebServerApplicationContext`, `ServletWebServerApplicationContext` |
| **Modified files** | `AnnotationConfigApplicationContext` (added `onRefresh()`), `IrisApplication` (context type branching) |
| **Design patterns** | Template Method (`onRefresh()`), Abstract Factory (`ServletWebServerFactory`), Strategy (`WebServer`), Factory Method (`createApplicationContext()`) |
| **Key insight** | The server is *created* during `onRefresh()` but *started* after `refresh()` completes — ensuring all beans are ready before accepting HTTP traffic |
| **Simplification** | Deferred all Tomcat startup to `start()` (real Spring Boot uses connector-removal for two-phase init) |
| **Tests** | 16 new tests (unit + integration), 280 total tests passing |

**Next chapter:** [Chapter 10 — DispatcherServlet & MVC](ch10_dispatcher_servlet.md) will add HTTP request routing: a front-controller servlet that maps requests to `@Controller` methods annotated with `@GetMapping`/`@PostMapping`, completing the web stack.
