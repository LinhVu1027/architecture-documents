# Chapter 8: IrisApplication Bootstrap

## Build Challenge

| Current State | Limitation | Objective |
|--------------|-----------|-----------|
| `AnnotationConfigApplicationContext` manages the full container lifecycle | Users must manually create the context, set the environment, call `refresh()`, and manage shutdown themselves | Create `IrisApplication.run()` — a single entry point that detects the web type, prepares the environment, creates the context, fires lifecycle events, runs post-startup callbacks, and prints a banner |

---

## 8.1 The Integration Point

The integration point is `IrisApplication.run()` calling the framework's `AnnotationConfigApplicationContext`. The bootstrap layer creates the context, configures it externally (sets environment, registers bean definitions), and then triggers `refresh()`. This is where **iris-boot** first connects to **iris-framework**.

**Direction:** We need to build from the outside in — first modify the framework to accept external configuration (no-arg constructor, `register()`, `setEnvironment()`), then build the bootstrap orchestrator (`IrisApplication`) and its supporting infrastructure (events, listeners, runners, web type detection).

To make this connection work, we must enhance the framework's `ConfigurableApplicationContext` interface with two new methods:

**Modifying:** `iris-framework/.../context/ConfigurableApplicationContext.java`
**Change:** Add `setEnvironment()` and `addApplicationListener()` to the interface

```java
void setEnvironment(Environment environment);

void addApplicationListener(ApplicationListener<?> listener);
```

**Modifying:** `iris-framework/.../context/AnnotationConfigApplicationContext.java`
**Change:** Add no-arg constructor (deferred refresh), `register()` method, and `setEnvironment()` implementation

```java
// No-arg constructor — creates the context without refreshing
public AnnotationConfigApplicationContext() {
    this.beanFactory = new DefaultBeanFactory();
    this.environment = createEnvironment();
}

// Convenience constructor delegates to no-arg + register + refresh
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses);
    refresh();
}

// Register classes BEFORE refresh — this is what IrisApplication uses
public void register(Class<?>... componentClasses) {
    for (Class<?> componentClass : componentClasses) {
        String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
        beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
    }
}
```

The field type also changes from `private final StandardEnvironment environment` to `private Environment environment` — making it settable and accepting any `Environment` implementation.

**Key decision:** The no-arg constructor still creates a default `StandardEnvironment`. When `IrisApplication` calls `setEnvironment()`, it replaces the default with the externally-prepared one (which already has `application.properties` loaded and lifecycle events fired). This separation is crucial — it's how Spring Boot controls what happens BEFORE the container starts.

---

## 8.2 The Core Implementation

### WebApplicationType — Classpath Detection

**New file:** `iris-boot-core/.../boot/WebApplicationType.java`

```java
package com.iris.boot;

public enum WebApplicationType {
    NONE,
    SERVLET;

    private static final String TOMCAT_INDICATOR_CLASS = "org.apache.catalina.startup.Tomcat";
    private static final String SERVLET_INDICATOR_CLASS = "jakarta.servlet.Servlet";

    public static WebApplicationType deduceFromClasspath() {
        if (isPresent(TOMCAT_INDICATOR_CLASS) && isPresent(SERVLET_INDICATOR_CLASS)) {
            return SERVLET;
        }
        return NONE;
    }

    private static boolean isPresent(String className) {
        try {
            Class.forName(className, false, WebApplicationType.class.getClassLoader());
            return true;
        } catch (ClassNotFoundException ex) {
            return false;
        }
    }
}
```

### ApplicationArguments — Parsed Command-Line Args

**New file:** `iris-boot-core/.../boot/ApplicationArguments.java`

```java
package com.iris.boot;

import java.util.List;
import java.util.Set;

public interface ApplicationArguments {
    String[] getSourceArgs();
    Set<String> getOptionNames();
    boolean containsOption(String name);
    List<String> getOptionValues(String name);
    List<String> getNonOptionArgs();
}
```

**New file:** `iris-boot-core/.../boot/DefaultApplicationArguments.java`

```java
package com.iris.boot;

import java.util.*;

public class DefaultApplicationArguments implements ApplicationArguments {
    private final String[] sourceArgs;
    private final Map<String, List<String>> optionArgs = new LinkedHashMap<>();
    private final List<String> nonOptionArgs = new ArrayList<>();

    public DefaultApplicationArguments(String... args) {
        this.sourceArgs = args.clone();
        parse(args);
    }

    private void parse(String[] args) {
        for (String arg : args) {
            if (arg.startsWith("--")) {
                String withoutDashes = arg.substring(2);
                int equalsIndex = withoutDashes.indexOf('=');
                if (equalsIndex >= 0) {
                    String name = withoutDashes.substring(0, equalsIndex);
                    String value = withoutDashes.substring(equalsIndex + 1);
                    this.optionArgs.computeIfAbsent(name, k -> new ArrayList<>()).add(value);
                } else {
                    this.optionArgs.computeIfAbsent(withoutDashes, k -> new ArrayList<>());
                }
            } else {
                this.nonOptionArgs.add(arg);
            }
        }
    }

    @Override public String[] getSourceArgs() { return this.sourceArgs.clone(); }
    @Override public Set<String> getOptionNames() {
        return Collections.unmodifiableSet(new LinkedHashSet<>(this.optionArgs.keySet()));
    }
    @Override public boolean containsOption(String name) { return this.optionArgs.containsKey(name); }
    @Override public List<String> getOptionValues(String name) {
        List<String> values = this.optionArgs.get(name);
        return (values != null) ? Collections.unmodifiableList(values) : null;
    }
    @Override public List<String> getNonOptionArgs() {
        return Collections.unmodifiableList(this.nonOptionArgs);
    }
}
```

### Runner Interfaces — Post-Startup Callbacks

**New file:** `iris-boot-core/.../boot/ApplicationRunner.java`

```java
package com.iris.boot;

@FunctionalInterface
public interface ApplicationRunner {
    void run(ApplicationArguments args) throws Exception;
}
```

**New file:** `iris-boot-core/.../boot/CommandLineRunner.java`

```java
package com.iris.boot;

@FunctionalInterface
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

---

## 8.3 The Event System

### Event Hierarchy

**New file:** `iris-boot-core/.../boot/context/event/IrisApplicationEvent.java`

```java
package com.iris.boot.context.event;

import com.iris.framework.context.ApplicationEvent;
import com.iris.boot.IrisApplication;

public abstract class IrisApplicationEvent extends ApplicationEvent {
    private final String[] args;

    protected IrisApplicationEvent(IrisApplication application, String[] args) {
        super(application);
        this.args = args;
    }

    public IrisApplication getIrisApplication() {
        return (IrisApplication) getSource();
    }

    public String[] getArgs() { return this.args; }
}
```

Five concrete events — one for each lifecycle stage:

| Event | When Fired | Context Available? |
|-------|-----------|-------------------|
| `ApplicationStartingEvent` | Immediately when `run()` starts | No |
| `ApplicationEnvironmentPreparedEvent` | After environment is prepared | No |
| `ApplicationStartedEvent` | After `context.refresh()` | Yes |
| `ApplicationReadyEvent` | After runners complete | Yes |
| `ApplicationFailedEvent` | On any failure | Maybe |

```java
// ApplicationStartingEvent — before anything happens
public class ApplicationStartingEvent extends IrisApplicationEvent {
    public ApplicationStartingEvent(IrisApplication app, String[] args) {
        super(app, args);
    }
}

// ApplicationEnvironmentPreparedEvent — environment ready, no context yet
public class ApplicationEnvironmentPreparedEvent extends IrisApplicationEvent {
    private final Environment environment;
    public ApplicationEnvironmentPreparedEvent(IrisApplication app, String[] args, Environment env) {
        super(app, args);
        this.environment = env;
    }
    public Environment getEnvironment() { return this.environment; }
}

// ApplicationStartedEvent — context refreshed, before runners
public class ApplicationStartedEvent extends IrisApplicationEvent {
    private final ConfigurableApplicationContext context;
    private final Duration timeTaken;
    // constructor, getters...
}

// ApplicationReadyEvent — runners complete, app fully started
public class ApplicationReadyEvent extends IrisApplicationEvent {
    private final ConfigurableApplicationContext context;
    private final Duration timeTaken;
    // constructor, getters...
}

// ApplicationFailedEvent — startup failed
public class ApplicationFailedEvent extends IrisApplicationEvent {
    private final ConfigurableApplicationContext context; // may be null
    private final Throwable exception;
    // constructor, getters...
}
```

### The Run Listener SPI

**New file:** `iris-boot-core/.../boot/IrisApplicationRunListener.java`

```java
package com.iris.boot;

public interface IrisApplicationRunListener {
    default void starting() {}
    default void environmentPrepared(Environment environment) {}
    default void contextPrepared(ConfigurableApplicationContext context) {}
    default void started(ConfigurableApplicationContext context, Duration timeTaken) {}
    default void ready(ConfigurableApplicationContext context, Duration timeTaken) {}
    default void failed(ConfigurableApplicationContext context, Throwable exception) {}
}
```

### EventPublishingRunListener — The Bridge

This is the key class that connects the callback-based run listener SPI to the event-based `ApplicationListener` system. It uses **two-phase event publishing**:

**New file:** `iris-boot-core/.../boot/context/event/EventPublishingRunListener.java`

```java
package com.iris.boot.context.event;

public class EventPublishingRunListener implements IrisApplicationRunListener {
    private final IrisApplication application;
    private final String[] args;
    private final List<ApplicationListener<?>> initialListeners;

    public EventPublishingRunListener(IrisApplication application, String[] args,
                                       List<ApplicationListener<?>> listeners) {
        this.application = application;
        this.args = args;
        this.initialListeners = new ArrayList<>(listeners);
    }

    // Phase 1: Before context exists — multicast via initial listener list
    @Override
    public void starting() {
        multicastInitialEvent(new ApplicationStartingEvent(this.application, this.args));
    }

    @Override
    public void environmentPrepared(Environment environment) {
        multicastInitialEvent(
                new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        // Bridge: register listeners onto the context for Phase 2
        for (ApplicationListener<?> listener : this.initialListeners) {
            context.addApplicationListener(listener);
        }
    }

    // Phase 2: After context refresh — publish via context
    @Override
    public void started(ConfigurableApplicationContext context, Duration timeTaken) {
        context.publishEvent(
                new ApplicationStartedEvent(this.application, this.args, context, timeTaken));
    }

    @Override
    public void ready(ConfigurableApplicationContext context, Duration timeTaken) {
        context.publishEvent(
                new ApplicationReadyEvent(this.application, this.args, context, timeTaken));
    }

    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        ApplicationFailedEvent event =
                new ApplicationFailedEvent(this.application, this.args, context, exception);
        if (context != null && context.isActive()) {
            context.publishEvent(event);
        } else {
            multicastInitialEvent(event);
        }
    }

    private void multicastInitialEvent(ApplicationEvent event) {
        for (ApplicationListener<?> listener : this.initialListeners) {
            Class<?> eventType = resolveEventType(listener);
            if (eventType == null || eventType.isInstance(event)) {
                invokeListener(listener, event);
            }
        }
    }
}
```

---

## 8.4 IrisApplication — The Bootstrap Orchestrator

**New file:** `iris-boot-core/.../boot/IrisApplication.java`

```java
package com.iris.boot;

public class IrisApplication {

    private final Class<?> primarySource;
    private WebApplicationType webApplicationType;
    private final List<ApplicationListener<?>> listeners = new ArrayList<>();

    public IrisApplication(Class<?> primarySource) {
        this.primarySource = primarySource;
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
    }

    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return new IrisApplication(primarySource).run(args);
    }

    public ConfigurableApplicationContext run(String... args) {
        long startTime = System.nanoTime();

        // Step 1: Create run listeners
        IrisApplicationRunListeners listeners = getRunListeners(args);

        // Step 2: Fire starting event
        listeners.starting();

        ConfigurableApplicationContext context = null;
        try {
            // Step 3: Parse command-line arguments
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

            // Step 4: Prepare the environment
            Environment environment = prepareEnvironment(listeners);

            // Step 5: Print banner
            printBanner(environment);

            // Step 6: Create the ApplicationContext
            context = createApplicationContext();

            // Step 7: Prepare the context
            prepareContext(context, environment, applicationArguments, listeners);

            // Step 8: Refresh the context
            refreshContext(context);

            // Step 9: Fire started event
            Duration timeTaken = Duration.ofNanos(System.nanoTime() - startTime);
            listeners.started(context, timeTaken);
            logStartupInfo(timeTaken);

            // Step 10: Call runners
            callRunners(context, applicationArguments);

            // Step 11: Fire ready event
            Duration totalTime = Duration.ofNanos(System.nanoTime() - startTime);
            listeners.ready(context, totalTime);

            return context;

        } catch (Throwable ex) {
            handleRunFailure(context, listeners, ex);
            throw new IllegalStateException("Application startup failed", ex);
        }
    }
}
```

The key methods inside `IrisApplication`:

**`prepareEnvironment()`** — Creates a `StandardEnvironment`, loads `application.properties`, then fires `ApplicationEnvironmentPreparedEvent` so listeners can modify the environment before the context uses it.

**`prepareContext()`** — Sets the externally-prepared environment on the context, registers the primary source class and `ApplicationArguments` as beans, then fires the `contextPrepared` callback (which bridges listeners from the initial multicaster to the context).

**`callRunners()`** — After refresh, discovers all `ApplicationRunner` and `CommandLineRunner` beans and invokes them with the parsed arguments.

**`handleRunFailure()`** — Fires `ApplicationFailedEvent`, then closes the context if it was created.

---

## 8.5 Try It Yourself

<details><summary>Challenge 1: What happens if you use a lambda as a typed ApplicationListener?</summary>

Java lambdas erase generic type parameters at runtime. Our `resolveEventType()` method walks the class hierarchy looking for `ParameterizedType` information, but lambda classes don't expose this.

Result: `resolveEventType()` returns `null`, meaning the listener receives ALL events. When a `ContextRefreshedEvent` is passed to a lambda typed as `ApplicationListener<ApplicationStartingEvent>`, a `ClassCastException` occurs.

**Fix:** Use anonymous inner classes instead of lambdas when you need type filtering:
```java
// This works — anonymous inner class preserves type info
app.addListener(new ApplicationListener<ApplicationStartingEvent>() {
    @Override
    public void onApplicationEvent(ApplicationStartingEvent event) {
        System.out.println("Starting!");
    }
});

// This does NOT work — lambda erases type parameter
app.addListener((ApplicationListener<ApplicationStartingEvent>) event ->
        System.out.println("Starting!")); // ClassCastException when ContextRefreshedEvent arrives
```

In real Spring, `ResolvableType` handles this through a much more sophisticated type resolution mechanism. That's a simplification we accept in Iris.

</details>

<details><summary>Challenge 2: Implement a listener that logs the startup time</summary>

```java
IrisApplication app = new IrisApplication(MyConfig.class);
app.addListener(new ApplicationListener<ApplicationReadyEvent>() {
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.printf("Application ready in %d ms%n",
                event.getTimeTaken().toMillis());
    }
});
app.run(args);
```

</details>

<details><summary>Challenge 3: Why does environment preparation happen OUTSIDE the context?</summary>

In both real Spring Boot and Iris, the environment is prepared before the `ApplicationContext` is created. This allows:

1. **Early event listeners** to modify properties before any bean sees them (e.g., `ConfigDataEnvironmentPostProcessor` loads profile-specific config)
2. **Web type detection** to use properties (e.g., `spring.main.web-application-type=none` to override)
3. **Banner printing** to use property values
4. The context to receive a fully-prepared environment via `setEnvironment()`

If the environment were prepared inside the context, you'd have a chicken-and-egg problem: beans would be created before all property sources are loaded.

</details>

---

## 8.6 Tests

### Unit Tests

```java
// WebApplicationTypeTest
@Test
void shouldDeduceServlet_WhenTomcatOnClasspath() {
    WebApplicationType type = WebApplicationType.deduceFromClasspath();
    assertThat(type).isEqualTo(WebApplicationType.SERVLET);
}

// DefaultApplicationArgumentsTest
@Test
void shouldParseOptionArguments() {
    DefaultApplicationArguments args = new DefaultApplicationArguments(
            "--server.port=9090", "--app.name=MyApp");
    assertThat(args.getOptionNames()).containsExactlyInAnyOrder("server.port", "app.name");
}

@Test
void shouldReturnEmptyListForBooleanFlags() {
    DefaultApplicationArguments args = new DefaultApplicationArguments("--debug");
    assertThat(args.containsOption("debug")).isTrue();
    assertThat(args.getOptionValues("debug")).isEmpty();
}

// IrisApplicationTest
@Test
void shouldCallApplicationRunners_WhenContextRefreshed() {
    RunnerConfig.executionLog.clear();
    ConfigurableApplicationContext context =
            IrisApplication.run(RunnerConfig.class, "--profile=test");
    assertThat(RunnerConfig.executionLog).anySatisfy(log ->
            assertThat(log).startsWith("appRunner:").contains("--profile=test"));
    context.close();
}

@Test
void shouldMakeApplicationArgumentsAvailableAsBean() {
    ConfigurableApplicationContext context =
            IrisApplication.run(SimpleConfig.class, "--server.port=9090");
    ApplicationArguments args = context.getBean(ApplicationArguments.class);
    assertThat(args.containsOption("server.port")).isTrue();
    context.close();
}
```

### Integration Tests

```java
@Test
void shouldFireEventsInCorrectOrder() {
    List<Class<? extends ApplicationEvent>> eventOrder = new ArrayList<>();
    IrisApplication app = new IrisApplication(FullAppConfig.class);

    app.addListener(new ApplicationListener<ApplicationEvent>() {
        @Override
        public void onApplicationEvent(ApplicationEvent event) {
            eventOrder.add(event.getClass());
        }
    });

    context = app.run("--test");

    assertThat(eventOrder).containsSubsequence(
            ApplicationStartingEvent.class,
            ApplicationEnvironmentPreparedEvent.class,
            ContextRefreshedEvent.class,
            ApplicationStartedEvent.class,
            ApplicationReadyEvent.class
    );
}

@Test
void shouldInjectValueAnnotationsFromProperties() {
    context = IrisApplication.run(ValueInjectionConfig.class);
    PropertyHolder holder = context.getBean(PropertyHolder.class);
    assertThat(holder.appName).isEqualTo("Iris Boot Test");
    assertThat(holder.serverPort).isEqualTo(8080);
}
```

Run all tests:
```bash
./gradlew test
```

---

## 8.7 Why This Works

`★ Insight ─────────────────────────────────────`
**Two-Phase Event Publishing:** Before `refresh()`, no ApplicationContext exists — so the `EventPublishingRunListener` maintains its own list of listeners and multicasts events directly (the "initial multicaster"). After `refresh()`, the context has its own event infrastructure, so events flow through `context.publishEvent()`. During `contextPrepared()`, listeners are bridged from Phase 1 to Phase 2 by calling `context.addApplicationListener()`. This two-phase design is the key architectural insight of Spring Boot's event system — it allows listeners to participate in the full lifecycle, including the earliest moments before the container exists.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Environment Preparation Happens Outside the Context:** The environment is created, populated (system props + env vars + application.properties), and event-fired BEFORE the ApplicationContext even exists. `IrisApplication.prepareEnvironment()` builds the environment externally, then `prepareContext()` sets it on the context via `setEnvironment()`. This inversion of control — the boot layer prepares what the framework layer consumes — is what makes Spring Boot an "opinionated runtime." The framework provides the container machinery; Boot decides how to configure it.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**The No-Arg Constructor Pattern:** The original `AnnotationConfigApplicationContext(Class<?>...)` constructor calls `refresh()` immediately — it's a convenience for simple cases. But `IrisApplication` needs control between construction and refresh: set environment, register classes, add listeners, register singleton beans. The no-arg constructor enables this "configure-then-refresh" pattern. Java's method resolution ensures no ambiguity: `new Context()` calls the no-arg constructor, `new Context(AppConfig.class)` calls the varargs one.
`─────────────────────────────────────────────────`

---

## 8.8 What We Enhanced

| File | What Changed | Why |
|------|-------------|-----|
| `ConfigurableApplicationContext.java` | Added `setEnvironment()` and `addApplicationListener()` methods | Bootstrap layer needs to configure the context externally before refresh |
| `AnnotationConfigApplicationContext.java` | Added no-arg constructor, `register()` method, `setEnvironment()` impl; changed `environment` field from `final StandardEnvironment` to mutable `Environment` | Enable the "create → configure → refresh" pattern that `IrisApplication` uses |

---

## 8.9 Connection to Real Framework

| Iris Simplified | Real Spring Boot | File:Line (commit `5922311a95a`) |
|----------------|-----------------|--------------------------------|
| `IrisApplication` | `SpringApplication` | `SpringApplication.java:304` (`run()` method) |
| `IrisApplicationRunListener` | `SpringApplicationRunListener` | `SpringApplicationRunListener.java:40` |
| `EventPublishingRunListener` | `EventPublishingRunListener` | `EventPublishingRunListener.java:47` |
| `IrisApplicationEvent` | `SpringApplicationEvent` | `SpringApplicationEvent.java:30` |
| `WebApplicationType` | `WebApplicationType` | `WebApplicationType.java:33` |
| `ApplicationRunner` | `ApplicationRunner` | `ApplicationRunner.java:36` |
| `CommandLineRunner` | `CommandLineRunner` | `CommandLineRunner.java:36` |
| `ApplicationArguments` | `ApplicationArguments` | `ApplicationArguments.java:31` |
| `IrisApplicationRunListeners` | `SpringApplicationRunListeners` | `SpringApplicationRunListeners.java:38` |

**Key differences from the real implementation:**
- No `BootstrapContext` for very-early-startup service registration
- No `SpringFactoriesLoader` SPI — listeners are wired directly
- No `ApplicationContextInitializer` support
- No `ApplicationStartup` instrumentation for startup tracing
- No `AvailabilityChangeEvent` for Kubernetes liveness/readiness
- No shutdown hook registration (Feature 19)
- No `@Order` support for runner execution order
- Banner is a static string (no `banner.txt` or custom `Banner` interface)

---

## 8.10 Complete Code

### Production Code

#### [MODIFIED] `iris-framework/.../context/ConfigurableApplicationContext.java`

```java
package com.iris.framework.context;

import java.io.Closeable;

import com.iris.framework.core.env.Environment;

public interface ConfigurableApplicationContext extends ApplicationContext, Closeable {

    void refresh();

    @Override
    void close();

    boolean isActive();

    void setEnvironment(Environment environment);

    void addApplicationListener(ApplicationListener<?> listener);
}
```

#### [MODIFIED] `iris-framework/.../context/AnnotationConfigApplicationContext.java`

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

    // ... refresh steps unchanged from ch06 ...

    @Override
    public void close() {
        if (this.closed.compareAndSet(false, true)) {
            publishEvent(new ContextClosedEvent(this));
            this.beanFactory.close();
            this.active.set(false);
            this.applicationListeners.clear();
        }
    }

    @Override public boolean isActive() { return this.active.get(); }

    @Override
    public void publishEvent(ApplicationEvent event) {
        for (ApplicationListener<?> listener : this.applicationListeners) {
            Class<?> eventType = resolveEventType(listener);
            if (eventType == null || eventType.isInstance(event)) {
                invokeListener(listener, event);
            }
        }
    }

    // ... bean factory delegation, getEnvironment(), etc. unchanged ...

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void addApplicationListener(ApplicationListener<?> listener) {
        this.applicationListeners.add(listener);
    }

    public DefaultBeanFactory getBeanFactory() { return this.beanFactory; }
}
```

#### [NEW] `iris-boot-core/.../boot/WebApplicationType.java`

```java
package com.iris.boot;

public enum WebApplicationType {
    NONE,
    SERVLET;

    private static final String TOMCAT_INDICATOR_CLASS = "org.apache.catalina.startup.Tomcat";
    private static final String SERVLET_INDICATOR_CLASS = "jakarta.servlet.Servlet";

    public static WebApplicationType deduceFromClasspath() {
        if (isPresent(TOMCAT_INDICATOR_CLASS) && isPresent(SERVLET_INDICATOR_CLASS)) {
            return SERVLET;
        }
        return NONE;
    }

    private static boolean isPresent(String className) {
        try {
            Class.forName(className, false, WebApplicationType.class.getClassLoader());
            return true;
        } catch (ClassNotFoundException ex) {
            return false;
        }
    }
}
```

#### [NEW] `iris-boot-core/.../boot/ApplicationArguments.java`

```java
package com.iris.boot;

import java.util.List;
import java.util.Set;

public interface ApplicationArguments {
    String[] getSourceArgs();
    Set<String> getOptionNames();
    boolean containsOption(String name);
    List<String> getOptionValues(String name);
    List<String> getNonOptionArgs();
}
```

#### [NEW] `iris-boot-core/.../boot/DefaultApplicationArguments.java`

```java
package com.iris.boot;

import java.util.*;

public class DefaultApplicationArguments implements ApplicationArguments {
    private final String[] sourceArgs;
    private final Map<String, List<String>> optionArgs = new LinkedHashMap<>();
    private final List<String> nonOptionArgs = new ArrayList<>();

    public DefaultApplicationArguments(String... args) {
        this.sourceArgs = args.clone();
        parse(args);
    }

    private void parse(String[] args) {
        for (String arg : args) {
            if (arg.startsWith("--")) {
                String withoutDashes = arg.substring(2);
                int equalsIndex = withoutDashes.indexOf('=');
                if (equalsIndex >= 0) {
                    String name = withoutDashes.substring(0, equalsIndex);
                    String value = withoutDashes.substring(equalsIndex + 1);
                    this.optionArgs.computeIfAbsent(name, k -> new ArrayList<>()).add(value);
                } else {
                    this.optionArgs.computeIfAbsent(withoutDashes, k -> new ArrayList<>());
                }
            } else {
                this.nonOptionArgs.add(arg);
            }
        }
    }

    @Override public String[] getSourceArgs() { return this.sourceArgs.clone(); }
    @Override public Set<String> getOptionNames() {
        return Collections.unmodifiableSet(new LinkedHashSet<>(this.optionArgs.keySet()));
    }
    @Override public boolean containsOption(String name) { return this.optionArgs.containsKey(name); }
    @Override public List<String> getOptionValues(String name) {
        List<String> values = this.optionArgs.get(name);
        return (values != null) ? Collections.unmodifiableList(values) : null;
    }
    @Override public List<String> getNonOptionArgs() {
        return Collections.unmodifiableList(this.nonOptionArgs);
    }
}
```

#### [NEW] `iris-boot-core/.../boot/ApplicationRunner.java`

```java
package com.iris.boot;

@FunctionalInterface
public interface ApplicationRunner {
    void run(ApplicationArguments args) throws Exception;
}
```

#### [NEW] `iris-boot-core/.../boot/CommandLineRunner.java`

```java
package com.iris.boot;

@FunctionalInterface
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

#### [NEW] `iris-boot-core/.../boot/IrisApplicationRunListener.java`

```java
package com.iris.boot;

import java.time.Duration;
import com.iris.framework.context.ConfigurableApplicationContext;
import com.iris.framework.core.env.Environment;

public interface IrisApplicationRunListener {
    default void starting() {}
    default void environmentPrepared(Environment environment) {}
    default void contextPrepared(ConfigurableApplicationContext context) {}
    default void started(ConfigurableApplicationContext context, Duration timeTaken) {}
    default void ready(ConfigurableApplicationContext context, Duration timeTaken) {}
    default void failed(ConfigurableApplicationContext context, Throwable exception) {}
}
```

#### [NEW] `iris-boot-core/.../boot/IrisApplicationRunListeners.java`

```java
package com.iris.boot;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import com.iris.framework.context.ConfigurableApplicationContext;
import com.iris.framework.core.env.Environment;

class IrisApplicationRunListeners {
    private final List<IrisApplicationRunListener> listeners;

    IrisApplicationRunListeners(List<IrisApplicationRunListener> listeners) {
        this.listeners = new ArrayList<>(listeners);
    }

    void starting() { for (var l : this.listeners) l.starting(); }
    void environmentPrepared(Environment env) { for (var l : this.listeners) l.environmentPrepared(env); }
    void contextPrepared(ConfigurableApplicationContext ctx) { for (var l : this.listeners) l.contextPrepared(ctx); }
    void started(ConfigurableApplicationContext ctx, Duration t) { for (var l : this.listeners) l.started(ctx, t); }
    void ready(ConfigurableApplicationContext ctx, Duration t) { for (var l : this.listeners) l.ready(ctx, t); }
    void failed(ConfigurableApplicationContext ctx, Throwable ex) { for (var l : this.listeners) l.failed(ctx, ex); }
}
```

#### [NEW] `iris-boot-core/.../boot/context/event/IrisApplicationEvent.java`

```java
package com.iris.boot.context.event;

import com.iris.framework.context.ApplicationEvent;
import com.iris.boot.IrisApplication;

public abstract class IrisApplicationEvent extends ApplicationEvent {
    private final String[] args;

    protected IrisApplicationEvent(IrisApplication application, String[] args) {
        super(application);
        this.args = args;
    }

    public IrisApplication getIrisApplication() { return (IrisApplication) getSource(); }
    public String[] getArgs() { return this.args; }
}
```

#### [NEW] Event classes (5 files)

`ApplicationStartingEvent.java`, `ApplicationEnvironmentPreparedEvent.java`, `ApplicationStartedEvent.java`, `ApplicationReadyEvent.java`, `ApplicationFailedEvent.java` — see Section 8.3 for implementations.

#### [NEW] `iris-boot-core/.../boot/context/event/EventPublishingRunListener.java`

See Section 8.3 for the full implementation. The key feature is two-phase publishing: initial multicaster for pre-context events, `context.publishEvent()` for post-refresh events.

#### [NEW] `iris-boot-core/.../boot/IrisApplication.java`

See Section 8.4 for the full implementation. The 11-step `run()` method is the core of this feature.

### Test Code

- `WebApplicationTypeTest.java` — 3 tests: classpath detection, enum values
- `DefaultApplicationArgumentsTest.java` — 9 tests: option parsing, boolean flags, non-option args, immutability
- `EventPublishingRunListenerTest.java` — 8 tests: two-phase publishing, type filtering, failure handling
- `IrisApplicationTest.java` — 10 tests: context creation, runners, events, properties, error handling
- `IrisApplicationBootstrapIntegrationTest.java` — 9 tests: full lifecycle, @Value injection, event ordering, runners with dependencies

---

## Summary

| What | Count |
|------|-------|
| New files (iris-boot-core) | 15 production, 5 test |
| Modified files (iris-framework) | 2 |
| New tests | 39 |
| All tests passing | 249 (210 framework + 39 boot) |

**The bootstrap sequence:**
```
IrisApplication.run(MyApp.class, args)
  → starting event
  → prepare environment (system props + application.properties)
  → environment prepared event
  → print banner
  → create AnnotationConfigApplicationContext (no-arg, deferred refresh)
  → set environment, register primary source, register ApplicationArguments bean
  → context prepared (bridge listeners to context)
  → context.refresh() (framework's 5-step pipeline)
  → started event
  → call ApplicationRunner/CommandLineRunner beans
  → ready event
  → return context
```

**Next chapter:** [Chapter 9: Embedded Web Server](ch09_embedded_web_server.md) — Run Tomcat inside your application with a `WebServer` abstraction and `TomcatWebServerFactory` that starts an embedded Tomcat on `context.refresh()`.
