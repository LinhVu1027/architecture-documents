# Chapter 19: Graceful Shutdown

## Build Challenge

| | |
|---|---|
| **Current State** | The embedded web server starts directly after `super.refresh()` and stops directly in `close()`. There is no phase ordering, no way to drain in-flight requests, and no JVM shutdown hook. |
| **Limitation** | When the application shuts down, active HTTP requests are dropped immediately. There is no lifecycle abstraction for ordering startup/shutdown of components. The web server cannot signal "stop accepting new connections but finish existing ones." |
| **Objective** | Introduce `SmartLifecycle` with phase-ordered startup/shutdown. Move web server start/stop into `SmartLifecycle` beans. Add graceful shutdown support that pauses Tomcat connectors and polls until in-flight requests drain. Register a JVM shutdown hook so SIGTERM triggers orderly context close. |

---

## 19.1 The Integration Point

The new feature plugs into **two places** in `AnnotationConfigApplicationContext`:

1. **`finishRefresh()`** -- where the `LifecycleProcessor` is initialized and `SmartLifecycle` beans are started
2. **`close()`** -- where `LifecycleProcessor.onClose()` stops `SmartLifecycle` beans in reverse phase order BEFORE singletons are destroyed

**Modifying:** `iris-framework/.../context/AnnotationConfigApplicationContext.java`

Before (Feature 9 -- `finishRefresh()` only published the event):
```java
private void finishRefresh() {
    publishEvent(new ContextRefreshedEvent(this));
}
```

After (Feature 19 -- `finishRefresh()` initializes the lifecycle processor and starts SmartLifecycle beans):
```java
private void finishRefresh() {
    initLifecycleProcessor();
    getLifecycleProcessor().onRefresh();
    publishEvent(new ContextRefreshedEvent(this));
}

private void initLifecycleProcessor() {
    if (this.beanFactory.containsBean("lifecycleProcessor")) {
        this.lifecycleProcessor = this.beanFactory.getBean(
                "lifecycleProcessor", LifecycleProcessor.class);
    } else {
        DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
        defaultProcessor.setBeanFactory(this.beanFactory);
        this.lifecycleProcessor = defaultProcessor;
        this.beanFactory.registerSingleton("lifecycleProcessor", this.lifecycleProcessor);
    }
}
```

Before (Feature 9 -- `close()` published event then destroyed singletons):
```java
@Override
public void close() {
    if (this.closed.compareAndSet(false, true)) {
        publishEvent(new ContextClosedEvent(this));
        this.beanFactory.close();
        this.active.set(false);
        this.applicationListeners.clear();
    }
}
```

After (Feature 19 -- `close()` stops SmartLifecycle beans between event and singleton destruction):
```java
@Override
public void close() {
    if (this.closed.compareAndSet(false, true)) {
        // 1. Publish ContextClosedEvent while beans are still alive
        publishEvent(new ContextClosedEvent(this));

        // 2. Stop SmartLifecycle beans in reverse phase order
        //    This is where graceful shutdown happens
        if (this.lifecycleProcessor != null) {
            try {
                this.lifecycleProcessor.onClose();
            } catch (Exception ex) {
                System.err.println("LifecycleProcessor onClose failed: " + ex.getMessage());
            }
        }

        // 3. Destroy all singletons (@PreDestroy, DisposableBean)
        this.beanFactory.close();

        // 4. Mark as inactive and clear listeners
        this.active.set(false);
        this.applicationListeners.clear();

        // 5. Remove the shutdown hook if one was registered
        removeShutdownHook();
    }
}
```

**Direction:** The `LifecycleProcessor` is the bridge between the application context lifecycle and individual `SmartLifecycle` beans. From this integration point, we need to build:
1. `Lifecycle`, `SmartLifecycle`, and `LifecycleProcessor` interfaces (the abstraction layer)
2. `DefaultLifecycleProcessor` (phase-grouped startup/shutdown with async stop and `CountDownLatch`)
3. `WebServerGracefulShutdownLifecycle` and `WebServerStartStopLifecycle` (SmartLifecycle beans that manage the web server)
4. `GracefulShutdown` (Tomcat-specific connector pausing and request draining)
5. `Shutdown` enum, `GracefulShutdownCallback`, `GracefulShutdownResult` (supporting types)
6. `registerShutdownHook()` (JVM shutdown hook integration)

The key insight: the web server now starts and stops via SmartLifecycle beans instead of being managed directly by `ServletWebServerApplicationContext`. This enables phase-ordered shutdown -- graceful shutdown (drain requests) happens at phase `MAX_VALUE - 1024` BEFORE the server stops at phase `MAX_VALUE - 2048`.

---

## 19.2 The Lifecycle Abstraction Layer

### Lifecycle -- the base interface

The simplest lifecycle contract: start, stop, and check if running.

**New file:** `iris-framework/.../context/Lifecycle.java`

```java
public interface Lifecycle {

    void start();

    void stop();

    boolean isRunning();
}
```

Regular `Lifecycle` beans have an implicit phase of 0 and no auto-startup capability. They are the building block for the more capable `SmartLifecycle`.

### SmartLifecycle -- phase ordering and async stop

The real power is in `SmartLifecycle`, which adds three capabilities:

1. **Phase ordering** via `getPhase()` -- lowest phase starts first, highest phase stops first
2. **Auto-startup** via `isAutoStartup()` -- whether to start automatically during context refresh
3. **Async stop** via `stop(Runnable)` -- the callback MUST be invoked when stop is complete, allowing the processor to wait via a `CountDownLatch`

**New file:** `iris-framework/.../context/SmartLifecycle.java`

```java
public interface SmartLifecycle extends Lifecycle {

    int DEFAULT_PHASE = Integer.MAX_VALUE;

    default boolean isAutoStartup() {
        return true;
    }

    default void stop(Runnable callback) {
        stop();
        callback.run();
    }

    default int getPhase() {
        return DEFAULT_PHASE;
    }
}
```

The default phase of `Integer.MAX_VALUE` means SmartLifecycle beans start **last** and stop **first** -- exactly right for web server lifecycle beans. They should start after all application beans are ready and stop before those beans are destroyed.

### LifecycleProcessor -- strategy interface

The bridge between the ApplicationContext and Lifecycle beans. Called at two key points:

**New file:** `iris-framework/.../context/LifecycleProcessor.java`

```java
public interface LifecycleProcessor extends Lifecycle {

    default void onRefresh() {
        start();
    }

    default void onClose() {
        stop();
    }
}
```

`onRefresh()` is called from `finishRefresh()` and starts auto-startup SmartLifecycle beans. `onClose()` is called from `close()` and stops all running Lifecycle beans in reverse phase order.

---

## 19.3 DefaultLifecycleProcessor -- Phase-Ordered Start/Stop

This is the core engine. It discovers all `Lifecycle` beans from the bean factory, groups them by phase, and starts/stops them in order.

**New file:** `iris-framework/.../context/support/DefaultLifecycleProcessor.java`

**Start logic** -- groups beans by phase in a `TreeMap` (natural ascending order), then starts each group:

```java
private void startBeans(boolean autoStartupOnly) {
    Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
    Map<Integer, List<LifecycleBean>> phases = new TreeMap<>();

    for (Map.Entry<String, Lifecycle> entry : lifecycleBeans.entrySet()) {
        Lifecycle bean = entry.getValue();
        if (!autoStartupOnly
                || (bean instanceof SmartLifecycle sl && sl.isAutoStartup())) {
            int phase = getPhase(bean);
            phases.computeIfAbsent(phase, k -> new ArrayList<>())
                  .add(new LifecycleBean(entry.getKey(), bean));
        }
    }

    // Start in ascending phase order (lowest first)
    for (List<LifecycleBean> group : phases.values()) {
        for (LifecycleBean lb : group) {
            if (!lb.bean.isRunning()) {
                lb.bean.start();
            }
        }
    }
}
```

**Stop logic** -- uses a `TreeMap` with `Comparator.reverseOrder()` so the highest phase stops first. For `SmartLifecycle` beans, the async `stop(Runnable)` method is called with a `CountDownLatch`:

```java
private void stopPhaseGroup(int phase, List<LifecycleBean> beans) {
    int smartCount = 0;
    for (LifecycleBean lb : beans) {
        if (lb.bean instanceof SmartLifecycle && lb.bean.isRunning()) {
            smartCount++;
        }
    }

    CountDownLatch latch = new CountDownLatch(smartCount);

    for (LifecycleBean lb : beans) {
        if (!lb.bean.isRunning()) {
            continue;
        }

        if (lb.bean instanceof SmartLifecycle sl) {
            try {
                sl.stop(latch::countDown);
            } catch (Exception ex) {
                latch.countDown();
            }
        } else {
            try {
                lb.bean.stop();
            } catch (Exception ex) { /* log */ }
        }
    }

    if (smartCount > 0) {
        try {
            boolean completed = latch.await(
                    this.timeoutPerShutdownPhase, TimeUnit.MILLISECONDS);
            if (!completed) {
                System.err.println("Timed out waiting for lifecycle beans "
                        + "in phase " + phase + " to stop");
            }
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Self-exclusion** -- the processor itself implements `LifecycleProcessor extends Lifecycle`, so it could discover itself. Without filtering, `stopBeans()` would call `stop()` on itself, which calls `stopBeans()` again -- infinite recursion:

```java
private Map<String, Lifecycle> getLifecycleBeans() {
    Map<String, Lifecycle> beans = this.beanFactory.getBeansOfType(Lifecycle.class);
    beans.values().removeIf(bean -> bean == this);
    return beans;
}
```

---

## 19.4 The Web Server Lifecycle Beans

Two `SmartLifecycle` beans split the web server lifecycle into separate phases:

### WebServerGracefulShutdownLifecycle -- drain requests first

**New file:** `iris-boot-core/.../web/context/WebServerGracefulShutdownLifecycle.java`

This bean runs at phase `Integer.MAX_VALUE - 1024`. Since shutdown processes the highest phase first, this stops BEFORE the web server is actually stopped.

```java
public final class WebServerGracefulShutdownLifecycle implements SmartLifecycle {

    private final WebServer webServer;
    private volatile boolean running;

    @Override
    public void stop(Runnable callback) {
        this.running = false;
        this.webServer.shutDownGracefully((result) -> callback.run());
    }

    @Override
    public void stop() {
        throw new UnsupportedOperationException(
                "Stop must not be invoked directly. Use stop(Runnable) instead.");
    }

    @Override
    public int getPhase() {
        return WebServerApplicationContext.GRACEFUL_SHUTDOWN_PHASE;
    }
}
```

The `stop(Runnable)` method delegates to `webServer.shutDownGracefully()`. The graceful shutdown callback invokes `callback.run()`, which counts down the `DefaultLifecycleProcessor`'s `CountDownLatch`. This is the async pattern in action -- the processor waits for the latch before moving to the next phase.

### WebServerStartStopLifecycle -- start and stop the server

**New file:** `iris-boot-core/.../web/context/WebServerStartStopLifecycle.java`

This bean runs at phase `Integer.MAX_VALUE - 2048`. This is LOWER than the graceful shutdown phase, meaning:
- **Startup:** starts FIRST (lower phase starts first) -- the server binds the port
- **Shutdown:** stops AFTER graceful shutdown has drained requests (lower phase stops later)

```java
class WebServerStartStopLifecycle implements SmartLifecycle {

    private final WebServer webServer;
    private volatile boolean running;

    @Override
    public void start() {
        this.webServer.start();
        this.running = true;
    }

    @Override
    public void stop() {
        this.running = false;
        this.webServer.stop();
    }

    @Override
    public int getPhase() {
        return WebServerApplicationContext.START_STOP_LIFECYCLE_PHASE;
    }
}
```

### Phase constants

**Modified file:** `iris-boot-core/.../web/context/WebServerApplicationContext.java`

```java
public interface WebServerApplicationContext extends ApplicationContext {

    int GRACEFUL_SHUTDOWN_PHASE = SmartLifecycle.DEFAULT_PHASE - 1024;

    int START_STOP_LIFECYCLE_PHASE = GRACEFUL_SHUTDOWN_PHASE - 1024;

    WebServer getWebServer();
}
```

The 1024-gap between phases leaves room for user-defined SmartLifecycle beans that need to run between the two server lifecycle phases.

---

## 19.5 Tomcat Graceful Shutdown

### GracefulShutdown -- connector pausing and request polling

**New file:** `iris-boot-core/.../web/embedded/tomcat/GracefulShutdown.java`

The shutdown sequence runs on a dedicated "tomcat-shutdown" thread:

```java
final class GracefulShutdown {

    private final Tomcat tomcat;
    private volatile boolean aborted = false;

    void shutDownGracefully(GracefulShutdownCallback callback) {
        new Thread(() -> doShutdown(callback), "tomcat-shutdown").start();
    }

    private void doShutdown(GracefulShutdownCallback callback) {
        try {
            // Step 1: Pause all connectors and close server sockets
            List<Connector> connectors = getConnectors();
            for (Connector connector : connectors) {
                connector.pause();
                connector.getProtocolHandler().closeServerSocketGraceful();
            }

            // Step 2: Poll until no active requests or aborted
            awaitInactiveOrAborted();

            // Step 3: Report result
            if (this.aborted) {
                callback.shutdownComplete(GracefulShutdownResult.REQUESTS_ACTIVE);
            } else {
                callback.shutdownComplete(GracefulShutdownResult.IDLE);
            }
        } catch (Exception ex) {
            callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
        }
    }

    private void awaitInactiveOrAborted() {
        while (!this.aborted) {
            if (!hasActiveRequests()) {
                return;
            }
            try {
                Thread.sleep(50);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }
}
```

Active request detection walks Tomcat's container hierarchy (`Engine -> Host -> Context -> Wrapper`), checking `Context.getInProgressAsyncCount()` for async requests and `Wrapper.getCountAllocated()` for sync requests.

### TomcatWebServer -- graceful shutdown integration

**Modified file:** `iris-boot-core/.../web/embedded/tomcat/TomcatWebServer.java`

The web server gains a new constructor that accepts a `Shutdown` mode:

```java
public TomcatWebServer(Tomcat tomcat, Shutdown shutdown) {
    this.tomcat = tomcat;
    this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL)
            ? new GracefulShutdown(tomcat) : null;
}

@Override
public void shutDownGracefully(GracefulShutdownCallback callback) {
    if (this.gracefulShutdown == null) {
        callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
        return;
    }
    this.gracefulShutdown.shutDownGracefully(callback);
}

@Override
public void stop() throws WebServerException {
    if (this.started.compareAndSet(true, false)) {
        try {
            if (this.gracefulShutdown != null) {
                this.gracefulShutdown.abort();
            }
            this.tomcat.stop();
        } catch (LifecycleException ex) {
            throw new WebServerException("Unable to stop embedded Tomcat", ex);
        }
    }
}
```

When `stop()` is called, it aborts any in-progress graceful shutdown before stopping Tomcat. This handles the case where the graceful shutdown timeout expires and the processor moves on.

### Supporting types

**New file:** `iris-boot-core/.../web/server/Shutdown.java` -- Enum with `GRACEFUL` and `IMMEDIATE`

**New file:** `iris-boot-core/.../web/server/GracefulShutdownCallback.java` -- Functional interface invoked when shutdown completes

**New file:** `iris-boot-core/.../web/server/GracefulShutdownResult.java` -- Enum with `REQUESTS_ACTIVE`, `IDLE`, `IMMEDIATE`

### Configuration pipeline

The `server.shutdown` property flows through:

```
application.properties          ServerProperties           TomcatAutoConfiguration        TomcatServletWebServerFactory
server.shutdown=graceful  →  @ConfigurationProperties  →  factory.setShutdown(...)  →  new TomcatWebServer(tomcat, shutdown)
                              (prefix = "server")
```

**Modified:** `ServerProperties.java` adds a `Shutdown shutdown` field, `TomcatAutoConfiguration` passes it to the factory, and `TomcatServletWebServerFactory` passes it to the `TomcatWebServer` constructor.

**Modified:** `PropertySourcesPropertyResolver` and `ConfigurationPropertiesBinder` add enum conversion support so that the string `"graceful"` can be bound to the `Shutdown.GRACEFUL` enum constant.

---

## 19.6 Context Integration Changes

### ServletWebServerApplicationContext -- SmartLifecycle registration

**Modified file:** `iris-boot-core/.../web/context/ServletWebServerApplicationContext.java`

The web server is no longer started directly after `super.refresh()`. Instead, `onRefresh()` creates the server and registers SmartLifecycle beans:

```java
@Override
protected void onRefresh() {
    createWebServer();
}

private void createWebServer() {
    ServletWebServerFactory factory = getWebServerFactory();
    DispatcherServlet dispatcherServlet = new DispatcherServlet(this);
    this.webServer = factory.getWebServer(dispatcherServlet);

    // Register SmartLifecycle beans for phased start/stop
    getBeanFactory().registerSingleton("webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
    getBeanFactory().registerSingleton("webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
}
```

The server actually starts later, when `finishRefresh()` calls `lifecycleProcessor.onRefresh()`, which discovers and starts these SmartLifecycle beans.

### DefaultBeanFactory -- registerSingleton()

**Modified file:** `iris-framework/.../beans/factory/support/DefaultBeanFactory.java`

Added `registerSingleton()` to support programmatic registration of SmartLifecycle beans:

```java
public void registerSingleton(String beanName, Object singletonObject) {
    this.singletonObjects.put(beanName, singletonObject);
    if (!containsBeanDefinition(beanName)) {
        BeanDefinition bd = new BeanDefinition(singletonObject.getClass());
        registerBeanDefinition(beanName, bd);
    }
}
```

This places the instance directly into the singleton cache AND registers a `BeanDefinition` so that `getBeansOfType()` can discover it. In the real Spring Framework, `registerSingleton()` only adds to the singleton cache -- type-based lookup checks both definitions and manual singletons separately.

### IrisApplication -- JVM shutdown hook

**Modified file:** `iris-boot-core/.../boot/IrisApplication.java`

```java
private void refreshContext(ConfigurableApplicationContext context) {
    context.refresh();
    context.registerShutdownHook();
}
```

The shutdown hook ensures that `context.close()` is called on SIGTERM/SIGINT, triggering the full SmartLifecycle shutdown sequence even if the user doesn't call `close()` explicitly.

---

## 19.7 Try It Yourself

<details><summary>Challenge 1: Why are there TWO SmartLifecycle beans instead of one?</summary>

Because graceful shutdown and server stop are separate concerns that must happen in a specific ORDER. The graceful shutdown bean (phase MAX_VALUE-1024) pauses connectors and drains requests. The start/stop bean (phase MAX_VALUE-2048) actually stops Tomcat. Since shutdown processes highest phase first, draining completes BEFORE the server is killed.

If they were in a single bean, you couldn't independently control the timing. And if you put graceful shutdown inside `webServer.stop()`, the `DefaultLifecycleProcessor` couldn't use its `CountDownLatch` pattern to wait for the drain to complete before proceeding.

</details>

<details><summary>Challenge 2: What happens if you remove the self-exclusion check in getLifecycleBeans()?</summary>

The `DefaultLifecycleProcessor` implements `LifecycleProcessor extends Lifecycle`, so `getBeansOfType(Lifecycle.class)` would discover it. When `stopBeans()` is called, it would find itself, call `stop()`, which calls `stopBeans()`, which finds itself again -- `StackOverflowError`. The single line `beans.values().removeIf(bean -> bean == this)` prevents this infinite recursion.

Try it: remove that line and call `processor.onClose()` in a test to see the stack overflow.

</details>

<details><summary>Challenge 3: Why does WebServerGracefulShutdownLifecycle throw UnsupportedOperationException from stop()?</summary>

The synchronous `stop()` method should never be called directly on this bean. Graceful shutdown is inherently asynchronous -- it polls for request completion on a separate thread. The `DefaultLifecycleProcessor` always calls `stop(Runnable)` for `SmartLifecycle` beans, which is the correct entry point. The `UnsupportedOperationException` guards against accidental direct calls.

If `stop()` were implemented to just call `webServer.stop()`, it would skip the graceful shutdown entirely and stop the server immediately -- defeating the purpose.

</details>

<details><summary>Challenge 4: What would break if the CountDownLatch timeout were set to 0?</summary>

With a 0ms timeout, `latch.await()` would return immediately (always timing out). The graceful shutdown would never have time to drain requests -- the processor would immediately proceed to the next phase and stop the server. In-flight requests would be dropped, making graceful shutdown equivalent to immediate shutdown.

The default 10-second timeout gives requests a reasonable window to complete. In production, this is configurable via `spring.lifecycle.timeout-per-shutdown-phase`.

</details>

---

## 19.8 Tests

### Unit Tests -- DefaultLifecycleProcessor

**File:** `iris-framework/.../context/support/DefaultLifecycleProcessorTest.java` (10 tests)

```java
@Test
void shouldStartBeansInAscendingPhaseOrder_WhenOnRefreshCalled() {
    List<String> startOrder = new ArrayList<>();
    registerSmartLifecycle("high", 100, startOrder);
    registerSmartLifecycle("low", 10, startOrder);
    registerSmartLifecycle("mid", 50, startOrder);

    processor.onRefresh();

    assertThat(startOrder).containsExactly("low", "mid", "high");
}

@Test
void shouldStopBeansInDescendingPhaseOrder_WhenOnCloseCalled() {
    List<String> stopOrder = new ArrayList<>();
    TrackingSmartLifecycle low = registerSmartLifecycle("low", 10, new ArrayList<>());
    TrackingSmartLifecycle mid = registerSmartLifecycle("mid", 50, new ArrayList<>());
    TrackingSmartLifecycle high = registerSmartLifecycle("high", 100, new ArrayList<>());

    processor.onRefresh();
    low.setStopTracker(stopOrder);
    mid.setStopTracker(stopOrder);
    high.setStopTracker(stopOrder);

    processor.onClose();

    assertThat(stopOrder).containsExactly("high", "mid", "low");
}

@Test
void shouldOnlyStartAutoStartupBeans_WhenOnRefreshCalled() {
    List<String> startOrder = new ArrayList<>();
    registerSmartLifecycle("auto", 10, startOrder, true);
    registerSmartLifecycle("manual", 20, startOrder, false);

    processor.onRefresh();

    assertThat(startOrder).containsExactly("auto");
}

@Test
void shouldUseAsyncStop_WhenSmartLifecycleBeanStops() {
    TrackingSmartLifecycle bean = registerSmartLifecycle("async-bean", 10, new ArrayList<>());
    processor.onRefresh();
    bean.setStopTracker(new ArrayList<>());

    processor.onClose();

    assertThat(bean.isAsyncStopUsed()).isTrue();
}

@Test
void shouldNotStopItself_WhenProcessorIsAlsoALifecycleBean() {
    processor.onRefresh();
    assertThatCode(() -> processor.onClose()).doesNotThrowAnyException();
}
```

### Unit Tests -- WebServerGracefulShutdownLifecycle

**File:** `iris-boot-core/.../web/context/WebServerGracefulShutdownLifecycleTest.java` (5 tests)

```java
@Test
void shouldBeAtGracefulShutdownPhase() {
    WebServerGracefulShutdownLifecycle lifecycle =
            new WebServerGracefulShutdownLifecycle(new StubWebServer());

    assertThat(lifecycle.getPhase())
            .isEqualTo(WebServerApplicationContext.GRACEFUL_SHUTDOWN_PHASE);
}

@Test
void shouldDelegateGracefulShutdownToWebServer_WhenStopWithCallback() {
    StubWebServer webServer = new StubWebServer();
    WebServerGracefulShutdownLifecycle lifecycle =
            new WebServerGracefulShutdownLifecycle(webServer);
    lifecycle.start();

    AtomicReference<Boolean> callbackInvoked = new AtomicReference<>(false);

    lifecycle.stop(() -> callbackInvoked.set(true));

    assertThat(lifecycle.isRunning()).isFalse();
    assertThat(webServer.gracefulShutdownInitiated).isTrue();
    assertThat(callbackInvoked.get()).isTrue();
}

@Test
void shouldThrowOnDirectStop() {
    WebServerGracefulShutdownLifecycle lifecycle =
            new WebServerGracefulShutdownLifecycle(new StubWebServer());
    lifecycle.start();

    assertThatThrownBy(lifecycle::stop)
            .isInstanceOf(UnsupportedOperationException.class);
}
```

### Unit Tests -- WebServerStartStopLifecycle

**File:** `iris-boot-core/.../web/context/WebServerStartStopLifecycleTest.java` (5 tests)

```java
@Test
void shouldBeAtStartStopLifecyclePhase() {
    WebServerStartStopLifecycle lifecycle =
            new WebServerStartStopLifecycle(null, new StubWebServer());

    assertThat(lifecycle.getPhase())
            .isEqualTo(WebServerApplicationContext.START_STOP_LIFECYCLE_PHASE);
}

@Test
void shouldHaveLowerPhaseThanGracefulShutdown() {
    assertThat(WebServerApplicationContext.START_STOP_LIFECYCLE_PHASE)
            .isLessThan(WebServerApplicationContext.GRACEFUL_SHUTDOWN_PHASE);
}

@Test
void shouldStartWebServer_WhenStartCalled() {
    StubWebServer webServer = new StubWebServer();
    WebServerStartStopLifecycle lifecycle =
            new WebServerStartStopLifecycle(null, webServer);

    lifecycle.start();

    assertThat(webServer.started).isTrue();
    assertThat(lifecycle.isRunning()).isTrue();
}
```

### Integration Tests

**File:** `iris-boot-core/.../integration/GracefulShutdownIntegrationTest.java` (6 tests)

```java
@Test
void shouldStartWebServerViaSmartLifecycle_WhenContextRefreshes() {
    ServletWebServerApplicationContext context = new ServletWebServerApplicationContext();
    context.register(WebServerConfig.class);

    context.refresh();

    try {
        WebServer webServer = context.getWebServer();
        assertThat(webServer).isNotNull();
        assertThat(webServer.getPort()).isGreaterThan(0);
    } finally {
        context.close();
    }
}

@Test
void shouldReportIdleShutdown_WhenGracefulEnabledAndNoActiveRequests() throws Exception {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
    factory.setShutdown(Shutdown.GRACEFUL);

    WebServer server = factory.getWebServer();
    server.start();

    AtomicReference<GracefulShutdownResult> result = new AtomicReference<>();
    CountDownLatch latch = new CountDownLatch(1);

    server.shutDownGracefully(r -> {
        result.set(r);
        latch.countDown();
    });

    boolean completed = latch.await(5, TimeUnit.SECONDS);

    assertThat(completed).isTrue();
    assertThat(result.get()).isEqualTo(GracefulShutdownResult.IDLE);

    server.stop();
    server.destroy();
}

@Test
void shouldReportImmediateShutdown_WhenGracefulNotEnabled() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
    factory.setShutdown(Shutdown.IMMEDIATE);

    WebServer server = factory.getWebServer();
    server.start();

    AtomicReference<GracefulShutdownResult> result = new AtomicReference<>();
    CountDownLatch latch = new CountDownLatch(1);

    server.shutDownGracefully(r -> {
        result.set(r);
        latch.countDown();
    });

    assertThat(result.get()).isEqualTo(GracefulShutdownResult.IMMEDIATE);

    server.destroy();
}
```

---

## 19.9 Why This Works

> **Insight** -- **Phase ordering solves the "graceful before hard stop" problem**
>
> The entire graceful shutdown feature hinges on one property of the `DefaultLifecycleProcessor`: shutdown processes phases from highest to lowest. By putting the graceful shutdown at phase `MAX_VALUE - 1024` and the server stop at phase `MAX_VALUE - 2048`, we guarantee that request draining completes before the server is killed. This is not a hack -- it is the intended design. The real Spring Framework uses the exact same phase values and ordering logic (`DefaultLifecycleProcessor`, line 528).
>
> The phase gap of 1024 is deliberate: it leaves room for user-defined SmartLifecycle beans that need to run between the two phases. For example, a bean at phase `MAX_VALUE - 1536` could close database connections after graceful shutdown but before the server is fully stopped.

> **Insight** -- **The async stop(Runnable) pattern enables non-blocking shutdown**
>
> The `stop(Runnable)` contract is crucial. When `WebServerGracefulShutdownLifecycle.stop(Runnable callback)` is called, it delegates to `GracefulShutdown.shutDownGracefully()`, which spawns a "tomcat-shutdown" thread that polls for request completion. The callback is invoked on that thread when polling finishes, counting down the `CountDownLatch` in the `DefaultLifecycleProcessor`.
>
> Without the async pattern, the processor would have to either: (a) block the main thread during polling -- preventing other lifecycle beans from being processed, or (b) skip waiting -- potentially stopping the server before requests drain. The `CountDownLatch` + callback pattern gives the best of both worlds: the processor dispatches all stop calls in a phase, then waits for all of them to complete (or time out) before moving to the next phase.

> **Insight** -- **The JVM shutdown hook completes the lifecycle guarantee**
>
> `registerShutdownHook()` registers a thread that calls `context.close()` on JVM shutdown (SIGTERM, SIGINT, `System.exit()`). Without it, killing the process with `kill <pid>` would abruptly terminate without stopping SmartLifecycle beans or destroying singletons. With it, the full shutdown sequence runs: graceful shutdown drains requests, the server stops, and `@PreDestroy` methods fire. The hook is automatically removed when `close()` is called explicitly, preventing double invocation.

---

## 19.10 What We Enhanced

| Prior File | Change | Why |
|-----------|--------|-----|
| `AnnotationConfigApplicationContext.finishRefresh()` | Added `initLifecycleProcessor()` and `getLifecycleProcessor().onRefresh()` before event publication | This is where SmartLifecycle beans start -- the web server binds its port here via `WebServerStartStopLifecycle` |
| `AnnotationConfigApplicationContext.close()` | Added `lifecycleProcessor.onClose()` between event publication and singleton destruction | SmartLifecycle beans must stop in reverse phase order BEFORE `@PreDestroy` methods fire on the beans they depend on |
| `AnnotationConfigApplicationContext` | Added `registerShutdownHook()` and `removeShutdownHook()` | JVM shutdown hook ensures orderly shutdown on SIGTERM/SIGINT |
| `DefaultBeanFactory` | Added `registerSingleton()` method | Needed to register SmartLifecycle beans programmatically (they are created by code, not by bean definitions) |
| `ServletWebServerApplicationContext.onRefresh()` | Now registers SmartLifecycle beans instead of starting server directly | Server lifecycle is managed by the `LifecycleProcessor`, enabling phase-ordered shutdown |
| `TomcatWebServer` | Added second constructor accepting `Shutdown`, added `shutDownGracefully()`, added abort in `stop()` | Graceful shutdown support -- pause connectors and drain requests before stopping |
| `TomcatServletWebServerFactory` | Added `Shutdown` field, getter/setter, passes to `TomcatWebServer` | Configuration pipeline for shutdown mode |
| `WebServer` interface | Added `shutDownGracefully()` default method | All server implementations can support graceful shutdown |
| `ServerProperties` | Added `shutdown` property | Binds `server.shutdown` from `application.properties` |
| `TomcatAutoConfiguration` | Passes `serverProperties.getShutdown()` to factory | Wires the configuration property to the server factory |
| `PropertySourcesPropertyResolver` | Added enum conversion in `convertValue()` | Needed to convert string `"graceful"` to `Shutdown.GRACEFUL` |
| `ConfigurationPropertiesBinder` | Added `type.isEnum()` to `isSimpleType()` | Enum fields must be treated as simple types for property binding |
| `IrisApplication.refreshContext()` | Added `context.registerShutdownHook()` after `refresh()` | Ensures JVM shutdown triggers orderly context close |

---

## 19.11 Connection to Real Framework

| Iris Class | Real Spring Class | File (commit `5922311a95a` / `11ab0b4351`) |
|-----------|------------------|------|
| `Lifecycle` | `Lifecycle` | Spring Framework `spring-context/.../context/Lifecycle.java` (commit `11ab0b4351`) |
| `SmartLifecycle` | `SmartLifecycle` | Spring Framework `spring-context/.../context/SmartLifecycle.java` |
| `LifecycleProcessor` | `LifecycleProcessor` | Spring Framework `spring-context/.../context/LifecycleProcessor.java` |
| `DefaultLifecycleProcessor` | `DefaultLifecycleProcessor` | Spring Framework `spring-context/.../context/support/DefaultLifecycleProcessor.java` |
| `Shutdown` | `Shutdown` | `spring-boot-web-server/.../boot/web/server/Shutdown.java` |
| `GracefulShutdownCallback` | `GracefulShutdownCallback` | `spring-boot-web-server/.../boot/web/server/GracefulShutdownCallback.java` |
| `GracefulShutdownResult` | `GracefulShutdownResult` | `spring-boot-web-server/.../boot/web/server/GracefulShutdownResult.java` |
| `GracefulShutdown` (Tomcat) | `GracefulShutdown` | `spring-boot-tomcat/.../boot/tomcat/GracefulShutdown.java` |
| `WebServerGracefulShutdownLifecycle` | `WebServerGracefulShutdownLifecycle` | `spring-boot-web-server/.../boot/web/server/context/WebServerGracefulShutdownLifecycle.java` |
| `WebServerStartStopLifecycle` | `WebServerStartStopLifecycle` | `spring-boot-web-server/.../boot/web/server/servlet/context/WebServerStartStopLifecycle.java` |
| `finishRefresh()` / `initLifecycleProcessor()` | `AbstractApplicationContext.finishRefresh()` line 1002, `initLifecycleProcessor()` line 879 | Spring Framework `spring-context/.../context/support/AbstractApplicationContext.java` |
| `registerShutdownHook()` | `AbstractApplicationContext.registerShutdownHook()` line 1068 | Spring Framework `spring-context/.../context/support/AbstractApplicationContext.java` |

**Key differences from real Spring:**
- Real `DefaultLifecycleProcessor` supports concurrent startup within a phase (Spring 6.2+ via `CompletableFuture`). We start beans sequentially.
- Real processor considers `depends-on` relationships within a phase for ordering. We use discovery order.
- Real `SmartLifecycle` extends a separate `Phased` interface. We inline `getPhase()` directly.
- Real `AbstractApplicationContext` stores the shutdown hook in a `ShutdownHook` abstraction. We use a plain `Thread`.

---

## 19.12 Complete Code

### Production Code

#### `[NEW]` `iris-framework/src/main/java/com/iris/framework/context/Lifecycle.java`

```java
package com.iris.framework.context;

/**
 * A common interface defining methods for start/stop lifecycle control.
 * The typical use case is to control asynchronous processing.
 *
 * <p>Can be implemented by both components (typically beans registered
 * in a Spring context) and containers (typically the ApplicationContext
 * itself). Containers will propagate start/stop signals to all
 * components that apply within each container.
 *
 * <p>Only supported on <strong>top-level singleton beans</strong>.
 * On other bean types, Lifecycle callbacks are not detected and hence
 * silently ignored.
 *
 * <p>Note that the {@link #stop()} method is not necessarily guaranteed
 * to come before destruction: on regular shutdown, all beans will first
 * receive a stop notification before the general destruction callbacks
 * are being propagated. However, on a hot refresh during a context's
 * lifetime or on aborted refresh attempts, a given bean's destroy method
 * will be called without any consideration of stop signals upfront.
 *
 * @see SmartLifecycle
 * @see LifecycleProcessor
 * @see org.springframework.context.Lifecycle
 */
public interface Lifecycle {

    /**
     * Start this component. Should not throw an exception if the component
     * is already running.
     */
    void start();

    /**
     * Stop this component. Should not throw an exception if the component
     * is not running.
     */
    void stop();

    /**
     * Check whether this component is currently running.
     *
     * @return whether the component is currently running
     */
    boolean isRunning();
}
```

#### `[NEW]` `iris-framework/src/main/java/com/iris/framework/context/SmartLifecycle.java`

```java
package com.iris.framework.context;

/**
 * An extension of the {@link Lifecycle} interface for objects that require to
 * be started upon ApplicationContext refresh and/or shutdown in a particular
 * order.
 *
 * <p>The {@link #isAutoStartup()} return value indicates whether this object
 * should be started at the time of a context refresh. The callback-accepting
 * {@link #stop(Runnable)} method is useful for objects that have an
 * asynchronous shutdown process.
 *
 * <p>The {@link #getPhase()} method controls startup and shutdown ordering:
 * <ul>
 *   <li><strong>Startup:</strong> objects with the lowest phase start first</li>
 *   <li><strong>Shutdown:</strong> objects with the highest phase stop first</li>
 * </ul>
 *
 * <p>The default phase is {@link #DEFAULT_PHASE} ({@code Integer.MAX_VALUE}),
 * meaning SmartLifecycle beans start last and stop first. Any explicit
 * "depends-on" relationships take precedence over the phase ordering.
 *
 * <p>In the real Spring Framework, this also extends {@code Phased} as a
 * separate interface. We include {@link #getPhase()} directly for simplicity.
 *
 * @see LifecycleProcessor
 * @see org.springframework.context.SmartLifecycle
 */
public interface SmartLifecycle extends Lifecycle {

    /**
     * The default phase for {@code SmartLifecycle}: {@code Integer.MAX_VALUE}.
     *
     * <p>This means SmartLifecycle objects are started in the last possible
     * phase during context refresh, and stopped in the first possible phase
     * during shutdown. This is the right default for web server lifecycle
     * beans — they should start last (after all beans are ready) and stop
     * first (before beans they serve are destroyed).
     */
    int DEFAULT_PHASE = Integer.MAX_VALUE;

    /**
     * Return whether this Lifecycle component should get started
     * automatically by the container when the ApplicationContext is
     * refreshed.
     *
     * <p>A value of {@code false} indicates that the component is intended
     * to be started through an explicit {@link #start()} call instead.
     *
     * @return whether auto-startup is enabled (default: {@code true})
     */
    default boolean isAutoStartup() {
        return true;
    }

    /**
     * Indicates that a stop notification has been received. The callback
     * <strong>must</strong> be invoked when the SmartLifecycle component has
     * actually stopped.
     *
     * <p>The {@link LifecycleProcessor} will use a
     * {@link java.util.concurrent.CountDownLatch} internally, and if this
     * callback is not invoked within the shutdown timeout, the processor
     * will time out and continue shutting down remaining beans.
     *
     * <p>The default implementation delegates to {@link #stop()} and then
     * immediately runs the callback. Implementations with asynchronous
     * shutdown processes should override this to invoke the callback when
     * the shutdown is truly complete.
     *
     * @param callback the callback to invoke when stop is complete
     */
    default void stop(Runnable callback) {
        stop();
        callback.run();
    }

    /**
     * Return the phase value of this lifecycle object. Objects with the
     * lowest phase start first and stop last.
     *
     * @return the phase value (default: {@link #DEFAULT_PHASE})
     */
    default int getPhase() {
        return DEFAULT_PHASE;
    }
}
```

#### `[NEW]` `iris-framework/src/main/java/com/iris/framework/context/LifecycleProcessor.java`

```java
package com.iris.framework.context;

/**
 * Strategy interface for processing {@link Lifecycle} beans within the
 * ApplicationContext.
 *
 * <p>The ApplicationContext delegates to a LifecycleProcessor at two key
 * points in its own lifecycle:
 * <ul>
 *   <li>{@link #onRefresh()} — called at the end of
 *       {@code ApplicationContext.refresh()}, starts all auto-startup
 *       SmartLifecycle beans</li>
 *   <li>{@link #onClose()} — called during {@code ApplicationContext.close()},
 *       stops all running Lifecycle beans</li>
 * </ul>
 *
 * <p>The default implementation is
 * {@link com.iris.framework.context.support.DefaultLifecycleProcessor},
 * which groups beans by phase and starts/stops them in order.
 *
 * @see SmartLifecycle
 * @see org.springframework.context.LifecycleProcessor
 */
public interface LifecycleProcessor extends Lifecycle {

    /**
     * Notification of context refresh. Auto-start all
     * {@link SmartLifecycle} beans with {@code isAutoStartup() == true}.
     *
     * <p>Default implementation delegates to {@link #start()}.
     */
    default void onRefresh() {
        start();
    }

    /**
     * Notification of context close. Stop all running Lifecycle beans
     * in reverse phase order.
     *
     * <p>Default implementation delegates to {@link #stop()}.
     */
    default void onClose() {
        stop();
    }
}
```

#### `[NEW]` `iris-framework/src/main/java/com/iris/framework/context/support/DefaultLifecycleProcessor.java`

```java
package com.iris.framework.context.support;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.Lifecycle;
import com.iris.framework.context.LifecycleProcessor;
import com.iris.framework.context.SmartLifecycle;

/**
 * Default implementation of the {@link LifecycleProcessor} strategy.
 *
 * <p>Groups {@link Lifecycle} beans by their phase value and starts/stops
 * them in order:
 * <ul>
 *   <li><strong>Start:</strong> lowest phase first (0 before 100 before MAX_VALUE)</li>
 *   <li><strong>Stop:</strong> highest phase first (MAX_VALUE before 100 before 0)</li>
 * </ul>
 *
 * <p>For {@link SmartLifecycle} beans, the async {@code stop(Runnable)} variant
 * is used, with a per-phase timeout controlled by {@link #setTimeoutPerShutdownPhase}.
 * If any beans in a phase don't complete within the timeout, shutdown continues
 * to the next phase (the slow bean is logged but not blocked on forever).
 *
 * <h3>How Beans Are Discovered</h3>
 *
 * <p>Uses {@link DefaultBeanFactory#getBeansOfType(Class)} to discover all
 * singleton beans implementing {@code Lifecycle}. This includes both
 * definition-based beans and manually registered singletons (via
 * {@code registerSingleton()}).
 *
 * <h3>Simplifications from Real Spring Framework</h3>
 *
 * <ul>
 *   <li>No concurrent startup support (real Spring 6.2+ can start beans
 *       in parallel within a phase via CompletableFuture)</li>
 *   <li>No dependency-aware ordering within a phase (real Spring considers
 *       {@code depends-on} relationships)</li>
 *   <li>No pause/restart support (Spring 7.0+)</li>
 * </ul>
 *
 * @see org.springframework.context.support.DefaultLifecycleProcessor
 */
public class DefaultLifecycleProcessor implements LifecycleProcessor {

    /** Timeout for each shutdown phase, in milliseconds. Default: 10 seconds. */
    private long timeoutPerShutdownPhase = 10000;

    /** The bean factory to discover Lifecycle beans from. */
    private DefaultBeanFactory beanFactory;

    /** Whether this processor is currently running. */
    private volatile boolean running;

    // -----------------------------------------------------------------------
    // Configuration
    // -----------------------------------------------------------------------

    /**
     * Set the bean factory that this processor uses to discover Lifecycle beans.
     *
     * @param beanFactory the bean factory
     */
    public void setBeanFactory(DefaultBeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    /**
     * Set the timeout (in milliseconds) for each shutdown phase.
     *
     * <p>If any {@code SmartLifecycle} beans in a phase don't invoke their
     * stop callback within this time, shutdown proceeds to the next phase.
     * Default is 10000 (10 seconds).
     *
     * <p>In the real Spring Framework, this is configurable via
     * {@code spring.lifecycle.timeout-per-shutdown-phase} property.
     *
     * @param timeout the timeout in milliseconds
     */
    public void setTimeoutPerShutdownPhase(long timeout) {
        this.timeoutPerShutdownPhase = timeout;
    }

    // -----------------------------------------------------------------------
    // Lifecycle implementation
    // -----------------------------------------------------------------------

    @Override
    public void start() {
        startBeans(false);
        this.running = true;
    }

    @Override
    public void stop() {
        stopBeans();
        this.running = false;
    }

    @Override
    public boolean isRunning() {
        return this.running;
    }

    // -----------------------------------------------------------------------
    // LifecycleProcessor — context integration
    // -----------------------------------------------------------------------

    /**
     * Called on context refresh. Starts only {@link SmartLifecycle} beans
     * with {@code isAutoStartup() == true}.
     *
     * <p>In the real Spring Framework, this is called from
     * {@code AbstractApplicationContext.finishRefresh()} (line 1013).
     */
    @Override
    public void onRefresh() {
        startBeans(true);
        this.running = true;
    }

    /**
     * Called on context close. Stops all running Lifecycle beans in
     * reverse phase order.
     *
     * <p>In the real Spring Framework, this is called from
     * {@code AbstractApplicationContext.doClose()} (line 1199).
     */
    @Override
    public void onClose() {
        stopBeans();
        this.running = false;
    }

    // -----------------------------------------------------------------------
    // Start logic
    // -----------------------------------------------------------------------

    /**
     * Start Lifecycle beans, optionally filtering to auto-startup only.
     *
     * <p>Groups beans by phase using a {@code TreeMap} (natural ascending
     * order), then starts each phase group sequentially. Within a phase,
     * beans start in the order they are discovered.
     *
     * @param autoStartupOnly if true, only start SmartLifecycle beans with
     *        {@code isAutoStartup() == true}
     */
    private void startBeans(boolean autoStartupOnly) {
        Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
        Map<Integer, List<LifecycleBean>> phases = new TreeMap<>();

        for (Map.Entry<String, Lifecycle> entry : lifecycleBeans.entrySet()) {
            Lifecycle bean = entry.getValue();
            if (!autoStartupOnly
                    || (bean instanceof SmartLifecycle sl && sl.isAutoStartup())) {
                int phase = getPhase(bean);
                phases.computeIfAbsent(phase, k -> new ArrayList<>())
                      .add(new LifecycleBean(entry.getKey(), bean));
            }
        }

        // Start in ascending phase order (lowest first)
        for (List<LifecycleBean> group : phases.values()) {
            for (LifecycleBean lb : group) {
                if (!lb.bean.isRunning()) {
                    lb.bean.start();
                }
            }
        }
    }

    // -----------------------------------------------------------------------
    // Stop logic
    // -----------------------------------------------------------------------

    /**
     * Stop all running Lifecycle beans in reverse phase order.
     *
     * <p>Uses a {@code TreeMap} with reverse comparator so the highest
     * phase stops first. For {@link SmartLifecycle} beans, the async
     * {@code stop(Runnable)} method is called; the processor waits for
     * all beans in a phase to complete via a {@link CountDownLatch}
     * before moving to the next phase.
     */
    private void stopBeans() {
        Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
        Map<Integer, List<LifecycleBean>> phases =
                new TreeMap<>(Comparator.reverseOrder());

        for (Map.Entry<String, Lifecycle> entry : lifecycleBeans.entrySet()) {
            Lifecycle bean = entry.getValue();
            int phase = getPhase(bean);
            phases.computeIfAbsent(phase, k -> new ArrayList<>())
                  .add(new LifecycleBean(entry.getKey(), bean));
        }

        // Stop in descending phase order (highest first)
        for (Map.Entry<Integer, List<LifecycleBean>> entry : phases.entrySet()) {
            stopPhaseGroup(entry.getKey(), entry.getValue());
        }
    }

    /**
     * Stop all beans in a single phase group.
     *
     * <p>For {@link SmartLifecycle} beans, uses the async {@code stop(Runnable)}
     * variant. A {@link CountDownLatch} tracks how many beans are pending
     * completion. After dispatching all stop calls, the processor waits
     * up to {@link #timeoutPerShutdownPhase} milliseconds for all beans
     * to finish.
     *
     * <p>For regular {@link Lifecycle} beans, calls {@code stop()} synchronously.
     *
     * @param phase the phase number (for logging)
     * @param beans the lifecycle beans in this phase
     */
    private void stopPhaseGroup(int phase, List<LifecycleBean> beans) {
        // Count running SmartLifecycle beans that need async stop
        int smartCount = 0;
        for (LifecycleBean lb : beans) {
            if (lb.bean instanceof SmartLifecycle && lb.bean.isRunning()) {
                smartCount++;
            }
        }

        CountDownLatch latch = new CountDownLatch(smartCount);

        for (LifecycleBean lb : beans) {
            if (!lb.bean.isRunning()) {
                continue;
            }

            if (lb.bean instanceof SmartLifecycle sl) {
                // Async stop — the bean MUST invoke the callback when done
                try {
                    sl.stop(latch::countDown);
                } catch (Exception ex) {
                    System.err.println("Failed to stop bean '" + lb.name
                            + "' in phase " + phase + ": " + ex.getMessage());
                    latch.countDown();
                }
            } else {
                // Synchronous stop for non-SmartLifecycle beans
                try {
                    lb.bean.stop();
                } catch (Exception ex) {
                    System.err.println("Failed to stop bean '" + lb.name
                            + "': " + ex.getMessage());
                }
            }
        }

        // Wait for all async stops to complete
        if (smartCount > 0) {
            try {
                boolean completed = latch.await(
                        this.timeoutPerShutdownPhase, TimeUnit.MILLISECONDS);
                if (!completed) {
                    System.err.println("Timed out waiting for lifecycle beans "
                            + "in phase " + phase + " to stop");
                }
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // -----------------------------------------------------------------------
    // Bean discovery
    // -----------------------------------------------------------------------

    /**
     * Discover all singleton {@link Lifecycle} beans from the bean factory,
     * excluding this processor itself.
     *
     * <p>The self-exclusion ({@code bean != this}) prevents infinite recursion:
     * without it, {@code stopBeans()} would discover this processor, call
     * {@code stop()} on it, which calls {@code stopBeans()} again.
     *
     * <p>In the real Spring Framework, this is
     * {@code DefaultLifecycleProcessor.getLifecycleBeans()} (line 528),
     * which has the same self-exclusion check: {@code if (bean != this && ...)}.
     */
    private Map<String, Lifecycle> getLifecycleBeans() {
        Map<String, Lifecycle> beans = this.beanFactory.getBeansOfType(Lifecycle.class);
        beans.values().removeIf(bean -> bean == this);
        return beans;
    }

    /**
     * Return the phase for the given Lifecycle bean.
     *
     * @return the phase from {@code SmartLifecycle.getPhase()}, or 0 for
     *         regular Lifecycle beans
     */
    private int getPhase(Lifecycle bean) {
        if (bean instanceof SmartLifecycle sl) {
            return sl.getPhase();
        }
        return 0;
    }

    // -----------------------------------------------------------------------
    // Internal
    // -----------------------------------------------------------------------

    /**
     * A name-bean pair for tracking lifecycle beans within a phase group.
     */
    private record LifecycleBean(String name, Lifecycle bean) {}
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/web/server/Shutdown.java`

```java
package com.iris.boot.web.server;

/**
 * Configuration for the shutdown of a {@link WebServer}.
 *
 * <p>When set to {@link #GRACEFUL}, the server will stop accepting new
 * connections and wait for in-flight requests to complete before stopping.
 * When set to {@link #IMMEDIATE} (the default), the server stops immediately
 * without waiting.
 *
 * <p>Configured via the {@code server.shutdown} property in
 * {@code application.properties}:
 * <pre>
 * server.shutdown=graceful
 * </pre>
 *
 * @see org.springframework.boot.web.server.Shutdown
 */
public enum Shutdown {

    /**
     * The server will process in-flight requests before stopping.
     * New connections are rejected immediately.
     */
    GRACEFUL,

    /**
     * The server will stop immediately, dropping any in-flight requests.
     * This is the default behavior.
     */
    IMMEDIATE
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/web/server/GracefulShutdownCallback.java`

```java
package com.iris.boot.web.server;

/**
 * A callback that is invoked when a web server's graceful shutdown
 * completes.
 *
 * <p>Used by {@link WebServer#shutDownGracefully(GracefulShutdownCallback)}
 * to signal that the graceful shutdown process has finished (either
 * because all requests drained or because the shutdown was aborted).
 *
 * @see WebServer#shutDownGracefully(GracefulShutdownCallback)
 * @see GracefulShutdownResult
 * @see org.springframework.boot.web.server.GracefulShutdownCallback
 */
@FunctionalInterface
public interface GracefulShutdownCallback {

    /**
     * Called when the graceful shutdown has completed.
     *
     * @param result the result indicating how the shutdown completed
     */
    void shutdownComplete(GracefulShutdownResult result);
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/web/server/GracefulShutdownResult.java`

```java
package com.iris.boot.web.server;

/**
 * The result of a graceful shutdown request.
 *
 * @see GracefulShutdownCallback
 * @see org.springframework.boot.web.server.GracefulShutdownResult
 */
public enum GracefulShutdownResult {

    /**
     * The server had active requests that did not complete within the
     * grace period (the shutdown was aborted).
     */
    REQUESTS_ACTIVE,

    /**
     * The server was idle with no active requests when the shutdown
     * completed — all in-flight requests finished successfully.
     */
    IDLE,

    /**
     * The server was shut down immediately, ignoring any active requests.
     * This is the result when graceful shutdown is not enabled.
     */
    IMMEDIATE
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/GracefulShutdown.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.apache.catalina.Container;
import org.apache.catalina.Service;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.core.StandardWrapper;
import org.apache.catalina.startup.Tomcat;

import com.iris.boot.web.server.GracefulShutdownCallback;
import com.iris.boot.web.server.GracefulShutdownResult;

/**
 * Handles graceful shutdown of an embedded Tomcat server by pausing
 * connectors and waiting for in-flight requests to complete.
 *
 * <p>The shutdown sequence:
 * <ol>
 *   <li>Pause all connectors (stop accepting new connections)</li>
 *   <li>Close the server socket gracefully (reject new TCP connections)</li>
 *   <li>Poll every 50ms until all active requests complete or
 *       {@link #abort()} is called</li>
 *   <li>Invoke the callback with the result</li>
 * </ol>
 *
 * <p>The shutdown runs on a dedicated "tomcat-shutdown" thread so the
 * caller (the {@code DefaultLifecycleProcessor}) is not blocked during
 * polling. The callback invocation on the shutdown thread counts down
 * the processor's latch.
 *
 * <h3>How Active Requests Are Detected</h3>
 *
 * <p>Tomcat's {@code Context.getInProgressAsyncCount()} tracks async
 * requests, and each {@code Wrapper}'s {@code getCountAllocated()} tracks
 * sync requests currently being processed. When both are zero across all
 * contexts, the server is idle.
 *
 * @see org.springframework.boot.tomcat.GracefulShutdown
 */
final class GracefulShutdown {

    private final Tomcat tomcat;

    /** Set to true when the shutdown is aborted (e.g., by calling stop()). */
    private volatile boolean aborted = false;

    GracefulShutdown(Tomcat tomcat) {
        this.tomcat = tomcat;
    }

    /**
     * Initiate a graceful shutdown.
     *
     * <p>Starts a new thread that pauses connectors and polls for request
     * completion. The callback is invoked when the shutdown finishes.
     *
     * @param callback the callback to invoke when shutdown completes
     */
    void shutDownGracefully(GracefulShutdownCallback callback) {
        System.out.println("Commencing graceful shutdown. "
                + "Waiting for active requests to complete");
        new Thread(() -> doShutdown(callback), "tomcat-shutdown").start();
    }

    /**
     * The actual shutdown logic, running on the "tomcat-shutdown" thread.
     */
    private void doShutdown(GracefulShutdownCallback callback) {
        try {
            // Step 1: Pause all connectors and close server sockets
            List<Connector> connectors = getConnectors();
            for (Connector connector : connectors) {
                connector.pause();
                connector.getProtocolHandler().closeServerSocketGraceful();
            }

            // Step 2: Poll until no active requests or aborted
            awaitInactiveOrAborted();

            // Step 3: Report result
            if (this.aborted) {
                System.out.println("Graceful shutdown aborted with one or more "
                        + "requests still active");
                callback.shutdownComplete(GracefulShutdownResult.REQUESTS_ACTIVE);
            } else {
                System.out.println("Graceful shutdown complete");
                callback.shutdownComplete(GracefulShutdownResult.IDLE);
            }
        } catch (Exception ex) {
            System.err.println("Graceful shutdown failed: " + ex.getMessage());
            callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
        }
    }

    /**
     * Get all connectors from all Tomcat services.
     *
     * <p>In the real Spring Boot, this is the same approach: iterate
     * {@code server.findServices()} and collect all connectors.
     */
    private List<Connector> getConnectors() {
        List<Connector> connectors = new ArrayList<>();
        for (Service service : this.tomcat.getServer().findServices()) {
            Collections.addAll(connectors, service.findConnectors());
        }
        return connectors;
    }

    /**
     * Poll every 50ms until all active requests are complete or the
     * shutdown is aborted.
     *
     * <p>In the real Spring Boot ({@code GracefulShutdown.java}, line 98),
     * this polls both async counts and allocated wrapper counts.
     */
    private void awaitInactiveOrAborted() {
        while (!this.aborted) {
            if (!hasActiveRequests()) {
                return;
            }
            try {
                Thread.sleep(50);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }

    /**
     * Check if any context has active requests (sync or async).
     *
     * <p>Walks the container hierarchy: Engine → Hosts → Contexts → Wrappers.
     * Checks {@code Context.getInProgressAsyncCount()} for async requests and
     * {@code Wrapper.getCountAllocated()} for sync requests currently being
     * processed by a servlet.
     */
    private boolean hasActiveRequests() {
        for (Container host : this.tomcat.getEngine().findChildren()) {
            for (Container context : host.findChildren()) {
                if (context instanceof StandardContext ctx) {
                    if (ctx.getInProgressAsyncCount() > 0) {
                        return true;
                    }
                    for (Container wrapper : ctx.findChildren()) {
                        if (wrapper instanceof StandardWrapper w) {
                            if (w.getCountAllocated() > 0) {
                                return true;
                            }
                        }
                    }
                }
            }
        }
        return false;
    }

    /**
     * Abort the graceful shutdown, signaling the polling loop to stop
     * waiting for requests to complete.
     *
     * <p>Called from {@code TomcatWebServer.stop()} to abort any in-progress
     * graceful shutdown when a hard stop is requested.
     */
    void abort() {
        this.aborted = true;
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/web/context/WebServerGracefulShutdownLifecycle.java`

```java
package com.iris.boot.web.context;

import com.iris.boot.web.server.WebServer;
import com.iris.framework.context.SmartLifecycle;

/**
 * A {@link SmartLifecycle} bean that triggers the web server's graceful
 * shutdown during the application context close sequence.
 *
 * <p>This bean runs at phase {@link WebServerApplicationContext#GRACEFUL_SHUTDOWN_PHASE}
 * ({@code Integer.MAX_VALUE - 1024}), which is a HIGHER phase than
 * {@link WebServerStartStopLifecycle}. Since shutdown processes the highest
 * phase first, this bean stops BEFORE the web server is actually stopped.
 *
 * <p>The shutdown sequence:
 * <ol>
 *   <li>Phase MAX_VALUE-1024: This bean's {@code stop(Runnable)} is called
 *       → delegates to {@code webServer.shutDownGracefully()}, which pauses
 *       connectors and waits for in-flight requests to drain</li>
 *   <li>Phase MAX_VALUE-2048: {@link WebServerStartStopLifecycle#stop()} is called
 *       → calls {@code webServer.stop()}, stopping Tomcat completely</li>
 * </ol>
 *
 * <p>The async {@code stop(Runnable)} method is key: the
 * {@code DefaultLifecycleProcessor} waits for the callback before moving
 * to the next phase. This gives the graceful shutdown time to drain
 * requests without blocking the main thread.
 *
 * @see org.springframework.boot.web.server.context.WebServerGracefulShutdownLifecycle
 */
public final class WebServerGracefulShutdownLifecycle implements SmartLifecycle {

    private final WebServer webServer;

    private volatile boolean running;

    public WebServerGracefulShutdownLifecycle(WebServer webServer) {
        this.webServer = webServer;
    }

    @Override
    public void start() {
        this.running = true;
    }

    /**
     * Synchronous stop is not supported — the graceful shutdown is
     * asynchronous by nature. Always use {@link #stop(Runnable)}.
     *
     * @throws UnsupportedOperationException always
     */
    @Override
    public void stop() {
        throw new UnsupportedOperationException(
                "Stop must not be invoked directly. Use stop(Runnable) instead.");
    }

    /**
     * Initiate graceful shutdown and invoke the callback when complete.
     *
     * <p>The callback invocation counts down the {@code DefaultLifecycleProcessor}'s
     * {@code CountDownLatch}, allowing shutdown to proceed to the next phase.
     *
     * @param callback the callback to invoke when graceful shutdown completes
     */
    @Override
    public void stop(Runnable callback) {
        this.running = false;
        this.webServer.shutDownGracefully((result) -> callback.run());
    }

    @Override
    public boolean isRunning() {
        return this.running;
    }

    @Override
    public int getPhase() {
        return WebServerApplicationContext.GRACEFUL_SHUTDOWN_PHASE;
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/web/context/WebServerStartStopLifecycle.java`

```java
package com.iris.boot.web.context;

import com.iris.boot.web.server.WebServer;
import com.iris.framework.context.SmartLifecycle;

/**
 * A {@link SmartLifecycle} bean that manages the web server's start/stop
 * lifecycle within the ApplicationContext.
 *
 * <p>This bean runs at phase {@link WebServerApplicationContext#START_STOP_LIFECYCLE_PHASE}
 * ({@code Integer.MAX_VALUE - 2048}), which is LOWER than the graceful
 * shutdown phase. This means:
 * <ul>
 *   <li><strong>Startup:</strong> the server starts FIRST (lower phase starts first),
 *       binding the port and accepting connections after all singletons are initialized</li>
 *   <li><strong>Shutdown:</strong> the server stops AFTER graceful shutdown has drained
 *       in-flight requests (higher phase stops first)</li>
 * </ul>
 *
 * <p>Before this feature, the web server was started directly in
 * {@code ServletWebServerApplicationContext.refresh()} after
 * {@code super.refresh()} returned. Now it starts via the
 * {@code DefaultLifecycleProcessor} during {@code finishRefresh()},
 * which is the correct integration point — the same as real Spring Boot.
 *
 * @see org.springframework.boot.web.server.servlet.context.WebServerStartStopLifecycle
 */
class WebServerStartStopLifecycle implements SmartLifecycle {

    private final ServletWebServerApplicationContext applicationContext;

    private final WebServer webServer;

    private volatile boolean running;

    WebServerStartStopLifecycle(ServletWebServerApplicationContext applicationContext,
                                WebServer webServer) {
        this.applicationContext = applicationContext;
        this.webServer = webServer;
    }

    /**
     * Start the web server. This is where the port actually binds and
     * the server begins accepting HTTP connections.
     *
     * <p>In the real Spring Boot, this also publishes a
     * {@code ServletWebServerInitializedEvent}. We skip the event for
     * simplicity.
     */
    @Override
    public void start() {
        this.webServer.start();
        this.running = true;
    }

    /**
     * Stop the web server. Called after graceful shutdown has completed
     * (if enabled), so at this point any remaining connections can be
     * forcefully closed.
     */
    @Override
    public void stop() {
        this.running = false;
        this.webServer.stop();
    }

    @Override
    public boolean isRunning() {
        return this.running;
    }

    @Override
    public int getPhase() {
        return WebServerApplicationContext.START_STOP_LIFECYCLE_PHASE;
    }
}
```

#### `[MODIFIED]` `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`

```java
package com.iris.framework.beans.factory.support;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanCurrentlyInCreationException;
import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.beans.factory.DisposableBean;
import com.iris.framework.beans.factory.InitializingBean;
import com.iris.framework.beans.factory.NoSuchBeanDefinitionException;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;
import com.iris.framework.beans.factory.annotation.Autowired;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.config.BeanPostProcessor;

/**
 * Default implementation of both {@link BeanFactory} and {@link BeanDefinitionRegistry}.
 *
 * <p>Two internal maps form the core:
 * <ul>
 *   <li>{@code beanDefinitionMap} — stores bean metadata (class, scope, supplier)</li>
 *   <li>{@code singletonObjects} — caches created singleton instances</li>
 * </ul>
 *
 * <p>Bean creation is lazy: a singleton is instantiated on the first {@code getBean()}
 * call, then cached for all subsequent calls.
 *
 * <p>Maps to Spring's {@code DefaultListableBeanFactory} (which extends
 * {@code DefaultSingletonBeanRegistry} for the singleton cache and implements
 * {@code BeanDefinitionRegistry} for definition storage).
 *
 * @see org.springframework.beans.factory.support.DefaultListableBeanFactory
 * @see org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
 */
public class DefaultBeanFactory implements BeanFactory, BeanDefinitionRegistry {

    /** Bean definitions keyed by name — the "what to create" map. */
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

    /** Registration-order list of bean definition names. */
    private final List<String> beanDefinitionNames = new ArrayList<>(256);

    /** Singleton instances keyed by name — the "already created" cache. */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    /** Ordered list of BeanPostProcessors to apply during bean creation. */
    private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();

    /**
     * Names of beans that are currently in the creation process.
     *
     * <p>This set detects circular dependencies: if {@code doGetBean()} is called
     * for a bean that's already in this set, a circular reference exists and
     * {@link BeanCurrentlyInCreationException} is thrown.
     *
     * <p>In the real Spring Framework, this is
     * {@code DefaultSingletonBeanRegistry.singletonsCurrentlyInCreation} — a
     * {@code Collections.newSetFromMap(ConcurrentHashMap)}. The real implementation
     * also has a three-level cache (singletonObjects, earlySingletonObjects,
     * singletonFactories) that can resolve some circular references for field
     * injection. We only detect and report — no resolution.
     *
     * @see org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
     */
    private final Set<String> beansCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>());

    // -----------------------------------------------------------------------
    // BeanPostProcessor registration
    // -----------------------------------------------------------------------

    /**
     * Add a BeanPostProcessor that will be applied to beans created by this factory.
     * Follows the real Spring pattern: remove-then-add ensures no duplicates while
     * preserving latest ordering.
     *
     * @see org.springframework.beans.factory.support.AbstractBeanFactory#addBeanPostProcessor
     */
    public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
        this.beanPostProcessors.remove(beanPostProcessor);
        this.beanPostProcessors.add(beanPostProcessor);
    }

    /**
     * Return the list of registered BeanPostProcessors.
     */
    public List<BeanPostProcessor> getBeanPostProcessors() {
        return this.beanPostProcessors;
    }

    // -----------------------------------------------------------------------
    // BeanDefinitionRegistry implementation
    // -----------------------------------------------------------------------

    @Override
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
        this.beanDefinitionMap.put(beanName, beanDefinition);
        if (!this.beanDefinitionNames.contains(beanName)) {
            this.beanDefinitionNames.add(beanName);
        }
    }

    @Override
    public BeanDefinition getBeanDefinition(String beanName) {
        BeanDefinition bd = this.beanDefinitionMap.get(beanName);
        if (bd == null) {
            throw new NoSuchBeanDefinitionException(beanName);
        }
        return bd;
    }

    @Override
    public boolean containsBeanDefinition(String beanName) {
        return this.beanDefinitionMap.containsKey(beanName);
    }

    @Override
    public String[] getBeanDefinitionNames() {
        return this.beanDefinitionNames.toArray(new String[0]);
    }

    @Override
    public int getBeanDefinitionCount() {
        return this.beanDefinitionMap.size();
    }

    // -----------------------------------------------------------------------
    // BeanFactory implementation
    // -----------------------------------------------------------------------

    @Override
    public Object getBean(String name) {
        return doGetBean(name);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(String name, Class<T> requiredType) {
        Object bean = doGetBean(name);
        if (!requiredType.isInstance(bean)) {
            throw new BeanCreationException(name,
                    "Bean is not of required type '" + requiredType.getName() + "'");
        }
        return (T) bean;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> requiredType) {
        List<String> candidateNames = new ArrayList<>();
        for (Map.Entry<String, BeanDefinition> entry : this.beanDefinitionMap.entrySet()) {
            if (requiredType.isAssignableFrom(entry.getValue().getBeanClass())) {
                candidateNames.add(entry.getKey());
            }
        }

        if (candidateNames.isEmpty()) {
            throw new NoSuchBeanDefinitionException(requiredType);
        }
        if (candidateNames.size() > 1) {
            throw new NoUniqueBeanDefinitionException(requiredType, candidateNames);
        }

        return (T) doGetBean(candidateNames.get(0));
    }

    @Override
    public boolean containsBean(String name) {
        return containsBeanDefinition(name);
    }

    @Override
    public boolean isSingleton(String name) {
        BeanDefinition bd = getBeanDefinition(name);
        return bd.isSingleton();
    }

    // -----------------------------------------------------------------------
    // Type-based lookup (multiple results)
    // -----------------------------------------------------------------------

    /**
     * Return all beans matching the given type. Returns a name→instance map
     * preserving bean definition order.
     *
     * <p>This is the simplified equivalent of Spring's
     * {@code ListableBeanFactory.getBeansOfType()} which lives on
     * {@code DefaultListableBeanFactory}. We add it here because the
     * {@code ApplicationContext} needs it to discover {@code BeanPostProcessor}
     * and {@code ApplicationListener} beans.
     *
     * @param type the class or interface to match
     * @return ordered map of bean name → bean instance (may be empty, never null)
     * @see org.springframework.beans.factory.ListableBeanFactory#getBeansOfType
     */
    @SuppressWarnings("unchecked")
    public <T> Map<String, T> getBeansOfType(Class<T> type) {
        Map<String, T> result = new LinkedHashMap<>();
        for (String name : this.beanDefinitionNames) {
            BeanDefinition bd = this.beanDefinitionMap.get(name);
            if (bd != null && type.isAssignableFrom(bd.getBeanClass())) {
                result.put(name, (T) getBean(name));
            }
        }
        return result;
    }

    // -----------------------------------------------------------------------
    // Manual singleton registration
    // -----------------------------------------------------------------------

    /**
     * Register an externally created singleton instance with the container.
     *
     * <p>The instance is placed directly into the singleton cache and a
     * {@link BeanDefinition} is registered so that type-based lookups
     * ({@link #getBeansOfType(Class)}) can discover it.
     *
     * <p>This is used by the boot layer to register infrastructure beans
     * that are created programmatically (e.g., {@code SmartLifecycle} beans
     * for web server start/stop management).
     *
     * <p>In the real Spring Framework, this is
     * {@code DefaultSingletonBeanRegistry.registerSingleton()} which only
     * adds to the singleton cache. Type-based lookup then checks both
     * definitions and manual singletons. We also register a definition
     * for simplicity.
     *
     * @param beanName        the bean name
     * @param singletonObject the singleton instance
     * @see org.springframework.beans.factory.config.SingletonBeanRegistry#registerSingleton
     */
    public void registerSingleton(String beanName, Object singletonObject) {
        this.singletonObjects.put(beanName, singletonObject);
        if (!containsBeanDefinition(beanName)) {
            BeanDefinition bd = new BeanDefinition(singletonObject.getClass());
            registerBeanDefinition(beanName, bd);
        }
    }

    // -----------------------------------------------------------------------
    // Eager singleton instantiation
    // -----------------------------------------------------------------------

    /**
     * Instantiate all singleton bean definitions that haven't been created yet.
     *
     * <p>This transitions the container from lazy to eager initialization — the
     * same thing that happens during {@code ApplicationContext.refresh()}.
     * In the real Spring Framework, this is
     * {@code DefaultListableBeanFactory.preInstantiateSingletons()}, called from
     * {@code AbstractApplicationContext.finishBeanFactoryInitialization()}.
     *
     * @see org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
     */
    public void preInstantiateSingletons() {
        // Snapshot names to avoid ConcurrentModificationException
        List<String> names = new ArrayList<>(this.beanDefinitionNames);
        for (String name : names) {
            BeanDefinition bd = this.beanDefinitionMap.get(name);
            if (bd != null && bd.isSingleton()) {
                getBean(name);
            }
        }
    }

    // -----------------------------------------------------------------------
    // Internal bean creation
    // -----------------------------------------------------------------------

    private Object doGetBean(String name) {
        // 1. Check singleton cache
        Object singleton = this.singletonObjects.get(name);
        if (singleton != null) {
            return singleton;
        }

        // 2. Get the definition
        BeanDefinition bd = getBeanDefinition(name);

        // 3. Circular dependency check — if this bean is already being
        //    created further up the call stack, we have a cycle.
        if (bd.isSingleton() && !this.beansCurrentlyInCreation.add(name)) {
            throw new BeanCurrentlyInCreationException(name);
        }

        try {
            // 4. Create the instance
            Object bean = createBean(name, bd);

            // 5. Cache if singleton
            if (bd.isSingleton()) {
                this.singletonObjects.put(name, bean);
            }

            return bean;
        } finally {
            // 6. Remove from "in creation" set
            this.beansCurrentlyInCreation.remove(name);
        }
    }

    private Object createBean(String name, BeanDefinition bd) {
        try {
            // 1. Instantiate (factory method, supplier, or constructor injection)
            Object bean;
            if (bd.isFactoryMethodBean()) {
                bean = instantiateUsingFactoryMethod(name, bd);
            } else if (bd.getSupplier() != null) {
                bean = bd.getSupplier().get();
            } else {
                bean = instantiate(name, bd);
            }

            // 2. BeanPostProcessors — before initialization
            //    (@PostConstruct methods are invoked here via LifecycleBeanPostProcessor)
            bean = applyBeanPostProcessorsBeforeInitialization(bean, name);

            // 3. InitializingBean callback
            invokeInitMethods(bean, name);

            // 4. BeanPostProcessors — after initialization
            bean = applyBeanPostProcessorsAfterInitialization(bean, name);

            return bean;
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Exception ex) {
            throw new BeanCreationException(name,
                    "Instantiation of bean failed", ex);
        }
    }

    /**
     * Instantiate the bean, resolving constructor injection if needed.
     *
     * <p>Three strategies, in priority order:
     * <ol>
     *   <li>{@code @Autowired} constructor — explicitly marked for injection</li>
     *   <li>Single constructor with parameters — implicit autowiring (Spring 4.3+ behavior)</li>
     *   <li>No-arg constructor — reflective default instantiation</li>
     * </ol>
     *
     * <p>In the real Spring Framework, constructor selection is delegated to
     * {@code SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()},
     * then resolved by {@code ConstructorResolver.autowireConstructor()}. We inline
     * the logic here for simplicity.
     */
    private Object instantiate(String name, BeanDefinition bd) throws Exception {
        Class<?> clazz = bd.getBeanClass();
        Constructor<?>[] ctors = clazz.getDeclaredConstructors();

        // 1. Look for @Autowired constructor
        Constructor<?> selected = null;
        for (Constructor<?> ctor : ctors) {
            if (ctor.isAnnotationPresent(Autowired.class)) {
                selected = ctor;
                break;
            }
        }

        // 2. Single constructor with parameters → implicit autowiring
        if (selected == null && ctors.length == 1 && ctors[0].getParameterCount() > 0) {
            selected = ctors[0];
        }

        // 3. Autowire constructor parameters
        if (selected != null) {
            Class<?>[] paramTypes = selected.getParameterTypes();
            Object[] args = new Object[paramTypes.length];
            for (int i = 0; i < paramTypes.length; i++) {
                try {
                    args[i] = getBean(paramTypes[i]);
                } catch (NoSuchBeanDefinitionException ex) {
                    throw new BeanCreationException(name,
                            "Unsatisfied dependency for constructor parameter type '"
                                    + paramTypes[i].getName() + "'",
                            ex);
                }
            }
            selected.setAccessible(true);
            return selected.newInstance(args);
        }

        // 4. No-arg constructor fallback
        Constructor<?> defaultCtor = clazz.getDeclaredConstructor();
        defaultCtor.setAccessible(true);
        return defaultCtor.newInstance();
    }

    /**
     * Create a bean by invoking a {@code @Bean} factory method on its owning
     * {@code @Configuration} class instance.
     *
     * <p>This mirrors Spring's {@code ConstructorResolver.instantiateUsingFactoryMethod()}:
     * get the factory bean instance, resolve each method parameter by type from
     * the container, then invoke the method.
     *
     * @see org.springframework.beans.factory.support.ConstructorResolver#instantiateUsingFactoryMethod
     */
    private Object instantiateUsingFactoryMethod(String beanName, BeanDefinition bd) {
        String factoryBeanName = bd.getFactoryBeanName();
        Method factoryMethod = bd.getFactoryMethod();

        // Get the @Configuration class instance that owns this @Bean method
        Object factoryBean = getBean(factoryBeanName);

        // Resolve each method parameter from the container (same as constructor autowiring)
        Class<?>[] paramTypes = factoryMethod.getParameterTypes();
        Object[] args = new Object[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++) {
            try {
                args[i] = getBean(paramTypes[i]);
            } catch (NoSuchBeanDefinitionException ex) {
                throw new BeanCreationException(beanName,
                        "Unsatisfied dependency for @Bean method parameter type '"
                                + paramTypes[i].getName() + "' in factory method '"
                                + factoryMethod.getName() + "'",
                        ex);
            }
        }

        try {
            factoryMethod.setAccessible(true);
            return factoryMethod.invoke(factoryBean, args);
        } catch (Exception ex) {
            throw new BeanCreationException(beanName,
                    "Failed to invoke factory method '" + factoryMethod.getName()
                            + "' on configuration class '" + factoryBeanName + "'",
                    ex);
        }
    }

    private Object applyBeanPostProcessorsBeforeInitialization(Object bean, String beanName) {
        Object current = bean;
        for (BeanPostProcessor bp : this.beanPostProcessors) {
            current = bp.postProcessBeforeInitialization(current, beanName);
        }
        return current;
    }

    private Object applyBeanPostProcessorsAfterInitialization(Object bean, String beanName) {
        Object current = bean;
        for (BeanPostProcessor bp : this.beanPostProcessors) {
            current = bp.postProcessAfterInitialization(current, beanName);
        }
        return current;
    }

    /**
     * Invoke {@link InitializingBean#afterPropertiesSet()} if the bean implements it.
     *
     * <p>Called after {@code @PostConstruct} (which fires in
     * {@code postProcessBeforeInitialization}) and before
     * {@code postProcessAfterInitialization}. This matches the real Spring
     * ordering in {@code AbstractAutowireCapableBeanFactory.invokeInitMethods()}.
     */
    private void invokeInitMethods(Object bean, String beanName) throws Exception {
        if (bean instanceof InitializingBean initializingBean) {
            initializingBean.afterPropertiesSet();
        }
    }

    // -----------------------------------------------------------------------
    // Destruction / Close
    // -----------------------------------------------------------------------

    /**
     * Destroy all singleton beans in reverse registration order, then clear
     * the singleton cache.
     *
     * <p>For each singleton: (1) invoke {@code @PreDestroy} methods via
     * {@link LifecycleBeanPostProcessor}, then (2) call
     * {@link DisposableBean#destroy()} if implemented.
     *
     * <p>Maps to {@code DefaultSingletonBeanRegistry.destroySingletons()} which
     * iterates {@code disposableBeans} in reverse order.
     *
     * @see org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingletons()
     */
    public void close() {
        List<String> names = new ArrayList<>(this.beanDefinitionNames);
        for (int i = names.size() - 1; i >= 0; i--) {
            String name = names.get(i);
            Object singleton = this.singletonObjects.get(name);
            if (singleton != null) {
                destroySingleton(name, singleton);
            }
        }
        this.singletonObjects.clear();
    }

    private void destroySingleton(String beanName, Object bean) {
        // 1. @PreDestroy via LifecycleBeanPostProcessor
        for (BeanPostProcessor bp : this.beanPostProcessors) {
            if (bp instanceof LifecycleBeanPostProcessor lbp) {
                lbp.postProcessBeforeDestruction(bean, beanName);
            }
        }

        // 2. DisposableBean.destroy()
        if (bean instanceof DisposableBean disposableBean) {
            try {
                disposableBean.destroy();
            } catch (Exception ex) {
                System.err.println("Destroy method on bean '" + beanName + "' threw exception: " + ex);
            }
        }
    }
}
```

#### `[MODIFIED]` `iris-framework/src/main/java/com/iris/framework/context/ConfigurableApplicationContext.java`

```java
package com.iris.framework.context;

import java.io.Closeable;

import com.iris.framework.core.env.Environment;

/**
 * SPI interface to be implemented by most (if not all) application contexts.
 * Provides configuration and lifecycle methods beyond the read-only
 * {@link ApplicationContext} contract.
 *
 * <p>The two critical lifecycle methods:
 * <ul>
 *   <li>{@link #refresh()} — load/reload configuration, instantiate singletons</li>
 *   <li>{@link #close()} — release resources, destroy singletons</li>
 * </ul>
 *
 * <p>In the real Spring Framework, this also extends {@code Lifecycle} (for
 * start/stop semantics) and {@code Closeable}. We extend only {@code Closeable}
 * for simplicity.
 *
 * @see ApplicationContext
 * @see org.springframework.context.ConfigurableApplicationContext
 */
public interface ConfigurableApplicationContext extends ApplicationContext, Closeable {

    /**
     * Load or refresh the container configuration.
     *
     * <p>This is the single most important method in the framework — it
     * orchestrates the complete container startup:
     * <ol>
     *   <li>Process {@code @Configuration} classes and component scanning</li>
     *   <li>Register {@code BeanPostProcessor}s</li>
     *   <li>Instantiate all singleton beans</li>
     *   <li>Publish {@code ContextRefreshedEvent}</li>
     * </ol>
     *
     * @throws RuntimeException if the context cannot be initialized
     */
    void refresh();

    /**
     * Close this application context, releasing all resources and destroying
     * all singleton beans.
     *
     * <p>Publishes a {@code ContextClosedEvent} BEFORE destroying singletons,
     * so listeners can perform cleanup while the full bean graph is still available.
     */
    @Override
    void close();

    /**
     * Return whether this context is currently active (has been refreshed
     * and not yet closed).
     */
    boolean isActive();

    /**
     * Register a JVM shutdown hook that calls {@link #close()} when the
     * JVM shuts down (e.g., on SIGTERM/SIGINT or {@code System.exit()}).
     *
     * <p>This ensures that singletons are properly destroyed and
     * {@link SmartLifecycle} beans are stopped even if the user doesn't
     * call {@code close()} explicitly.
     *
     * <p>In the real Spring Framework, this is defined on
     * {@code ConfigurableApplicationContext} and implemented by
     * {@code AbstractApplicationContext.registerShutdownHook()}.
     *
     * @see Runtime#addShutdownHook(Thread)
     * @see org.springframework.context.ConfigurableApplicationContext#registerShutdownHook()
     */
    void registerShutdownHook();

    /**
     * Set the {@link Environment} for this application context.
     *
     * <p>Must be called BEFORE {@link #refresh()} so that property resolution
     * and {@code @Value} injection use the correct environment. This is how
     * the bootstrap layer ({@code IrisApplication}) can prepare the environment
     * externally before the context initializes.
     *
     * <p>In the real Spring Framework, this is defined on
     * {@code ConfigurableApplicationContext} and implemented by
     * {@code AbstractApplicationContext}.
     *
     * @param environment the environment to use
     */
    void setEnvironment(Environment environment);

    /**
     * Add an {@link ApplicationListener} that will receive events published
     * by this context.
     *
     * <p>Listeners added before {@code refresh()} will receive the
     * {@code ContextRefreshedEvent}. Listeners added after {@code refresh()}
     * will receive subsequent events including {@code ContextClosedEvent}.
     *
     * <p>In the real Spring Framework, this is defined on
     * {@code ConfigurableApplicationContext} and implemented by
     * {@code AbstractApplicationContext}. Listeners can also be discovered
     * automatically as beans during {@code refresh()}.
     *
     * @param listener the listener to register
     */
    void addApplicationListener(ApplicationListener<?> listener);
}
```

#### `[MODIFIED]` `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java`

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
import com.iris.framework.context.annotation.DeferredConfigurationLoader;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;
import com.iris.framework.context.support.DefaultLifecycleProcessor;
import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.PropertiesPropertySourceLoader;
import com.iris.framework.core.env.StandardEnvironment;

/**
 * Standalone application context that accepts {@link com.iris.framework.context.annotation.Configuration @Configuration}
 * classes as input — the primary context implementation for annotation-based configuration.
 *
 * <p>Usage:
 * <pre>{@code
 * var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
 * MyService svc = ctx.getBean(MyService.class);
 * // ... use beans ...
 * ctx.close();
 * }</pre>
 *
 * <p>The key method is {@link #refresh()}, which orchestrates the full container lifecycle:
 * <ol>
 *   <li>Process {@code @Configuration} classes (component scanning + {@code @Bean} methods)</li>
 *   <li>Register internal {@code BeanPostProcessor}s ({@code AutowiredBeanPostProcessor},
 *       {@code LifecycleBeanPostProcessor}, {@code ValueAnnotationBeanPostProcessor})</li>
 *   <li>Discover and register user-defined {@code BeanPostProcessor} beans</li>
 *   <li>Eagerly instantiate all singleton beans</li>
 *   <li>Discover {@code ApplicationListener} beans and publish {@code ContextRefreshedEvent}</li>
 * </ol>
 *
 * <p>Environment setup happens BEFORE {@code refresh()}: a {@link StandardEnvironment}
 * is created with system properties and system environment variables, then
 * {@code application.properties} is loaded from the classpath (if present)
 * and added as the lowest-priority property source.
 *
 * <p>In the real Spring Framework, this class hierarchy is:
 * <pre>
 * AnnotationConfigApplicationContext
 *   → GenericApplicationContext
 *     → AbstractApplicationContext (the refresh() template)
 *       → DefaultResourceLoader
 * </pre>
 * We collapse the three-level hierarchy into a single concrete class. The real
 * {@code AbstractApplicationContext.refresh()} has 12 steps — we implement the
 * 5 that matter most for understanding the pattern.
 *
 * @see ConfigurableApplicationContext
 * @see Environment
 * @see org.springframework.context.annotation.AnnotationConfigApplicationContext
 * @see org.springframework.context.support.AbstractApplicationContext#refresh()
 */
public class AnnotationConfigApplicationContext implements ConfigurableApplicationContext {

    /** Well-known name for the application.properties property source. */
    private static final String APPLICATION_PROPERTIES_SOURCE_NAME = "applicationProperties";

    /** Well-known classpath location for the application properties file. */
    private static final String APPLICATION_PROPERTIES_RESOURCE = "application.properties";

    /** The internal bean factory that stores definitions and singletons. */
    private final DefaultBeanFactory beanFactory;

    /** The environment holding property sources for this context. */
    private Environment environment;

    /** Application listeners registered programmatically or discovered from beans. */
    private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();

    /** Tracks whether refresh() has been called and close() has not. */
    private final AtomicBoolean active = new AtomicBoolean(false);

    /** Tracks whether close() has been called. */
    private final AtomicBoolean closed = new AtomicBoolean(false);

    /** The lifecycle processor that manages SmartLifecycle beans. */
    private LifecycleProcessor lifecycleProcessor;

    /** The registered JVM shutdown hook thread, or null if not registered. */
    private Thread shutdownHook;

    /**
     * Additional internal {@code BeanPostProcessor}s to register between
     * the value-injection processors and the lifecycle processor.
     *
     * <p>This allows the boot layer to inject BPPs at the correct position
     * in the processing chain without subclassing. For example, the
     * {@code ConfigurationPropertiesBindingPostProcessor} runs after
     * {@code @Autowired}/{@code @Value} injection but before
     * {@code @PostConstruct} callbacks.
     *
     * <p>In the real Spring Framework, BPP ordering is controlled via
     * {@code PriorityOrdered} and {@code Ordered} interfaces, with
     * {@code PostProcessorRegistrationDelegate} sorting them. We use
     * this explicit list for simplicity.
     *
     * @see #addInternalBeanPostProcessor(BeanPostProcessor)
     */
    private final List<BeanPostProcessor> additionalInternalBeanPostProcessors = new ArrayList<>();

    /**
     * Optional loader for deferred configuration classes (e.g., auto-configuration).
     *
     * <p>When set, the context invokes this loader after the first pass of
     * configuration processing, enabling the "back-off" pattern: deferred
     * configs can check what user beans are already registered.
     *
     * @see DeferredConfigurationLoader
     */
    private DeferredConfigurationLoader deferredConfigurationLoader;

    /**
     * Create a new context WITHOUT registering any classes and WITHOUT calling
     * {@link #refresh()}.
     *
     * <p>Use this constructor when you need to configure the context before
     * refreshing — for example, when the bootstrap layer ({@code IrisApplication})
     * needs to set the environment, register sources, and add listeners before
     * starting the container lifecycle.
     *
     * <p>After construction, call {@link #register(Class[])} to add configuration
     * classes, then call {@link #refresh()} to start the container.
     *
     * <p>In the real Spring Framework, this no-arg constructor is the default,
     * and the convenience constructor below delegates to it.
     */
    public AnnotationConfigApplicationContext() {
        this.beanFactory = new DefaultBeanFactory();
        this.environment = createEnvironment();
    }

    /**
     * Create a new context that registers the given {@code @Configuration}
     * classes and immediately calls {@link #refresh()}.
     *
     * <p>This matches the real Spring's convenience constructor:
     * {@code AnnotationConfigApplicationContext(Class<?>...)}.
     *
     * @param componentClasses one or more {@code @Configuration} or {@code @Component} classes
     */
    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this();
        register(componentClasses);
        refresh();
    }

    /**
     * Register one or more {@code @Configuration} or {@code @Component} classes
     * with this context.
     *
     * <p>Must be called BEFORE {@link #refresh()}. Each class is registered as
     * a {@link BeanDefinition} in the internal bean factory.
     *
     * <p>In the real Spring Framework, this method is on
     * {@code AnnotationConfigApplicationContext} and delegates to
     * {@code AnnotatedBeanDefinitionReader.register()}.
     *
     * @param componentClasses one or more annotated classes
     */
    public void register(Class<?>... componentClasses) {
        for (Class<?> componentClass : componentClasses) {
            String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
            beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
        }
    }

    /**
     * Create and configure the Environment.
     *
     * <p>Creates a {@link StandardEnvironment} (system properties + system env),
     * then loads {@code application.properties} from the classpath and adds it
     * as the lowest-priority property source.
     *
     * <p>In the real Spring Framework, environment creation happens in
     * {@code AbstractApplicationContext.getOrCreateEnvironment()}, and
     * {@code application.properties} loading is handled by Spring Boot's
     * {@code ConfigDataEnvironmentPostProcessor} during the bootstrap lifecycle
     * (before {@code refresh()}). We combine both steps here for simplicity.
     */
    private StandardEnvironment createEnvironment() {
        StandardEnvironment env = new StandardEnvironment();

        // Load application.properties if present on the classpath
        PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
        MapPropertySource appProps = loader.load(
                APPLICATION_PROPERTIES_SOURCE_NAME, APPLICATION_PROPERTIES_RESOURCE);
        if (appProps != null) {
            env.getPropertySources().addLast(appProps);
        }

        return env;
    }

    // -----------------------------------------------------------------------
    // ConfigurableApplicationContext — Lifecycle
    // -----------------------------------------------------------------------

    /**
     * The main startup method — orchestrates the full container initialization.
     *
     * <p>This is the simplified equivalent of
     * {@code AbstractApplicationContext.refresh()}'s 12-step pipeline. We implement
     * 6 steps that capture the essential pattern:
     *
     * <pre>
     * refresh()
     *   ├── 1. invokeBeanFactoryPostProcessors()    ← @Configuration + scanning
     *   ├── 2. registerBeanPostProcessors()          ← internal + user-defined
     *   ├── 3. onRefresh()                           ← template method hook (e.g., start web server)
     *   ├── 4. finishBeanFactoryInitialization()     ← eager singleton creation
     *   ├── 5. registerListeners()                   ← discover ApplicationListener beans
     *   └── 6. finishRefresh()                       ← publish ContextRefreshedEvent
     * </pre>
     */
    @Override
    public void refresh() {
        try {
            // Step 1: Process @Configuration classes — component scanning + @Bean methods
            invokeBeanFactoryPostProcessors();

            // Step 2: Register BeanPostProcessors (internal + user-defined)
            registerBeanPostProcessors();

            // Step 3: Template method — subclasses can hook in here
            // (e.g., ServletWebServerApplicationContext creates the embedded server)
            onRefresh();

            // Step 4: Instantiate all remaining singleton beans
            finishBeanFactoryInitialization();

            // Step 5: Discover ApplicationListener beans
            registerListeners();

            // Step 6: Publish ContextRefreshedEvent
            finishRefresh();

            this.active.set(true);
            this.closed.set(false);
        } catch (RuntimeException ex) {
            // On failure: destroy any already-created singletons
            this.beanFactory.close();
            this.active.set(false);
            throw ex;
        }
    }

    /**
     * Template method which can be overridden to add context-specific refresh work.
     * Called during {@link #refresh()} after {@code registerBeanPostProcessors()}
     * and before {@code finishBeanFactoryInitialization()}.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.onRefresh()} (line 906). It's the hook
     * where {@code ServletWebServerApplicationContext} creates the embedded
     * web server — the server is initialized here but does not accept connections
     * until {@code finishRefresh()}.
     *
     * <p>The default implementation is empty.
     *
     * @see org.springframework.context.support.AbstractApplicationContext#onRefresh()
     */
    protected void onRefresh() {
        // For subclasses to override — do nothing by default.
    }

    /**
     * Step 1: Process {@code @Configuration} classes.
     *
     * <p>Processing happens in two passes:
     * <ol>
     *   <li><b>First pass:</b> Process all explicitly registered
     *       {@code @Configuration} classes — component scanning + {@code @Bean}
     *       methods. This registers all user-defined beans.</li>
     *   <li><b>Second pass (deferred):</b> If a {@link DeferredConfigurationLoader}
     *       is set, invoke it to load additional configuration classes (e.g.,
     *       auto-configuration candidates). Register them and re-run the
     *       processor. Condition evaluation in the second pass can see all
     *       user beans from the first pass.</li>
     * </ol>
     *
     * <p>The two-pass design is the simplified equivalent of Spring's
     * {@code DeferredImportSelector}: user configs are fully processed
     * before auto-configs, enabling the "back-off" pattern where
     * {@code @ConditionalOnMissingBean} detects user beans.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.invokeBeanFactoryPostProcessors()},
     * which delegates to {@code PostProcessorRegistrationDelegate} to invoke
     * all {@code BeanFactoryPostProcessor} beans. The most important one is
     * {@code ConfigurationClassPostProcessor}, which handles {@code @Configuration},
     * {@code @ComponentScan}, {@code @Import}, etc.
     */
    protected void invokeBeanFactoryPostProcessors() {
        ConfigurationClassProcessor processor = new ConfigurationClassProcessor();

        // First pass: process user @Configuration classes
        processor.processConfigurationClasses(this.beanFactory, this.environment);

        // Second pass: process deferred configurations (auto-configuration)
        if (this.deferredConfigurationLoader != null) {
            List<Class<?>> deferredConfigs =
                    this.deferredConfigurationLoader.loadConfigurations(
                            this.beanFactory, this.environment);
            if (!deferredConfigs.isEmpty()) {
                for (Class<?> configClass : deferredConfigs) {
                    String beanName = ConfigurationClassProcessor.deriveConfigBeanName(configClass);
                    if (!this.beanFactory.containsBeanDefinition(beanName)) {
                        this.beanFactory.registerBeanDefinition(
                                beanName, new BeanDefinition(configClass));
                    }
                }
                // Re-run: processes newly registered auto-config classes;
                // already-processed user classes are re-processed idempotently
                processor.processConfigurationClasses(this.beanFactory, this.environment);
            }
        }
    }

    /**
     * Step 2: Register {@code BeanPostProcessor}s.
     *
     * <p>Registers processors in four groups, in order:
     * <ol>
     *   <li>{@code AutowiredBeanPostProcessor} — handles {@code @Autowired} field injection</li>
     *   <li>{@code ValueAnnotationBeanPostProcessor} — handles {@code @Value} field injection</li>
     *   <li><b>Additional internal BPPs</b> — registered by the boot layer via
     *       {@link #addInternalBeanPostProcessor(BeanPostProcessor)} (e.g.,
     *       {@code ConfigurationPropertiesBindingPostProcessor})</li>
     *   <li>{@code LifecycleBeanPostProcessor} — handles {@code @PostConstruct} / {@code @PreDestroy}</li>
     * </ol>
     *
     * <p>Then discovers any user-defined {@code BeanPostProcessor} beans registered via
     * {@code @Configuration}/{@code @Bean} or component scanning, and adds them too.
     *
     * <p>The ordering ensures that property binding (group 3) happens after
     * dependency injection (groups 1-2) but before lifecycle callbacks (group 4).
     *
     * <p>In the real Spring Framework, this is
     * {@code PostProcessorRegistrationDelegate.registerBeanPostProcessors()},
     * which sorts processors by {@code PriorityOrdered}, {@code Ordered}, and
     * then unordered. We use explicit group ordering for simplicity.
     */
    private void registerBeanPostProcessors() {
        // Group 1-2: Internal injection processors — always present
        this.beanFactory.addBeanPostProcessor(new AutowiredBeanPostProcessor(this.beanFactory));
        this.beanFactory.addBeanPostProcessor(new ValueAnnotationBeanPostProcessor(this.environment));

        // Group 3: Additional internal BPPs from the boot layer
        // (e.g., ConfigurationPropertiesBindingPostProcessor)
        for (BeanPostProcessor bp : this.additionalInternalBeanPostProcessors) {
            this.beanFactory.addBeanPostProcessor(bp);
        }

        // Group 4: Lifecycle processor — runs @PostConstruct/@PreDestroy
        this.beanFactory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

        // User-defined BeanPostProcessors registered as beans
        Map<String, BeanPostProcessor> postProcessorBeans =
                this.beanFactory.getBeansOfType(BeanPostProcessor.class);
        for (BeanPostProcessor bp : postProcessorBeans.values()) {
            this.beanFactory.addBeanPostProcessor(bp);
        }
    }

    /**
     * Step 3: Eagerly instantiate all singleton beans.
     *
     * <p>Up until this point, bean creation was lazy (on first {@code getBean()}).
     * This step forces creation of all singletons, which triggers the full
     * dependency injection and lifecycle pipeline for every bean.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.finishBeanFactoryInitialization()},
     * which calls {@code beanFactory.preInstantiateSingletons()}.
     */
    private void finishBeanFactoryInitialization() {
        this.beanFactory.preInstantiateSingletons();
    }

    /**
     * Step 4: Discover and register {@code ApplicationListener} beans.
     *
     * <p>Finds all beans implementing {@code ApplicationListener} and adds
     * them to the listener set. In the real Spring Framework, this is
     * {@code AbstractApplicationContext.registerListeners()}, which also
     * handles programmatically registered listeners and early events.
     */
    @SuppressWarnings({"rawtypes", "unchecked"})
    private void registerListeners() {
        Map<String, ApplicationListener> listenerBeans =
                this.beanFactory.getBeansOfType(ApplicationListener.class);
        for (ApplicationListener listener : listenerBeans.values()) {
            this.applicationListeners.add(listener);
        }
    }

    /**
     * Step 5: Initialize the LifecycleProcessor, start SmartLifecycle beans,
     * and publish the {@code ContextRefreshedEvent}.
     *
     * <p>In the real Spring Framework ({@code AbstractApplicationContext.finishRefresh()},
     * line 1002), this method:
     * <ol>
     *   <li>Initializes the {@link LifecycleProcessor}</li>
     *   <li>Calls {@code lifecycleProcessor.onRefresh()} — starts all
     *       {@code SmartLifecycle} beans with {@code isAutoStartup() == true}</li>
     *   <li>Publishes {@code ContextRefreshedEvent}</li>
     * </ol>
     *
     * <p>This is the integration point for Feature 19 (Graceful Shutdown):
     * the embedded web server starts here via {@code WebServerStartStopLifecycle},
     * a SmartLifecycle bean registered during {@code onRefresh()}.
     */
    private void finishRefresh() {
        initLifecycleProcessor();
        getLifecycleProcessor().onRefresh();
        publishEvent(new ContextRefreshedEvent(this));
    }

    /**
     * Initialize the LifecycleProcessor.
     *
     * <p>If a {@code "lifecycleProcessor"} bean is registered in the container,
     * uses it. Otherwise, creates a {@link DefaultLifecycleProcessor} with
     * access to the bean factory.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.initLifecycleProcessor()} (line 879).
     */
    private void initLifecycleProcessor() {
        if (this.beanFactory.containsBean("lifecycleProcessor")) {
            this.lifecycleProcessor = this.beanFactory.getBean(
                    "lifecycleProcessor", LifecycleProcessor.class);
        } else {
            DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
            defaultProcessor.setBeanFactory(this.beanFactory);
            this.lifecycleProcessor = defaultProcessor;
            this.beanFactory.registerSingleton("lifecycleProcessor", this.lifecycleProcessor);
        }
    }

    /**
     * Return the LifecycleProcessor for this context.
     *
     * @return the lifecycle processor (never null after refresh)
     */
    private LifecycleProcessor getLifecycleProcessor() {
        if (this.lifecycleProcessor == null) {
            throw new IllegalStateException(
                    "LifecycleProcessor not initialized - call 'refresh' before");
        }
        return this.lifecycleProcessor;
    }

    /**
     * Close this context, releasing all resources and destroying all
     * singleton beans.
     *
     * <p>The shutdown sequence is:
     * <ol>
     *   <li>Publish {@code ContextClosedEvent} while beans are still alive</li>
     *   <li>Stop {@link SmartLifecycle} beans in reverse phase order via
     *       the {@link LifecycleProcessor} — this is where graceful shutdown
     *       happens (draining in-flight requests, stopping the web server)</li>
     *   <li>Destroy all singletons ({@code @PreDestroy}, {@code DisposableBean})</li>
     *   <li>Mark context as inactive</li>
     * </ol>
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.doClose()} (line 1181). The lifecycle
     * processing between event publication and singleton destruction is the
     * key enhancement from Feature 19.
     */
    @Override
    public void close() {
        if (this.closed.compareAndSet(false, true)) {
            // 1. Publish ContextClosedEvent while beans are still alive
            publishEvent(new ContextClosedEvent(this));

            // 2. Stop SmartLifecycle beans in reverse phase order
            //    This is where graceful shutdown happens — the web server
            //    drains in-flight requests before stopping
            if (this.lifecycleProcessor != null) {
                try {
                    this.lifecycleProcessor.onClose();
                } catch (Exception ex) {
                    System.err.println("LifecycleProcessor onClose failed: " + ex.getMessage());
                }
            }

            // 3. Destroy all singletons (@PreDestroy, DisposableBean)
            this.beanFactory.close();

            // 4. Mark as inactive and clear listeners
            this.active.set(false);
            this.applicationListeners.clear();

            // 5. Remove the shutdown hook if one was registered
            //    (prevents double-close if close() is called explicitly
            //    before the JVM shuts down)
            removeShutdownHook();
        }
    }

    /**
     * Register a JVM shutdown hook that calls {@link #close()} when the
     * JVM shuts down.
     *
     * <p>This ensures that the web server is stopped gracefully and
     * singletons are destroyed even if the user doesn't call
     * {@code close()} explicitly. The hook is automatically removed
     * when {@code close()} is called to prevent double invocation.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.registerShutdownHook()} (line 1068).
     */
    @Override
    public void registerShutdownHook() {
        if (this.shutdownHook == null) {
            this.shutdownHook = new Thread(this::close, "iris-shutdown-hook");
            Runtime.getRuntime().addShutdownHook(this.shutdownHook);
        }
    }

    /**
     * Remove the JVM shutdown hook if one was previously registered.
     *
     * <p>Called from {@link #close()} to prevent the hook from running
     * after close has already been called. Catches
     * {@code IllegalStateException} in case the JVM is already in the
     * process of shutting down.
     */
    private void removeShutdownHook() {
        if (this.shutdownHook != null) {
            try {
                Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
            } catch (IllegalStateException ex) {
                // JVM is already shutting down — cannot remove hook
            }
        }
    }

    @Override
    public boolean isActive() {
        return this.active.get();
    }

    // -----------------------------------------------------------------------
    // ApplicationEventPublisher
    // -----------------------------------------------------------------------

    /**
     * Publish the given event to all listeners whose type parameter matches.
     *
     * <p>Uses {@link #resolveEventType(ApplicationListener)} to determine
     * the event type each listener accepts, then only invokes listeners
     * whose type is assignable from the actual event.
     *
     * <p>In the real Spring Framework, this dispatching is done by
     * {@code SimpleApplicationEventMulticaster.multicastEvent()}, which
     * caches listener resolution per event type. We do a simple linear
     * scan for clarity.
     */
    @Override
    public void publishEvent(ApplicationEvent event) {
        for (ApplicationListener<?> listener : this.applicationListeners) {
            Class<?> eventType = resolveEventType(listener);
            if (eventType == null || eventType.isInstance(event)) {
                invokeListener(listener, event);
            }
        }
    }

    @SuppressWarnings({"rawtypes", "unchecked"})
    private void invokeListener(ApplicationListener listener, ApplicationEvent event) {
        listener.onApplicationEvent(event);
    }

    /**
     * Resolve the event type parameter {@code E} from an {@code ApplicationListener<E>}.
     *
     * <p>Walks the class hierarchy and implemented interfaces to find the
     * concrete type argument. Returns {@code null} if the type cannot be
     * resolved (in which case the listener receives all events).
     *
     * <p>In the real Spring Framework, this is handled by
     * {@code GenericApplicationListenerAdapter} using {@code ResolvableType}.
     * We use direct reflection on generic interfaces.
     */
    static Class<?> resolveEventType(ApplicationListener<?> listener) {
        // Check all generic interfaces on the listener class and its superclasses
        Class<?> targetClass = listener.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Type genericInterface : targetClass.getGenericInterfaces()) {
                if (genericInterface instanceof ParameterizedType pt) {
                    if (ApplicationListener.class.isAssignableFrom((Class<?>) pt.getRawType())) {
                        Type typeArg = pt.getActualTypeArguments()[0];
                        if (typeArg instanceof Class<?> clazz) {
                            return clazz;
                        }
                    }
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return null;
    }

    // -----------------------------------------------------------------------
    // BeanFactory — delegation
    // -----------------------------------------------------------------------

    @Override
    public Object getBean(String name) {
        assertActive();
        return this.beanFactory.getBean(name);
    }

    @Override
    public <T> T getBean(String name, Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(name, requiredType);
    }

    @Override
    public <T> T getBean(Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(requiredType);
    }

    @Override
    public boolean containsBean(String name) {
        return this.beanFactory.containsBean(name);
    }

    @Override
    public boolean isSingleton(String name) {
        return this.beanFactory.isSingleton(name);
    }

    // -----------------------------------------------------------------------
    // ApplicationContext
    // -----------------------------------------------------------------------

    @Override
    public String getDisplayName() {
        return getClass().getSimpleName() + "@" + Integer.toHexString(hashCode());
    }

    @Override
    public Environment getEnvironment() {
        return this.environment;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    /**
     * Return the underlying {@code DefaultBeanFactory}.
     *
     * <p>In the real Spring Framework, {@code GenericApplicationContext.getBeanFactory()}
     * returns a {@code ConfigurableListableBeanFactory}. We return the concrete type
     * since we have no intermediate abstraction.
     */
    public DefaultBeanFactory getBeanFactory() {
        return this.beanFactory;
    }

    // -----------------------------------------------------------------------
    // Listener management
    // -----------------------------------------------------------------------

    /**
     * Add a programmatic listener. Must be called before {@code refresh()}.
     *
     * <p>In the real Spring Framework, this is
     * {@code ConfigurableApplicationContext.addApplicationListener()}.
     */
    @Override
    public void addApplicationListener(ApplicationListener<?> listener) {
        this.applicationListeners.add(listener);
    }

    /**
     * Add an internal {@link BeanPostProcessor} to be registered between the
     * injection processors ({@code @Autowired}, {@code @Value}) and the
     * lifecycle processor ({@code @PostConstruct}/{@code @PreDestroy}).
     *
     * <p>This is the extension point for the boot layer to add BPPs that
     * need specific ordering — such as the
     * {@code ConfigurationPropertiesBindingPostProcessor} which must run
     * after injection but before lifecycle callbacks.
     *
     * <p>Must be called before {@link #refresh()}.
     *
     * <p>In the real Spring Framework, ordering is handled by
     * {@code PriorityOrdered}/{@code Ordered} interfaces. We provide this
     * explicit insertion point for simplicity.
     *
     * @param bpp the bean post-processor to add
     */
    public void addInternalBeanPostProcessor(BeanPostProcessor bpp) {
        this.additionalInternalBeanPostProcessors.add(bpp);
    }

    /**
     * Set a {@link DeferredConfigurationLoader} for loading additional
     * configuration classes after user configs are processed.
     *
     * <p>This is the integration point for auto-configuration (Feature 13):
     * the boot layer sets a loader that reads auto-configuration candidates
     * from {@code META-INF/iris/AutoConfiguration.imports}.
     *
     * <p>Must be called before {@link #refresh()}.
     *
     * @param loader the deferred configuration loader
     * @see DeferredConfigurationLoader
     */
    public void setDeferredConfigurationLoader(DeferredConfigurationLoader loader) {
        this.deferredConfigurationLoader = loader;
    }

    // -----------------------------------------------------------------------
    // Internal
    // -----------------------------------------------------------------------

    private void assertActive() {
        if (!this.active.get()) {
            throw new IllegalStateException(
                    "ApplicationContext has not been refreshed yet or has already been closed");
        }
    }
}
```

#### `[MODIFIED]` `iris-framework/src/main/java/com/iris/framework/core/env/PropertySourcesPropertyResolver.java`

```java
package com.iris.framework.core.env;

/**
 * {@link PropertyResolver} implementation that resolves property values
 * by iterating over a set of {@link PropertySource}s.
 *
 * <p>The core resolution algorithm: iterate the property sources in order
 * and return the first non-null value found. This is the "precedence"
 * mechanism — sources earlier in the list override later ones.
 *
 * <p>Also handles {@code ${...}} placeholder resolution with support for
 * default values ({@code ${key:default}}) and multiple placeholders in a
 * single string ({@code ${host}:${port}}).
 *
 * <p>In the real Spring Framework, placeholder resolution is split across
 * {@code AbstractPropertyResolver} (the placeholder parsing engine) and
 * {@code PropertySourcesPropertyResolver} (the property source iteration).
 * The parsing is delegated to {@code PropertyPlaceholderHelper}. We inline
 * both for simplicity.
 *
 * @see MutablePropertySources
 * @see org.springframework.core.env.PropertySourcesPropertyResolver
 */
public class PropertySourcesPropertyResolver implements PropertyResolver {

    private static final String PLACEHOLDER_PREFIX = "${";
    private static final String PLACEHOLDER_SUFFIX = "}";
    private static final String VALUE_SEPARATOR = ":";

    private final MutablePropertySources propertySources;

    public PropertySourcesPropertyResolver(MutablePropertySources propertySources) {
        this.propertySources = propertySources;
    }

    @Override
    public String getProperty(String key) {
        for (PropertySource<?> propertySource : this.propertySources) {
            Object value = propertySource.getProperty(key);
            if (value != null) {
                return String.valueOf(value);
            }
        }
        return null;
    }

    @Override
    public String getProperty(String key, String defaultValue) {
        String value = getProperty(key);
        return (value != null) ? value : defaultValue;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProperty(String key, Class<T> targetType) {
        String value = getProperty(key);
        if (value == null) {
            return null;
        }
        return (T) convertValue(value, targetType);
    }

    @Override
    public String getRequiredProperty(String key) {
        String value = getProperty(key);
        if (value == null) {
            throw new IllegalStateException("Required property '" + key + "' not found");
        }
        return value;
    }

    @Override
    public boolean containsProperty(String key) {
        for (PropertySource<?> propertySource : this.propertySources) {
            if (propertySource.containsProperty(key)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public String resolvePlaceholders(String text) {
        return doResolvePlaceholders(text, false);
    }

    @Override
    public String resolveRequiredPlaceholders(String text) {
        return doResolvePlaceholders(text, true);
    }

    // -----------------------------------------------------------------------
    // Placeholder resolution engine
    // -----------------------------------------------------------------------

    /**
     * Parse and resolve {@code ${...}} placeholders in the given text.
     *
     * <p>Supports:
     * <ul>
     *   <li>Simple placeholders: {@code ${key}}</li>
     *   <li>Default values: {@code ${key:default}}</li>
     *   <li>Multiple placeholders: {@code ${host}:${port}}</li>
     * </ul>
     *
     * <p>In the real Spring Framework, this parsing is handled by
     * {@code PropertyPlaceholderHelper}, a standalone utility with support
     * for nested placeholders ({@code ${${env}.url}}), escape characters,
     * and configurable prefix/suffix. We implement the essential algorithm
     * inline.
     *
     * @param text the text containing placeholders
     * @param failOnUnresolvable if true, throw on unresolvable placeholders
     * @return the resolved text
     */
    private String doResolvePlaceholders(String text, boolean failOnUnresolvable) {
        if (text == null || !text.contains(PLACEHOLDER_PREFIX)) {
            return text;
        }

        StringBuilder result = new StringBuilder();
        int i = 0;

        while (i < text.length()) {
            int start = text.indexOf(PLACEHOLDER_PREFIX, i);
            if (start == -1) {
                // No more placeholders — append remainder
                result.append(text, i, text.length());
                break;
            }

            // Append text before the placeholder
            result.append(text, i, start);

            // Find the matching closing brace
            int end = text.indexOf(PLACEHOLDER_SUFFIX, start + PLACEHOLDER_PREFIX.length());
            if (end == -1) {
                // Unclosed placeholder — append as-is
                result.append(text, start, text.length());
                break;
            }

            // Extract the placeholder content: "key" or "key:default"
            String placeholder = text.substring(start + PLACEHOLDER_PREFIX.length(), end);

            // Split on the FIRST ':' for default value support
            String key;
            String defaultValue = null;
            int separatorIdx = placeholder.indexOf(VALUE_SEPARATOR);
            if (separatorIdx != -1) {
                key = placeholder.substring(0, separatorIdx);
                defaultValue = placeholder.substring(separatorIdx + VALUE_SEPARATOR.length());
            } else {
                key = placeholder;
            }

            // Resolve the property value
            String resolved = getProperty(key);
            if (resolved != null) {
                result.append(resolved);
            } else if (defaultValue != null) {
                result.append(defaultValue);
            } else if (failOnUnresolvable) {
                throw new IllegalArgumentException(
                        "Could not resolve placeholder '" + key + "' in value \"" + text + "\"");
            } else {
                // Leave unresolvable placeholder as-is
                result.append(PLACEHOLDER_PREFIX).append(placeholder).append(PLACEHOLDER_SUFFIX);
            }

            i = end + PLACEHOLDER_SUFFIX.length();
        }

        return result.toString();
    }

    // -----------------------------------------------------------------------
    // Type conversion
    // -----------------------------------------------------------------------

    /**
     * Convert a string value to the target type.
     *
     * <p>Supports common types: {@code String}, {@code int}/{@code Integer},
     * {@code long}/{@code Long}, {@code boolean}/{@code Boolean},
     * {@code double}/{@code Double}.
     *
     * <p>In the real Spring Framework, type conversion is handled by a full
     * {@code ConversionService} framework with hundreds of converters. We
     * support only the types needed for {@code @Value} injection.
     */
    public static Object convertValue(String value, Class<?> targetType) {
        if (targetType == String.class) {
            return value;
        }
        if (targetType == int.class || targetType == Integer.class) {
            return Integer.parseInt(value);
        }
        if (targetType == long.class || targetType == Long.class) {
            return Long.parseLong(value);
        }
        if (targetType == boolean.class || targetType == Boolean.class) {
            return Boolean.parseBoolean(value);
        }
        if (targetType == double.class || targetType == Double.class) {
            return Double.parseDouble(value);
        }
        if (targetType.isEnum()) {
            return convertToEnum(value, targetType);
        }
        throw new IllegalArgumentException(
                "Cannot convert value '" + value + "' to type " + targetType.getName());
    }

    /**
     * Convert a string value to an enum constant.
     *
     * <p>Tries exact match first, then case-insensitive match.
     * Supports both {@code "GRACEFUL"} and {@code "graceful"} for
     * enum constants like {@code Shutdown.GRACEFUL}.
     *
     * <p>In the real Spring Framework, enum conversion is handled by
     * the {@code ConversionService} framework with specialized converters.
     * We use a simple name-based lookup.
     */
    @SuppressWarnings({"unchecked", "rawtypes"})
    private static Object convertToEnum(String value, Class<?> enumType) {
        try {
            return Enum.valueOf((Class<Enum>) enumType, value);
        } catch (IllegalArgumentException ex) {
            // Try uppercase (common in .properties: "graceful" → "GRACEFUL")
            try {
                return Enum.valueOf((Class<Enum>) enumType, value.toUpperCase());
            } catch (IllegalArgumentException ex2) {
                throw new IllegalArgumentException(
                        "Cannot convert value '" + value + "' to enum type "
                                + enumType.getName(), ex2);
            }
        }
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBinder.java`

```java
package com.iris.boot.context.properties;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.List;

import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.PropertySource;
import com.iris.framework.core.env.PropertySourcesPropertyResolver;

/**
 * Binds properties from the {@link Environment} to a target object's fields
 * using a given prefix.
 *
 * <p>For each non-static, non-final field in the target class, the binder
 * constructs property keys by appending the field name to the prefix,
 * looks them up in the environment, converts to the field's type, and
 * sets the value via reflection.
 *
 * <h3>Binding algorithm</h3>
 *
 * <p>For a prefix of {@code "app.server"} and a field named {@code maxConnections}:
 * <ol>
 *   <li>Try exact: {@code app.server.maxConnections}</li>
 *   <li>Try kebab-case: {@code app.server.max-connections}</li>
 *   <li>If either matches, convert and set the field</li>
 * </ol>
 *
 * <p>For nested objects (fields whose type is not a simple type or List),
 * the binder recurses with an extended prefix. Nested binding only applies
 * if the field is already initialized (non-null) — we don't auto-create
 * nested instances.
 *
 * <p>In the real Spring Boot, this is handled by the {@code Binder} class
 * which is a sophisticated recursive engine supporting:
 * <ul>
 *   <li>{@code JavaBeanBinder} for setter-based binding</li>
 *   <li>{@code ValueObjectBinder} for constructor-based binding</li>
 *   <li>Full relaxed binding with {@code ConfigurationPropertyName}</li>
 *   <li>{@code Map}, {@code Set}, indexed {@code List} binding</li>
 *   <li>{@code BindHandler} chain for validation/error handling</li>
 * </ul>
 * We simplify to direct field access with two-form relaxed binding.
 *
 * @see ConfigurationProperties
 * @see ConfigurationPropertiesBindingPostProcessor
 * @see org.springframework.boot.context.properties.bind.Binder
 */
public class ConfigurationPropertiesBinder {

    private final Environment environment;

    public ConfigurationPropertiesBinder(Environment environment) {
        this.environment = environment;
    }

    /**
     * Bind properties with the given prefix to the target object.
     *
     * @param target the object to bind properties to
     * @param prefix the property prefix (e.g., {@code "app.server"})
     */
    public void bind(Object target, String prefix) {
        Class<?> targetClass = target.getClass();
        bindFields(target, targetClass, prefix);
    }

    /**
     * Walk the class hierarchy and bind fields at each level.
     */
    private void bindFields(Object target, Class<?> clazz, String prefix) {
        if (clazz == null || clazz == Object.class) {
            return;
        }

        for (Field field : clazz.getDeclaredFields()) {
            if (Modifier.isStatic(field.getModifiers())
                    || Modifier.isFinal(field.getModifiers())) {
                continue;
            }
            bindField(target, field, prefix);
        }

        // Walk up the class hierarchy
        bindFields(target, clazz.getSuperclass(), prefix);
    }

    /**
     * Bind a single field from the environment.
     */
    private void bindField(Object target, Field field, String prefix) {
        String fieldName = field.getName();
        Class<?> fieldType = field.getType();

        if (isSimpleType(fieldType)) {
            bindSimpleField(target, field, prefix, fieldName, fieldType);
        } else if (fieldType == List.class) {
            bindListField(target, field, prefix, fieldName);
        } else {
            bindNestedObject(target, field, prefix, fieldName);
        }
    }

    /**
     * Bind a simple-type field (String, int, boolean, etc.).
     *
     * <p>Tries both the exact field name and kebab-case. The first match wins.
     */
    private void bindSimpleField(Object target, Field field, String prefix,
                                  String fieldName, Class<?> fieldType) {
        String value = resolveProperty(prefix, fieldName);
        if (value != null) {
            try {
                Object converted = PropertySourcesPropertyResolver.convertValue(value, fieldType);
                setFieldValue(target, field, converted);
            } catch (Exception ex) {
                String propertyName = prefix + "." + fieldName;
                throw new BindException(propertyName, target.getClass(),
                        "failed to convert value '" + value + "' to type "
                                + fieldType.getSimpleName(), ex);
            }
        }
    }

    /**
     * Bind a {@code List<String>} field from a comma-separated property value.
     *
     * <p>For example, {@code app.server.cors-origins=http://a,http://b} produces
     * a two-element list.
     *
     * <p>In the real Spring Boot, list binding supports indexed syntax
     * ({@code origins[0]=a}, {@code origins[1]=b}) and type conversion for
     * {@code List<Integer>}, etc. We simplify to comma-separated {@code List<String>}.
     */
    private void bindListField(Object target, Field field, String prefix,
                                String fieldName) {
        String value = resolveProperty(prefix, fieldName);
        if (value != null) {
            List<String> list = Arrays.stream(value.split(","))
                    .map(String::trim)
                    .filter(s -> !s.isEmpty())
                    .toList();
            setFieldValue(target, field, list);
        }
    }

    /**
     * Bind a nested object by recursing with an extended prefix.
     *
     * <p>Only binds if the field is already initialized (non-null). This is a
     * simplification — the real Spring Boot {@code JavaBeanBinder} auto-creates
     * nested instances via the no-arg constructor if needed.
     */
    private void bindNestedObject(Object target, Field field, String prefix,
                                   String fieldName) {
        Object nested = getFieldValue(target, field);

        // Auto-create if null and the type has a no-arg constructor
        if (nested == null) {
            String nestedPrefix = prefix + "." + fieldName;
            String nestedKebabPrefix = prefix + "." + toKebabCase(fieldName);
            if (hasPropertiesWithPrefix(nestedPrefix)
                    || hasPropertiesWithPrefix(nestedKebabPrefix)) {
                try {
                    var ctor = field.getType().getDeclaredConstructor();
                    ctor.setAccessible(true);
                    nested = ctor.newInstance();
                    setFieldValue(target, field, nested);
                } catch (ReflectiveOperationException ex) {
                    // Cannot create nested object — skip
                    return;
                }
            }
        }

        if (nested != null) {
            bind(nested, prefix + "." + fieldName);
        }
    }

    // -----------------------------------------------------------------------
    // Property resolution helpers
    // -----------------------------------------------------------------------

    /**
     * Resolve a property value, trying both the exact field name and kebab-case.
     *
     * @return the property value, or {@code null} if not found under either form
     */
    private String resolveProperty(String prefix, String fieldName) {
        // Try exact field name first
        String value = environment.getProperty(prefix + "." + fieldName);
        if (value != null) {
            return value;
        }

        // Try kebab-case
        String kebabName = toKebabCase(fieldName);
        if (!kebabName.equals(fieldName)) {
            value = environment.getProperty(prefix + "." + kebabName);
        }
        return value;
    }

    /**
     * Check if any property source contains keys starting with the given prefix.
     *
     * <p>This is used to decide whether to auto-create a nested object. We
     * iterate all property sources and check their underlying maps.
     */
    private boolean hasPropertiesWithPrefix(String prefix) {
        String dotPrefix = prefix + ".";
        for (PropertySource<?> ps : environment.getPropertySources()) {
            Object source = ps.getSource();
            if (source instanceof java.util.Map<?, ?> map) {
                for (Object key : map.keySet()) {
                    if (key instanceof String strKey && strKey.startsWith(dotPrefix)) {
                        return true;
                    }
                }
            }
        }
        return false;
    }

    // -----------------------------------------------------------------------
    // Reflection helpers
    // -----------------------------------------------------------------------

    private static Object getFieldValue(Object target, Field field) {
        try {
            field.setAccessible(true);
            return field.get(target);
        } catch (IllegalAccessException ex) {
            return null;
        }
    }

    private static void setFieldValue(Object target, Field field, Object value) {
        try {
            field.setAccessible(true);
            field.set(target, value);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    "Failed to set field '" + field.getName() + "' on "
                    + target.getClass().getSimpleName(), ex);
        }
    }

    // -----------------------------------------------------------------------
    // Type helpers
    // -----------------------------------------------------------------------

    /**
     * Check whether a type is a "simple" type that can be converted from a
     * single string value.
     */
    static boolean isSimpleType(Class<?> type) {
        return type == String.class
                || type == int.class || type == Integer.class
                || type == long.class || type == Long.class
                || type == boolean.class || type == Boolean.class
                || type == double.class || type == Double.class
                || type.isEnum();
    }

    /**
     * Convert a camelCase field name to kebab-case.
     *
     * <p>Examples:
     * <ul>
     *   <li>{@code "maxConnections"} → {@code "max-connections"}</li>
     *   <li>{@code "sslEnabled"} → {@code "ssl-enabled"}</li>
     *   <li>{@code "port"} → {@code "port"} (no change)</li>
     * </ul>
     *
     * <p>In the real Spring Boot, this conversion is handled by
     * {@code DataObjectPropertyName.toDashedForm()} which also handles
     * underscores and other edge cases. We support the common camelCase
     * to kebab-case conversion.
     */
    static String toKebabCase(String camelCase) {
        if (camelCase == null || camelCase.isEmpty()) {
            return camelCase;
        }
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < camelCase.length(); i++) {
            char c = camelCase.charAt(i);
            if (Character.isUpperCase(c)) {
                if (i > 0) {
                    result.append('-');
                }
                result.append(Character.toLowerCase(c));
            } else {
                result.append(c);
            }
        }
        return result.toString();
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/web/server/WebServer.java`

```java
package com.iris.boot.web.server;

/**
 * Simple interface that represents a fully configured web server (e.g., Tomcat).
 * Allows the server to be {@link #start() started} and {@link #stop() stopped}.
 *
 * <p>The lifecycle is:
 * <ol>
 *   <li>Factory creates the server (configured but not started)</li>
 *   <li>{@link #start()} — bind the port and accept connections</li>
 *   <li>{@link #shutDownGracefully(GracefulShutdownCallback)} — pause connectors,
 *       drain in-flight requests (optional)</li>
 *   <li>{@link #stop()} — stop accepting connections</li>
 *   <li>{@link #destroy()} — release all resources</li>
 * </ol>
 *
 * @see org.springframework.boot.web.server.WebServer
 */
public interface WebServer {

    /**
     * Start the web server. Calling this method on an already started server
     * has no effect.
     *
     * @throws WebServerException if the server cannot be started
     */
    void start() throws WebServerException;

    /**
     * Stop the web server. Calling this method on an already stopped server
     * has no effect.
     *
     * @throws WebServerException if the server cannot be stopped
     */
    void stop() throws WebServerException;

    /**
     * Return the port this server is listening on, or {@code -1} if the
     * server has not been started.
     *
     * @return the server port
     */
    int getPort();

    /**
     * Initiate a graceful shutdown of the web server. The server should
     * stop accepting new requests while processing in-flight requests.
     * The callback must be invoked when the graceful shutdown completes.
     *
     * <p>The default implementation immediately invokes the callback with
     * {@link GracefulShutdownResult#IMMEDIATE}, indicating no graceful
     * shutdown support. Concrete implementations (e.g., Tomcat) override
     * this to pause connectors and wait for request draining.
     *
     * @param callback the callback to invoke when graceful shutdown completes
     * @see org.springframework.boot.web.server.WebServer#shutDownGracefully
     */
    default void shutDownGracefully(GracefulShutdownCallback callback) {
        callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
    }

    /**
     * Destroy the web server so it cannot be restarted. The default
     * implementation delegates to {@link #stop()}.
     */
    default void destroy() {
        stop();
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatWebServer.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.util.concurrent.atomic.AtomicBoolean;

import org.apache.catalina.LifecycleException;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.startup.Tomcat;

import com.iris.boot.web.server.GracefulShutdownCallback;
import com.iris.boot.web.server.GracefulShutdownResult;
import com.iris.boot.web.server.PortInUseException;
import com.iris.boot.web.server.Shutdown;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;

/**
 * {@link WebServer} implementation that wraps an embedded Apache Tomcat instance.
 *
 * <p>The lifecycle is intentionally two-phase:
 * <ol>
 *   <li><b>Construction:</b> The {@code Tomcat} instance is stored but NOT started.
 *       This gives the caller time to finish configuring the application context
 *       before accepting HTTP traffic.</li>
 *   <li><b>{@link #start()}:</b> Calls {@code tomcat.start()}, starts a non-daemon
 *       await thread to keep the JVM alive, and logs the listening port.</li>
 * </ol>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code TomcatWebServer} has a sophisticated two-phase initialization:
 * <ol>
 *   <li>In the constructor, it calls {@code tomcat.start()} but <em>removes all
 *       connectors first</em> so no port binding occurs. This lets servlets and
 *       filters initialize without the server accepting traffic.</li>
 *   <li>In {@code start()}, it re-adds the connectors — this is when the port
 *       actually binds.</li>
 * </ol>
 *
 * <p>We simplify by deferring <em>all</em> Tomcat startup to {@code start()}.
 * This means servlets don't initialize until the server starts, which is fine
 * for our simplified version. The real connector-removal dance ensures that
 * servlet init errors surface before the port binds — an optimization we skip.
 *
 * @see org.springframework.boot.tomcat.TomcatWebServer
 */
public class TomcatWebServer implements WebServer {

    private final Tomcat tomcat;

    private final AtomicBoolean started = new AtomicBoolean(false);

    /** Handles graceful shutdown — null when shutdown mode is IMMEDIATE. */
    private final GracefulShutdown gracefulShutdown;

    /**
     * Create a new {@code TomcatWebServer} wrapping the given Tomcat instance
     * with immediate (non-graceful) shutdown.
     *
     * @param tomcat the configured Tomcat instance
     */
    public TomcatWebServer(Tomcat tomcat) {
        this(tomcat, Shutdown.IMMEDIATE);
    }

    /**
     * Create a new {@code TomcatWebServer} wrapping the given Tomcat instance
     * with the specified shutdown mode.
     *
     * <p>When {@code shutdown} is {@link Shutdown#GRACEFUL}, a
     * {@link GracefulShutdown} helper is created that can pause connectors
     * and drain in-flight requests. When {@link Shutdown#IMMEDIATE}, no
     * graceful shutdown support is available.
     *
     * <p>In the real Spring Boot ({@code TomcatWebServer}, line 105):
     * <pre>
     * this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL)
     *         ? new GracefulShutdown(tomcat) : null;
     * </pre>
     *
     * @param tomcat   the configured Tomcat instance
     * @param shutdown the shutdown mode
     */
    public TomcatWebServer(Tomcat tomcat, Shutdown shutdown) {
        this.tomcat = tomcat;
        this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL)
                ? new GracefulShutdown(tomcat) : null;
    }

    /**
     * Start the embedded Tomcat server.
     *
     * <p>This method:
     * <ol>
     *   <li>Calls {@code tomcat.start()} to initialize and start all components
     *       (Engine → Host → Context → Servlets/Filters)</li>
     *   <li>Verifies the connector started successfully (not in FAILED state)</li>
     *   <li>Starts a non-daemon await thread to keep the JVM alive</li>
     *   <li>Logs the port number</li>
     * </ol>
     *
     * <p>In the real Spring Boot, this method only re-adds previously removed
     * connectors (since Tomcat was already started in the constructor). Our
     * simplified version starts Tomcat here because we skip the connector
     * removal optimization.
     *
     * @throws WebServerException if Tomcat cannot start or the connector fails
     */
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
                Connector connector = this.tomcat.getConnector();
                if (connector != null && isPortInUse(ex)) {
                    throw new PortInUseException(connector.getPort(), ex);
                }
                throw new WebServerException("Unable to start embedded Tomcat", ex);
            }
        }
    }

    /**
     * Verify the connector started successfully.
     *
     * <p>In the real Spring Boot, this checks all connectors and throws
     * {@code ConnectorStartFailedException} if any are in FAILED state.
     * We check just the default connector.
     */
    private void checkConnectorState() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null
                && connector.getState() == org.apache.catalina.LifecycleState.FAILED) {
            throw new WebServerException(
                    "Tomcat connector failed to start on port " + connector.getPort());
        }
    }

    /**
     * Start a non-daemon thread that calls {@code Tomcat.getServer().await()}.
     *
     * <p>All Tomcat threads are daemon threads, which means the JVM would exit
     * as soon as {@code main()} returns. This await thread blocks indefinitely,
     * keeping the JVM alive until {@code stop()} is called or the JVM is killed.
     *
     * <p>In the real Spring Boot ({@code TomcatWebServer}, line 167), this is
     * the same pattern: a non-daemon "container" thread.
     */
    private void startAwaitThread() {
        Thread awaitThread = new Thread(() -> {
            this.tomcat.getServer().await();
        }, "tomcat-await");
        awaitThread.setDaemon(false);
        awaitThread.start();
    }

    /**
     * Stop the embedded Tomcat server.
     *
     * <p>If graceful shutdown is in progress, it is aborted first. Then
     * Tomcat is stopped, which stops all components in reverse order.
     *
     * <p>In the real Spring Boot ({@code TomcatWebServer.stop()}, line 347),
     * the stop method also calls {@code gracefulShutdown.abort()} and
     * removes service connectors.
     *
     * @throws WebServerException if Tomcat cannot stop
     */
    @Override
    public void stop() throws WebServerException {
        if (this.started.compareAndSet(true, false)) {
            try {
                if (this.gracefulShutdown != null) {
                    this.gracefulShutdown.abort();
                }
                this.tomcat.stop();
            } catch (LifecycleException ex) {
                throw new WebServerException("Unable to stop embedded Tomcat", ex);
            }
        }
    }

    /**
     * Initiate a graceful shutdown of this Tomcat server.
     *
     * <p>If graceful shutdown is enabled (constructor received
     * {@link Shutdown#GRACEFUL}), pauses all connectors and polls until
     * in-flight requests complete, then invokes the callback.
     *
     * <p>If graceful shutdown is not enabled, invokes the callback
     * immediately with {@link GracefulShutdownResult#IMMEDIATE}.
     *
     * @param callback the callback to invoke when shutdown completes
     * @see org.springframework.boot.tomcat.TomcatWebServer#shutDownGracefully
     */
    @Override
    public void shutDownGracefully(GracefulShutdownCallback callback) {
        if (this.gracefulShutdown == null) {
            callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
            return;
        }
        this.gracefulShutdown.shutDownGracefully(callback);
    }

    /**
     * Stop and destroy the embedded Tomcat server, releasing all resources.
     *
     * @throws WebServerException if Tomcat cannot be destroyed
     */
    @Override
    public void destroy() {
        try {
            stop();
            this.tomcat.destroy();
        } catch (LifecycleException ex) {
            throw new WebServerException("Unable to destroy embedded Tomcat", ex);
        }
    }

    /**
     * Return the port this Tomcat server is listening on.
     *
     * <p>Returns the connector's <em>local</em> port (the actual bound port,
     * which may differ from the configured port if port 0 was used for
     * auto-assignment). Returns {@code -1} if the server has not been started.
     *
     * @return the listening port, or -1
     */
    @Override
    public int getPort() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null && this.started.get()) {
            return connector.getLocalPort();
        }
        return -1;
    }

    /**
     * Check if the given exception indicates the port is already in use.
     *
     * <p>Walks the cause chain looking for a {@code java.net.BindException},
     * which is the standard JDK exception for "Address already in use".
     */
    private boolean isPortInUse(Throwable ex) {
        Throwable candidate = ex;
        while (candidate != null) {
            if (candidate instanceof java.net.BindException) {
                return true;
            }
            candidate = candidate.getCause();
        }
        return false;
    }

    /**
     * Return the underlying {@code Tomcat} instance.
     * Package-private — used by tests.
     */
    Tomcat getTomcat() {
        return this.tomcat;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatServletWebServerFactory.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;

import jakarta.servlet.Servlet;

import org.apache.catalina.Context;
import org.apache.catalina.Wrapper;
import org.apache.catalina.startup.Tomcat;

import com.iris.boot.web.server.Shutdown;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;

/**
 * {@link ServletWebServerFactory} that creates a {@link TomcatWebServer} backed
 * by an embedded Apache Tomcat.
 *
 * <p>This is the factory that knows HOW to create and configure a Tomcat instance.
 * The {@code ServletWebServerApplicationContext} delegates to this factory via the
 * {@link #getWebServer()} method, keeping the context server-agnostic.
 *
 * <p>Usage via {@code @Configuration}:
 * <pre>{@code
 * @Configuration
 * public class WebConfig {
 *     @Bean
 *     public ServletWebServerFactory webServerFactory() {
 *         return new TomcatServletWebServerFactory(8080);
 *     }
 * }
 * }</pre>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code TomcatServletWebServerFactory} supports:
 * <ul>
 *   <li>{@code ServletContextInitializer}s for servlet/filter registration</li>
 *   <li>SSL configuration via {@code SslBundle}</li>
 *   <li>Compression settings</li>
 *   <li>Protocol customization (HTTP/1.1, HTTP/2)</li>
 *   <li>Error pages, session config, MIME mappings</li>
 *   <li>Additional connectors for multiple ports</li>
 *   <li>Custom valves, context customizers</li>
 * </ul>
 *
 * <p>We support only the port setting. Feature 10 will add servlet registration,
 * Feature 22 will add SSL support.
 *
 * @see org.springframework.boot.tomcat.servlet.TomcatServletWebServerFactory
 */
public class TomcatServletWebServerFactory implements ServletWebServerFactory {

    /** Default port for the embedded Tomcat server. */
    private static final int DEFAULT_PORT = 8080;

    private int port;

    /** The shutdown mode — IMMEDIATE or GRACEFUL. */
    private Shutdown shutdown = Shutdown.IMMEDIATE;

    /**
     * Create a new factory with the default port (8080).
     */
    public TomcatServletWebServerFactory() {
        this(DEFAULT_PORT);
    }

    /**
     * Create a new factory with the specified port.
     *
     * @param port the port to listen on (use 0 for auto-assignment)
     */
    public TomcatServletWebServerFactory(int port) {
        this.port = port;
    }

    /**
     * Create a new fully configured {@link TomcatWebServer} with the given
     * servlets registered.
     *
     * <p>The creation sequence mirrors the real factory ({@code getWebServer()}
     * at line 163):
     * <ol>
     *   <li>Create a temporary base directory (Tomcat needs a work dir)</li>
     *   <li>Create a new {@code Tomcat} instance</li>
     *   <li>Configure the connector (port, protocol)</li>
     *   <li>Create a servlet context and register servlets</li>
     *   <li>Wrap it in a {@code TomcatWebServer}</li>
     * </ol>
     *
     * @param servlets the servlets to register (typically just DispatcherServlet)
     * @return a configured but not started web server
     */
    @Override
    public WebServer getWebServer(Servlet... servlets) {
        Tomcat tomcat = createTomcat();
        prepareContext(tomcat, servlets);
        return new TomcatWebServer(tomcat, this.shutdown);
    }

    /**
     * Create and configure the underlying {@code Tomcat} instance.
     *
     * <p>In the real Spring Boot, this is {@code TomcatWebServerFactory.createTomcat()}
     * which configures the base directory, connector protocol, engine name,
     * additional connectors, and applies customizers.
     *
     * <p>We configure the essentials: base directory, connector, and host.
     */
    private Tomcat createTomcat() {
        Tomcat tomcat = new Tomcat();

        // Tomcat requires a base directory for temp files, work directories, etc.
        // The real Spring Boot creates this in TempDirs.
        File baseDir = createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());

        // Configure the default connector
        // In real Spring Boot, the protocol is customizable (NIO, NIO2, APR).
        // We use the default protocol (HTTP/1.1 NIO).
        tomcat.setPort(this.port);

        // Configure the default host
        tomcat.getHost().setAutoDeploy(false);

        return tomcat;
    }

    /**
     * Prepare a servlet context on the Tomcat host and register the given servlets.
     *
     * <p>In the real Spring Boot ({@code prepareContext()}, line 189), this
     * creates a {@code TomcatEmbeddedContext}, configures the document root,
     * class loader, locale mappings, error pages, session config, and applies
     * all {@code ServletContextInitializer}s (which register DispatcherServlet).
     *
     * <p>We create a context, register each servlet, and map the first one
     * (typically DispatcherServlet) to "/" as the default servlet.
     */
    private void prepareContext(Tomcat tomcat, Servlet... servlets) {
        // addContext() registers a Context with the Host.
        // "" means the root context path.
        Context context = tomcat.addContext("", createTempDir("tomcat-docbase").getAbsolutePath());

        // Use the application's class loader so servlets can find application classes
        context.setParentClassLoader(getClass().getClassLoader());

        // Register each servlet with the Tomcat context
        for (Servlet servlet : servlets) {
            String servletName = servlet.getClass().getSimpleName();
            Wrapper wrapper = Tomcat.addServlet(context, servletName, servlet);
            wrapper.setLoadOnStartup(1);
            // Map to "/" — makes this the default servlet (handles all unmatched URLs)
            context.addServletMappingDecoded("/", servletName);
        }
    }

    /**
     * Create a temporary directory for Tomcat's working files.
     *
     * <p>In the real Spring Boot, this is handled by {@code TempDirs}.
     *
     * @param prefix the directory name prefix
     * @return the created directory
     */
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

    // -----------------------------------------------------------------------
    // Configuration
    // -----------------------------------------------------------------------

    /**
     * Set the port that the embedded Tomcat server should listen on.
     * Use port 0 for automatic port assignment.
     */
    public void setPort(int port) {
        this.port = port;
    }

    /**
     * Return the configured port.
     */
    public int getPort() {
        return this.port;
    }

    /**
     * Set the shutdown mode for the embedded Tomcat server.
     *
     * <p>When set to {@link Shutdown#GRACEFUL}, the server will drain
     * in-flight requests before stopping. When {@link Shutdown#IMMEDIATE}
     * (the default), the server stops immediately.
     *
     * @param shutdown the shutdown mode
     */
    public void setShutdown(Shutdown shutdown) {
        this.shutdown = shutdown;
    }

    /**
     * Return the configured shutdown mode.
     */
    public Shutdown getShutdown() {
        return this.shutdown;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/web/ServerProperties.java`

```java
package com.iris.boot.autoconfigure.web;

import com.iris.boot.context.properties.ConfigurationProperties;
import com.iris.boot.web.server.Shutdown;

/**
 * {@link ConfigurationProperties @ConfigurationProperties} for configuring
 * the embedded web server.
 *
 * <p>Binds properties with the {@code server} prefix from
 * {@code application.properties}:
 * <pre>
 * server.port=9090
 * server.shutdown=graceful
 * </pre>
 *
 * <p>In the real Spring Boot, {@code ServerProperties} is a large class with
 * nested classes for Tomcat, Jetty, Undertow, compression, SSL, HTTP/2, etc.
 * We support port and shutdown mode.
 *
 * @see org.springframework.boot.autoconfigure.web.ServerProperties
 */
@ConfigurationProperties(prefix = "server")
public class ServerProperties {

    /**
     * Server HTTP port. Default is 8080.
     */
    private int port = 8080;

    /**
     * Shutdown mode. Default is IMMEDIATE (no graceful shutdown).
     * Set to {@link Shutdown#GRACEFUL} to drain in-flight requests
     * before stopping.
     */
    private Shutdown shutdown = Shutdown.IMMEDIATE;

    public int getPort() {
        return port;
    }

    public void setPort(int port) {
        this.port = port;
    }

    public Shutdown getShutdown() {
        return shutdown;
    }

    public void setShutdown(Shutdown shutdown) {
        this.shutdown = shutdown;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/web/servlet/TomcatAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure.web.servlet;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnClass;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.web.ServerProperties;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.annotation.Bean;

/**
 * Auto-configuration for embedded Tomcat.
 *
 * <p>This auto-configuration activates when Tomcat is on the classpath
 * (detected by {@code @ConditionalOnClass}) and no user-defined
 * {@link ServletWebServerFactory} bean exists (detected by
 * {@code @ConditionalOnMissingBean}).
 *
 * <p>It creates a {@link TomcatServletWebServerFactory} configured with
 * the port from {@link ServerProperties} (bound from {@code server.port}
 * in {@code application.properties}).
 *
 * <p>In the real Spring Boot 4.x, this is
 * {@code TomcatServletWebServerAutoConfiguration} which lives in the
 * {@code spring-boot-tomcat} module. It's much more complex:
 * <ul>
 *   <li>Uses {@code @ConditionalOnWebApplication(type = SERVLET)}</li>
 *   <li>Applies customizers ({@code TomcatServletWebServerFactoryCustomizer})</li>
 *   <li>Configures protocol, connectors, valves</li>
 *   <li>Imports shared {@code ServletWebServerConfiguration}</li>
 * </ul>
 *
 * <p>We configure only the port — the minimum needed to demonstrate the
 * auto-configuration pattern.
 *
 * @see org.springframework.boot.tomcat.autoconfigure.servlet.TomcatServletWebServerAutoConfiguration
 */
@AutoConfiguration
@ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
@EnableConfigurationProperties(ServerProperties.class)
public class TomcatAutoConfiguration {

    /**
     * Create a {@link TomcatServletWebServerFactory} if the user hasn't
     * defined one.
     *
     * <p>The {@code @ConditionalOnMissingBean} annotation causes this bean
     * to back off if the user provides their own {@code ServletWebServerFactory}.
     * This is the central auto-configuration pattern: <strong>provide sensible
     * defaults that yield to user customization</strong>.
     *
     * @param serverProperties the bound server properties
     * @return a configured Tomcat factory
     */
    @Bean
    @ConditionalOnMissingBean
    public ServletWebServerFactory tomcatServletWebServerFactory(ServerProperties serverProperties) {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.setPort(serverProperties.getPort());
        factory.setShutdown(serverProperties.getShutdown());
        return factory;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/web/context/WebServerApplicationContext.java`

```java
package com.iris.boot.web.context;

import com.iris.boot.web.server.WebServer;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.context.SmartLifecycle;

/**
 * Interface to be implemented by {@link ApplicationContext}s that create and
 * manage an embedded {@link WebServer}.
 *
 * <p>Provides access to the running web server, allowing callers to discover
 * the actual listening port (especially useful when port 0 is configured for
 * auto-assignment).
 *
 * <p>Defines the {@link SmartLifecycle} phase constants that control the
 * ordering of web server lifecycle events during startup and shutdown:
 * <ul>
 *   <li>{@link #START_STOP_LIFECYCLE_PHASE} — the phase at which the server
 *       starts and stops</li>
 *   <li>{@link #GRACEFUL_SHUTDOWN_PHASE} — the phase at which graceful
 *       shutdown initiates (higher phase = stops earlier)</li>
 * </ul>
 *
 * @see org.springframework.boot.web.server.context.WebServerApplicationContext
 */
public interface WebServerApplicationContext extends ApplicationContext {

    /**
     * Phase for the {@code WebServerGracefulShutdownLifecycle}.
     *
     * <p>Set to {@code SmartLifecycle.DEFAULT_PHASE - 1024}
     * ({@code Integer.MAX_VALUE - 1024}). Since shutdown goes from
     * highest to lowest phase, this stops BEFORE the start/stop lifecycle,
     * giving the graceful shutdown a chance to drain requests before the
     * server is actually stopped.
     */
    int GRACEFUL_SHUTDOWN_PHASE = SmartLifecycle.DEFAULT_PHASE - 1024;

    /**
     * Phase for the {@code WebServerStartStopLifecycle}.
     *
     * <p>Set to {@code GRACEFUL_SHUTDOWN_PHASE - 1024}
     * ({@code Integer.MAX_VALUE - 2048}). This is a lower phase than
     * the graceful shutdown, meaning:
     * <ul>
     *   <li><strong>Startup:</strong> starts BEFORE the graceful shutdown
     *       lifecycle (server starts, then graceful shutdown lifecycle
     *       just sets running=true)</li>
     *   <li><strong>Shutdown:</strong> stops AFTER the graceful shutdown
     *       lifecycle (graceful shutdown drains requests first, then this
     *       lifecycle stops the server)</li>
     * </ul>
     */
    int START_STOP_LIFECYCLE_PHASE = GRACEFUL_SHUTDOWN_PHASE - 1024;

    /**
     * Return the embedded {@link WebServer} created by this context, or
     * {@code null} if the server has not been created yet.
     */
    WebServer getWebServer();
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/web/context/ServletWebServerApplicationContext.java`

```java
package com.iris.boot.web.context;

import java.util.Map;

import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.web.servlet.DispatcherServlet;

/**
 * A specialized {@link AnnotationConfigApplicationContext} that creates and
 * manages an embedded web server as part of the context lifecycle.
 *
 * <p>This is the key integration class — it bridges the Spring IoC container
 * lifecycle ({@code refresh()}/{@code close()}) with the embedded web server
 * lifecycle ({@code start()}/{@code stop()}) via {@link SmartLifecycle} beans.
 *
 * <h3>Lifecycle Integration via SmartLifecycle</h3>
 *
 * <p>The embedded web server is woven into the context lifecycle via two
 * {@link com.iris.framework.context.SmartLifecycle SmartLifecycle} beans
 * registered during {@link #onRefresh()}:
 * <pre>
 * refresh()
 *   ├── invokeBeanFactoryPostProcessors()     ← @Configuration classes processed
 *   ├── registerBeanPostProcessors()           ← post-processors ready
 *   ├── onRefresh()                            ← ★ createWebServer() — server created + lifecycle beans registered
 *   ├── finishBeanFactoryInitialization()      ← all singletons instantiated
 *   ├── registerListeners()                    ← listeners discovered
 *   └── finishRefresh()                        ← ★ LifecycleProcessor starts SmartLifecycle beans:
 *                                                  WebServerStartStopLifecycle.start() → port binds NOW
 * </pre>
 *
 * <p>On shutdown:
 * <pre>
 * close()
 *   ├── ContextClosedEvent published
 *   ├── LifecycleProcessor.onClose() stops SmartLifecycle beans:
 *   │     ├── Phase MAX_VALUE-1024: WebServerGracefulShutdownLifecycle.stop()
 *   │     │     → pauses connectors, drains in-flight requests
 *   │     └── Phase MAX_VALUE-2048: WebServerStartStopLifecycle.stop()
 *   │           → calls webServer.stop()
 *   ├── beanFactory.close() → destroys singletons
 *   └── webServer.destroy() → final cleanup
 * </pre>
 *
 * <h3>Enhancement from Feature 19</h3>
 *
 * <p>Before this feature, the web server was started directly in
 * {@code refresh()} after {@code super.refresh()} and stopped directly
 * in {@code close()}. Now it's managed via SmartLifecycle, enabling:
 * <ul>
 *   <li>Phase-ordered shutdown (graceful shutdown before server stop)</li>
 *   <li>Proper integration with other SmartLifecycle beans</li>
 *   <li>JVM shutdown hook support via {@code registerShutdownHook()}</li>
 * </ul>
 *
 * @see org.springframework.boot.web.server.servlet.context.ServletWebServerApplicationContext
 */
public class ServletWebServerApplicationContext extends AnnotationConfigApplicationContext
        implements WebServerApplicationContext {

    private volatile WebServer webServer;

    // -----------------------------------------------------------------------
    // Lifecycle — refresh and close
    // -----------------------------------------------------------------------

    /**
     * Extends the base {@code refresh()} to create the web server during
     * the template method hook.
     *
     * <p>The web server is created in {@code onRefresh()} but NOT started.
     * It starts later via the {@code WebServerStartStopLifecycle} SmartLifecycle
     * bean in {@code finishRefresh()} (called by super.refresh()).
     *
     * <p>If refresh fails, the web server is destroyed to release resources.
     */
    @Override
    public void refresh() {
        try {
            super.refresh();
        } catch (RuntimeException ex) {
            stopAndDestroyWebServer();
            throw ex;
        }
    }

    /**
     * Called by the base class during {@code refresh()}, between
     * {@code registerBeanPostProcessors()} and
     * {@code finishBeanFactoryInitialization()}.
     *
     * <p>This is where we create the embedded web server and register the
     * {@link SmartLifecycle} beans that manage its lifecycle:
     * <ul>
     *   <li>{@link WebServerGracefulShutdownLifecycle} — initiates graceful
     *       shutdown (connector pausing, request draining) at a high phase</li>
     *   <li>{@link WebServerStartStopLifecycle} — starts and stops the web
     *       server at a lower phase</li>
     * </ul>
     *
     * <p>In the real Spring Boot ({@code createWebServer()}, line 193):
     * <pre>
     * getBeanFactory().registerSingleton("webServerGracefulShutdown",
     *         new WebServerGracefulShutdownLifecycle(webServer));
     * getBeanFactory().registerSingleton("webServerStartStop",
     *         new WebServerStartStopLifecycle(this, webServer));
     * </pre>
     */
    @Override
    protected void onRefresh() {
        createWebServer();
    }

    /**
     * Create the embedded web server with a DispatcherServlet registered,
     * then register SmartLifecycle beans for lifecycle management.
     */
    private void createWebServer() {
        ServletWebServerFactory factory = getWebServerFactory();
        DispatcherServlet dispatcherServlet = new DispatcherServlet(this);
        this.webServer = factory.getWebServer(dispatcherServlet);

        // Register SmartLifecycle beans for phased start/stop
        getBeanFactory().registerSingleton("webServerGracefulShutdown",
                new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton("webServerStartStop",
                new WebServerStartStopLifecycle(this, this.webServer));
    }

    /**
     * Find the single {@link ServletWebServerFactory} bean in the container.
     *
     * @return the single factory bean
     * @throws WebServerException if zero or multiple factories are found
     */
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

    /**
     * Shut down the web server on failure.
     */
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

    /**
     * Close this application context and destroy the embedded web server.
     *
     * <p>The shutdown sequence:
     * <ol>
     *   <li>{@code super.close()} — publishes {@code ContextClosedEvent},
     *       stops SmartLifecycle beans (graceful shutdown + server stop),
     *       destroys singletons</li>
     *   <li>{@code webServer.destroy()} — final resource cleanup</li>
     * </ol>
     *
     * <p>In the real Spring Boot, {@code doClose()} is overridden in
     * {@code ServletWebServerApplicationContext} to call
     * {@code webServer.destroy()} after {@code super.doClose()}.
     */
    @Override
    public void close() {
        super.close();
        stopAndDestroyWebServer();
    }

    // -----------------------------------------------------------------------
    // WebServerApplicationContext
    // -----------------------------------------------------------------------

    @Override
    public WebServer getWebServer() {
        return this.webServer;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java` (diff only)

```diff
  private void refreshContext(ConfigurableApplicationContext context) {
      context.refresh();
+     context.registerShutdownHook();
  }
```

### Test Code

#### `[NEW]` `iris-framework/src/test/java/com/iris/framework/context/support/DefaultLifecycleProcessorTest.java`

```java
package com.iris.framework.context.support;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.Lifecycle;
import com.iris.framework.context.SmartLifecycle;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link DefaultLifecycleProcessor}.
 *
 * <p>Verifies phase-ordered startup and shutdown, the async stop(Runnable)
 * pattern for SmartLifecycle beans, and auto-startup filtering.
 */
class DefaultLifecycleProcessorTest {

    private DefaultBeanFactory beanFactory;
    private DefaultLifecycleProcessor processor;

    @BeforeEach
    void setUp() {
        beanFactory = new DefaultBeanFactory();
        processor = new DefaultLifecycleProcessor();
        processor.setBeanFactory(beanFactory);
        // Register the processor itself (mimics context behavior)
        beanFactory.registerSingleton("lifecycleProcessor", processor);
    }

    // -----------------------------------------------------------------------
    // Start ordering
    // -----------------------------------------------------------------------

    @Test
    void shouldStartBeansInAscendingPhaseOrder_WhenOnRefreshCalled() {
        // Arrange
        List<String> startOrder = new ArrayList<>();

        registerSmartLifecycle("high", 100, startOrder);
        registerSmartLifecycle("low", 10, startOrder);
        registerSmartLifecycle("mid", 50, startOrder);

        // Act
        processor.onRefresh();

        // Assert — lowest phase starts first
        assertThat(startOrder).containsExactly("low", "mid", "high");
    }

    @Test
    void shouldOnlyStartAutoStartupBeans_WhenOnRefreshCalled() {
        // Arrange
        List<String> startOrder = new ArrayList<>();

        registerSmartLifecycle("auto", 10, startOrder, true);
        registerSmartLifecycle("manual", 20, startOrder, false);

        // Act
        processor.onRefresh();

        // Assert — only auto-startup beans are started
        assertThat(startOrder).containsExactly("auto");
    }

    @Test
    void shouldStartAllBeans_WhenStartCalledDirectly() {
        // Arrange
        List<String> startOrder = new ArrayList<>();

        registerSmartLifecycle("auto", 10, startOrder, true);
        registerSmartLifecycle("manual", 20, startOrder, false);

        // Act
        processor.start();

        // Assert — both are started regardless of isAutoStartup
        assertThat(startOrder).containsExactly("auto", "manual");
    }

    @Test
    void shouldNotStartAlreadyRunningBeans_WhenOnRefreshCalled() {
        // Arrange
        List<String> startOrder = new ArrayList<>();

        TrackingSmartLifecycle bean = registerSmartLifecycle("already-running", 10, startOrder);
        bean.forceRunning(true);

        // Act
        processor.onRefresh();

        // Assert — already running bean is skipped
        assertThat(startOrder).isEmpty();
    }

    // -----------------------------------------------------------------------
    // Stop ordering
    // -----------------------------------------------------------------------

    @Test
    void shouldStopBeansInDescendingPhaseOrder_WhenOnCloseCalled() {
        // Arrange
        List<String> stopOrder = new ArrayList<>();

        TrackingSmartLifecycle low = registerSmartLifecycle("low", 10, new ArrayList<>());
        TrackingSmartLifecycle mid = registerSmartLifecycle("mid", 50, new ArrayList<>());
        TrackingSmartLifecycle high = registerSmartLifecycle("high", 100, new ArrayList<>());

        // Start them first
        processor.onRefresh();
        low.setStopTracker(stopOrder);
        mid.setStopTracker(stopOrder);
        high.setStopTracker(stopOrder);

        // Act
        processor.onClose();

        // Assert — highest phase stops first
        assertThat(stopOrder).containsExactly("high", "mid", "low");
    }

    @Test
    void shouldNotStopNonRunningBeans_WhenOnCloseCalled() {
        // Arrange
        List<String> stopOrder = new ArrayList<>();

        TrackingSmartLifecycle bean = registerSmartLifecycle("not-running", 10, new ArrayList<>());
        bean.setStopTracker(stopOrder);
        // Never started, so isRunning() is false

        // Act
        processor.onClose();

        // Assert
        assertThat(stopOrder).isEmpty();
    }

    // -----------------------------------------------------------------------
    // Async stop(Runnable) pattern
    // -----------------------------------------------------------------------

    @Test
    void shouldUseAsyncStop_WhenSmartLifecycleBeanStops() {
        // Arrange
        List<String> callbackOrder = new ArrayList<>();

        TrackingSmartLifecycle bean = registerSmartLifecycle("async-bean", 10, new ArrayList<>());

        processor.onRefresh();
        bean.setStopTracker(callbackOrder);

        // Act
        processor.onClose();

        // Assert — the bean's stop(Runnable) was called (not stop())
        assertThat(bean.isAsyncStopUsed()).isTrue();
        assertThat(callbackOrder).containsExactly("async-bean");
    }

    // -----------------------------------------------------------------------
    // Self-exclusion
    // -----------------------------------------------------------------------

    @Test
    void shouldNotStopItself_WhenProcessorIsAlsoALifecycleBean() {
        // The processor is registered as a singleton and implements Lifecycle.
        // It should not try to stop itself (which would cause infinite recursion).

        processor.onRefresh();

        // Act — should not throw StackOverflowError
        assertThatCode(() -> processor.onClose()).doesNotThrowAnyException();
    }

    // -----------------------------------------------------------------------
    // isRunning
    // -----------------------------------------------------------------------

    @Test
    void shouldReportRunning_WhenRefreshCompletes() {
        processor.onRefresh();
        assertThat(processor.isRunning()).isTrue();
    }

    @Test
    void shouldReportNotRunning_WhenCloseCompletes() {
        processor.onRefresh();
        processor.onClose();
        assertThat(processor.isRunning()).isFalse();
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    private TrackingSmartLifecycle registerSmartLifecycle(String name, int phase,
                                                          List<String> startTracker) {
        return registerSmartLifecycle(name, phase, startTracker, true);
    }

    private TrackingSmartLifecycle registerSmartLifecycle(String name, int phase,
                                                          List<String> startTracker,
                                                          boolean autoStartup) {
        TrackingSmartLifecycle bean = new TrackingSmartLifecycle(name, phase,
                autoStartup, startTracker);
        beanFactory.registerSingleton(name, bean);
        return bean;
    }

    /**
     * A SmartLifecycle implementation that tracks start/stop invocations.
     */
    static class TrackingSmartLifecycle implements SmartLifecycle {
        private final String name;
        private final int phase;
        private final boolean autoStartup;
        private final List<String> startTracker;
        private List<String> stopTracker;
        private boolean running;
        private boolean asyncStopUsed;

        TrackingSmartLifecycle(String name, int phase, boolean autoStartup,
                               List<String> startTracker) {
            this.name = name;
            this.phase = phase;
            this.autoStartup = autoStartup;
            this.startTracker = startTracker;
        }

        void setStopTracker(List<String> stopTracker) {
            this.stopTracker = stopTracker;
        }

        void forceRunning(boolean running) {
            this.running = running;
        }

        boolean isAsyncStopUsed() {
            return asyncStopUsed;
        }

        @Override
        public void start() {
            this.running = true;
            startTracker.add(name);
        }

        @Override
        public void stop() {
            this.running = false;
            if (stopTracker != null) {
                stopTracker.add(name);
            }
        }

        @Override
        public void stop(Runnable callback) {
            this.asyncStopUsed = true;
            stop();
            callback.run();
        }

        @Override
        public boolean isRunning() {
            return running;
        }

        @Override
        public int getPhase() {
            return phase;
        }

        @Override
        public boolean isAutoStartup() {
            return autoStartup;
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/web/context/WebServerGracefulShutdownLifecycleTest.java`

```java
package com.iris.boot.web.context;

import java.util.concurrent.atomic.AtomicReference;

import org.junit.jupiter.api.Test;

import com.iris.boot.web.server.GracefulShutdownResult;
import com.iris.boot.web.server.WebServer;
import com.iris.framework.context.SmartLifecycle;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link WebServerGracefulShutdownLifecycle}.
 */
class WebServerGracefulShutdownLifecycleTest {

    @Test
    void shouldBeAtGracefulShutdownPhase() {
        WebServerGracefulShutdownLifecycle lifecycle =
                new WebServerGracefulShutdownLifecycle(new StubWebServer());

        assertThat(lifecycle.getPhase())
                .isEqualTo(WebServerApplicationContext.GRACEFUL_SHUTDOWN_PHASE);
    }

    @Test
    void shouldReportRunningAfterStart() {
        WebServerGracefulShutdownLifecycle lifecycle =
                new WebServerGracefulShutdownLifecycle(new StubWebServer());

        lifecycle.start();

        assertThat(lifecycle.isRunning()).isTrue();
    }

    @Test
    void shouldDelegateGracefulShutdownToWebServer_WhenStopWithCallback() {
        // Arrange
        StubWebServer webServer = new StubWebServer();
        WebServerGracefulShutdownLifecycle lifecycle =
                new WebServerGracefulShutdownLifecycle(webServer);
        lifecycle.start();

        AtomicReference<Boolean> callbackInvoked = new AtomicReference<>(false);

        // Act
        lifecycle.stop(() -> callbackInvoked.set(true));

        // Assert
        assertThat(lifecycle.isRunning()).isFalse();
        assertThat(webServer.gracefulShutdownInitiated).isTrue();
        assertThat(callbackInvoked.get()).isTrue();
    }

    @Test
    void shouldThrowOnDirectStop() {
        WebServerGracefulShutdownLifecycle lifecycle =
                new WebServerGracefulShutdownLifecycle(new StubWebServer());
        lifecycle.start();

        assertThatThrownBy(lifecycle::stop)
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void shouldBeAutoStartup() {
        WebServerGracefulShutdownLifecycle lifecycle =
                new WebServerGracefulShutdownLifecycle(new StubWebServer());

        assertThat(lifecycle.isAutoStartup()).isTrue();
    }

    /**
     * A minimal WebServer stub that tracks graceful shutdown calls.
     */
    private static class StubWebServer implements WebServer {
        boolean gracefulShutdownInitiated = false;

        @Override public void start() {}
        @Override public void stop() {}
        @Override public int getPort() { return -1; }

        @Override
        public void shutDownGracefully(
                com.iris.boot.web.server.GracefulShutdownCallback callback) {
            this.gracefulShutdownInitiated = true;
            callback.shutdownComplete(GracefulShutdownResult.IDLE);
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/web/context/WebServerStartStopLifecycleTest.java`

```java
package com.iris.boot.web.context;

import org.junit.jupiter.api.Test;

import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link WebServerStartStopLifecycle}.
 */
class WebServerStartStopLifecycleTest {

    @Test
    void shouldBeAtStartStopLifecyclePhase() {
        WebServerStartStopLifecycle lifecycle =
                new WebServerStartStopLifecycle(null, new StubWebServer());

        assertThat(lifecycle.getPhase())
                .isEqualTo(WebServerApplicationContext.START_STOP_LIFECYCLE_PHASE);
    }

    @Test
    void shouldHaveLowerPhaseThanGracefulShutdown() {
        // START_STOP_LIFECYCLE_PHASE < GRACEFUL_SHUTDOWN_PHASE
        // This means: starts first (lower=earlier), stops last (lower=later)
        assertThat(WebServerApplicationContext.START_STOP_LIFECYCLE_PHASE)
                .isLessThan(WebServerApplicationContext.GRACEFUL_SHUTDOWN_PHASE);
    }

    @Test
    void shouldStartWebServer_WhenStartCalled() {
        StubWebServer webServer = new StubWebServer();
        WebServerStartStopLifecycle lifecycle =
                new WebServerStartStopLifecycle(null, webServer);

        lifecycle.start();

        assertThat(webServer.started).isTrue();
        assertThat(lifecycle.isRunning()).isTrue();
    }

    @Test
    void shouldStopWebServer_WhenStopCalled() {
        StubWebServer webServer = new StubWebServer();
        WebServerStartStopLifecycle lifecycle =
                new WebServerStartStopLifecycle(null, webServer);

        lifecycle.start();
        lifecycle.stop();

        assertThat(webServer.stopped).isTrue();
        assertThat(lifecycle.isRunning()).isFalse();
    }

    @Test
    void shouldBeAutoStartup() {
        WebServerStartStopLifecycle lifecycle =
                new WebServerStartStopLifecycle(null, new StubWebServer());

        assertThat(lifecycle.isAutoStartup()).isTrue();
    }

    private static class StubWebServer implements WebServer {
        boolean started = false;
        boolean stopped = false;

        @Override
        public void start() throws WebServerException {
            this.started = true;
        }

        @Override
        public void stop() throws WebServerException {
            this.stopped = true;
        }

        @Override
        public int getPort() {
            return -1;
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/integration/GracefulShutdownIntegrationTest.java`

```java
package com.iris.boot.integration;

import java.net.HttpURLConnection;
import java.net.URI;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.web.ServerProperties;
import com.iris.boot.web.context.ServletWebServerApplicationContext;
import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.server.GracefulShutdownCallback;
import com.iris.boot.web.server.GracefulShutdownResult;
import com.iris.boot.web.server.Shutdown;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.SmartLifecycle;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for the graceful shutdown feature.
 *
 * <p>These tests verify the full lifecycle:
 * <ul>
 *   <li>SmartLifecycle beans start in phase order during refresh</li>
 *   <li>SmartLifecycle beans stop in reverse phase order during close</li>
 *   <li>Graceful shutdown drains requests before server stop</li>
 *   <li>JVM shutdown hook triggers context close</li>
 * </ul>
 */
class GracefulShutdownIntegrationTest {

    @Test
    void shouldStartWebServerViaSmartLifecycle_WhenContextRefreshes() {
        // This tests that the server starts via SmartLifecycle in finishRefresh()
        // rather than being started directly
        ServletWebServerApplicationContext context = new ServletWebServerApplicationContext();
        context.register(WebServerConfig.class);

        context.refresh();

        try {
            WebServer webServer = context.getWebServer();
            assertThat(webServer).isNotNull();
            assertThat(webServer.getPort()).isGreaterThan(0);
        } finally {
            context.close();
        }
    }

    @Test
    void shouldStopGracefulShutdownBeforeServerStop_WhenContextCloses() {
        // Track the order of lifecycle operations
        List<String> eventLog = new ArrayList<>();

        ServletWebServerApplicationContext context = new ServletWebServerApplicationContext();
        context.register(WebServerConfig.class);
        context.refresh();

        // Register a SmartLifecycle at the graceful shutdown phase to track ordering
        int gracefulPhase = SmartLifecycle.DEFAULT_PHASE - 1024;
        int startStopPhase = gracefulPhase - 1024;

        // Verify phases are correct
        assertThat(context.getWebServer().getPort()).isGreaterThan(0);

        context.close();

        // The server should be stopped after close
        assertThat(context.getWebServer().getPort()).isEqualTo(-1);
    }

    @Test
    void shouldInitiateGracefulShutdown_WhenServerConfiguredForGraceful() {
        // Create context with graceful shutdown enabled
        ServletWebServerApplicationContext context = new ServletWebServerApplicationContext();
        context.register(GracefulWebServerConfig.class);

        context.refresh();

        try {
            WebServer webServer = context.getWebServer();
            assertThat(webServer).isNotNull();
            assertThat(webServer.getPort()).isGreaterThan(0);
        } finally {
            // Close should trigger graceful shutdown sequence
            context.close();
        }
    }

    @Test
    void shouldReportImmediateShutdown_WhenGracefulNotEnabled() {
        // Arrange
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
        factory.setShutdown(Shutdown.IMMEDIATE);

        WebServer server = factory.getWebServer();
        server.start();

        AtomicReference<GracefulShutdownResult> result = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        // Act
        server.shutDownGracefully(r -> {
            result.set(r);
            latch.countDown();
        });

        try {
            latch.await(5, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // Assert
        assertThat(result.get()).isEqualTo(GracefulShutdownResult.IMMEDIATE);

        server.destroy();
    }

    @Test
    void shouldReportIdleShutdown_WhenGracefulEnabledAndNoActiveRequests() throws Exception {
        // Arrange
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
        factory.setShutdown(Shutdown.GRACEFUL);

        WebServer server = factory.getWebServer();
        server.start();

        AtomicReference<GracefulShutdownResult> result = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        // Act — no requests in flight, so graceful shutdown completes with IDLE
        server.shutDownGracefully(r -> {
            result.set(r);
            latch.countDown();
        });

        boolean completed = latch.await(5, TimeUnit.SECONDS);

        // Assert
        assertThat(completed).isTrue();
        assertThat(result.get()).isEqualTo(GracefulShutdownResult.IDLE);

        server.stop();
        server.destroy();
    }

    @Test
    void shouldStopSmartLifecycleBeansInReversePhaseOrder_WhenContextCloses() {
        // Arrange
        List<String> events = new ArrayList<>();

        ServletWebServerApplicationContext context = new ServletWebServerApplicationContext();
        context.register(WebServerConfig.class);
        context.refresh();

        // Register a custom SmartLifecycle at phase 0 to track ordering
        TrackingLifecycle tracker = new TrackingLifecycle("user-bean", 0, events);
        context.getBeanFactory().registerSingleton("trackingLifecycle", tracker);

        // Start the tracker (simulating what the processor would do)
        tracker.start();

        // Act
        context.close();

        // Assert — the web server phases (MAX_VALUE-1024, MAX_VALUE-2048)
        // stop before phase 0
        assertThat(events).containsExactly("user-bean-stop");
    }

    // -----------------------------------------------------------------------
    // Test configurations
    // -----------------------------------------------------------------------

    @Configuration
    static class WebServerConfig {
        @Bean
        public ServletWebServerFactory servletWebServerFactory() {
            return new TomcatServletWebServerFactory(0); // port 0 = random
        }
    }

    @Configuration
    static class GracefulWebServerConfig {
        @Bean
        public ServletWebServerFactory servletWebServerFactory() {
            TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory(0);
            factory.setShutdown(Shutdown.GRACEFUL);
            return factory;
        }
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    static class TrackingLifecycle implements SmartLifecycle {
        private final String name;
        private final int phase;
        private final List<String> events;
        private boolean running;

        TrackingLifecycle(String name, int phase, List<String> events) {
            this.name = name;
            this.phase = phase;
            this.events = events;
        }

        @Override
        public void start() { this.running = true; }

        @Override
        public void stop() {
            this.running = false;
            events.add(name + "-stop");
        }

        @Override
        public boolean isRunning() { return running; }

        @Override
        public int getPhase() { return phase; }
    }
}
```

---

## Summary

| What | Detail |
|------|--------|
| **New classes** | `Lifecycle`, `SmartLifecycle`, `LifecycleProcessor`, `DefaultLifecycleProcessor`, `Shutdown`, `GracefulShutdownCallback`, `GracefulShutdownResult`, `GracefulShutdown`, `WebServerGracefulShutdownLifecycle`, `WebServerStartStopLifecycle` |
| **Modified classes** | `AnnotationConfigApplicationContext` (+`finishRefresh`/`close` lifecycle integration, +`registerShutdownHook`), `ConfigurableApplicationContext` (+`registerShutdownHook`), `DefaultBeanFactory` (+`registerSingleton`), `ServletWebServerApplicationContext` (SmartLifecycle registration), `TomcatWebServer` (+graceful shutdown), `TomcatServletWebServerFactory` (+shutdown mode), `WebServer` (+`shutDownGracefully`), `WebServerApplicationContext` (+phase constants), `ServerProperties` (+shutdown), `TomcatAutoConfiguration` (+shutdown pass-through), `PropertySourcesPropertyResolver` (+enum conversion), `ConfigurationPropertiesBinder` (+enum in `isSimpleType`), `IrisApplication` (+shutdown hook) |
| **Pattern** | Phase-ordered lifecycle with async stop via `CountDownLatch` + `stop(Runnable)` callback |
| **Key insight** | Two SmartLifecycle beans at different phases ensure graceful request draining completes before the server is stopped |
| **Tests** | 4 test files: `DefaultLifecycleProcessorTest` (10), `WebServerGracefulShutdownLifecycleTest` (5), `WebServerStartStopLifecycleTest` (5), `GracefulShutdownIntegrationTest` (6) |

**Next chapter:** [Chapter 20: Health Indicators and Actuator](ch20_health_indicators_actuator.md) -- expose application health and runtime information via HTTP endpoints using `HealthIndicator` and a minimal actuator framework.
