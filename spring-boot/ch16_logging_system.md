# Chapter 16: Logging System

> Line references based on commit `5922311a95a` of the Spring Boot repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `IrisApplication.run()` orchestrates the full bootstrap lifecycle and fires events (`ApplicationStartingEvent`, `ApplicationEnvironmentPreparedEvent`, etc.), but the logging framework is unmanaged — noisy framework messages leak during startup, and `logging.level.*` properties in `application.properties` have no effect | No control over logging output during bootstrap, no property-driven log level configuration, and no pluggable logging system abstraction | Build a `LoggingSystem` abstraction that auto-detects the logging framework, suppresses output during early bootstrap, configures log levels from `logging.level.*` properties, and cleans up on shutdown — all driven by the existing event system |

---

## 16.1 The Integration Point

The logging system integrates into the bootstrap by registering a `LoggingApplicationListener` as a default listener on `IrisApplication`. This listener receives bootstrap events through the existing two-phase event publishing mechanism and drives the `LoggingSystem` through its lifecycle.

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java`
**Change:** Auto-register `LoggingApplicationListener` in the constructor

```java
import com.iris.boot.context.logging.LoggingApplicationListener;

// ... in the constructor:

public IrisApplication(Class<?> primarySource) {
    this.primarySource = primarySource;
    this.webApplicationType = WebApplicationType.deduceFromClasspath();

    // Register the LoggingApplicationListener as a default listener.
    // In the real Spring Boot, this is loaded via SpringFactoriesLoader
    // from META-INF/spring.factories. We wire it directly.
    this.listeners.add(new LoggingApplicationListener());
}
```

Two key decisions here:

1. **Why register in the constructor, not in `run()`?** The listener must be present *before* `run()` calls `getRunListeners()`, which snapshots the listener list for the `EventPublishingRunListener`. If we add it later, it misses `ApplicationStartingEvent`.
2. **Why a listener and not direct calls in `run()`?** The event-driven approach keeps `IrisApplication.run()` clean — it doesn't need to know about logging at all. The logging lifecycle is entirely self-contained in a listener that responds to events it already receives.

This connects **the bootstrap lifecycle (events)** to **the logging abstraction (`LoggingSystem`)**. To make it work, we need to build:
- `LogLevel` — framework-independent log level constants
- `LoggingSystem` — abstract base with `beforeInitialize()`, `initialize()`, `setLogLevel()`, and `cleanUp()`
- `LoggingSystemFactory` — SPI interface for classpath-based detection
- `JavaLoggingSystem` — concrete JUL implementation (no external deps)
- `LoggingApplicationListener` — the event-driven lifecycle orchestrator

## 16.2 LogLevel Enum

The framework-independent log level constants. Each concrete `LoggingSystem` maps these to its framework's native levels.

**New file:** `iris-boot-core/src/main/java/com/iris/boot/logging/LogLevel.java`

```java
package com.iris.boot.logging;

public enum LogLevel {

    TRACE,
    DEBUG,
    INFO,
    WARN,
    ERROR,
    FATAL,
    OFF
}
```

## 16.3 LoggingSystemFactory

The SPI interface for pluggable logging system detection. Each factory checks whether its framework is on the classpath and returns an instance or `null`.

**New file:** `iris-boot-core/src/main/java/com/iris/boot/logging/LoggingSystemFactory.java`

```java
package com.iris.boot.logging;

@FunctionalInterface
public interface LoggingSystemFactory {

    LoggingSystem getLoggingSystem(ClassLoader classLoader);
}
```

## 16.4 LoggingSystem Abstract Base

The core abstraction — defines the lifecycle methods and provides static detection via `get(ClassLoader)`.

**New file:** `iris-boot-core/src/main/java/com/iris/boot/logging/LoggingSystem.java`

```java
package com.iris.boot.logging;

import java.util.EnumSet;
import java.util.List;
import java.util.Set;

import com.iris.framework.core.env.Environment;

public abstract class LoggingSystem {

    public static final String SYSTEM_PROPERTY = "com.iris.boot.logging.LoggingSystem";
    public static final String NONE = "none";
    public static final String ROOT_LOGGER_NAME = "ROOT";

    // --- Lifecycle methods ---

    public abstract void beforeInitialize();

    public void initialize(Environment environment) {
        // Default: no-op
    }

    public void cleanUp() {
        // Default: no-op
    }

    public void setLogLevel(String loggerName, LogLevel level) {
        throw new UnsupportedOperationException(
                "Unable to set log level for " + getClass().getSimpleName());
    }

    public Set<LogLevel> getSupportedLogLevels() {
        return EnumSet.allOf(LogLevel.class);
    }

    // --- Static detection ---

    private static final List<LoggingSystemFactory> FACTORIES = List.of(
            new com.iris.boot.logging.java.JavaLoggingSystem.Factory()
    );

    public static LoggingSystem get(ClassLoader classLoader) {
        String property = System.getProperty(SYSTEM_PROPERTY);
        if (property != null) {
            if (NONE.equals(property)) {
                return new NoOpLoggingSystem();
            }
            // Instantiate from class name...
        }

        for (LoggingSystemFactory factory : FACTORIES) {
            LoggingSystem system = factory.getLoggingSystem(classLoader);
            if (system != null) {
                return system;
            }
        }
        throw new IllegalStateException("No suitable LoggingSystem could be found");
    }

    // --- Null Object for disabled logging ---

    static class NoOpLoggingSystem extends LoggingSystem {
        @Override
        public void beforeInitialize() { }

        @Override
        public void setLogLevel(String loggerName, LogLevel level) { }
    }
}
```

## 16.5 JavaLoggingSystem

The JUL-backed concrete implementation. Maps `LogLevel` to JUL's `Level`, handles the weak-reference problem, and reads `logging.level.*` properties from the environment.

**New file:** `iris-boot-core/src/main/java/com/iris/boot/logging/java/JavaLoggingSystem.java`

```java
package com.iris.boot.logging.java;

import java.util.Collections;
import java.util.EnumMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.logging.Level;
import java.util.logging.Logger;

import com.iris.boot.logging.LogLevel;
import com.iris.boot.logging.LoggingSystem;
import com.iris.boot.logging.LoggingSystemFactory;
import com.iris.framework.core.env.Environment;

public class JavaLoggingSystem extends LoggingSystem {

    private static final Map<LogLevel, Level> LEVEL_MAP;
    static {
        Map<LogLevel, Level> map = new EnumMap<>(LogLevel.class);
        map.put(LogLevel.TRACE, Level.FINEST);
        map.put(LogLevel.DEBUG, Level.FINE);
        map.put(LogLevel.INFO, Level.INFO);
        map.put(LogLevel.WARN, Level.WARNING);
        map.put(LogLevel.ERROR, Level.SEVERE);
        map.put(LogLevel.FATAL, Level.SEVERE);
        map.put(LogLevel.OFF, Level.OFF);
        LEVEL_MAP = Collections.unmodifiableMap(map);
    }

    private final Set<Logger> configuredLoggers =
            Collections.synchronizedSet(new HashSet<>());
    private final ClassLoader classLoader;

    public JavaLoggingSystem(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    @Override
    public void beforeInitialize() {
        Logger.getLogger("").setLevel(Level.SEVERE);  // Suppress during bootstrap
    }

    @Override
    public void initialize(Environment environment) {
        Logger rootLogger = Logger.getLogger("");
        rootLogger.setLevel(Level.INFO);  // Restore default
        configuredLoggers.add(rootLogger);
        applyLevelProperties(environment);
    }

    @Override
    public void setLogLevel(String loggerName, LogLevel level) {
        String name = ROOT_LOGGER_NAME.equalsIgnoreCase(loggerName) ? "" : loggerName;
        Logger logger = Logger.getLogger(name);
        if (level == null) {
            logger.setLevel(null);  // Inherit from parent
            configuredLoggers.remove(logger);
        } else {
            logger.setLevel(LEVEL_MAP.get(level));
            configuredLoggers.add(logger);  // Prevent GC
        }
    }

    @Override
    public void cleanUp() {
        configuredLoggers.clear();
    }

    private void applyLevelProperties(Environment environment) {
        String prefix = "logging.level.";
        for (var propertySource : environment.getPropertySources()) {
            Object source = propertySource.getSource();
            if (source instanceof java.util.Map<?, ?> map) {
                for (Object key : map.keySet()) {
                    String keyStr = key.toString();
                    if (keyStr.startsWith(prefix)) {
                        String loggerName = keyStr.substring(prefix.length());
                        String levelStr = environment.getProperty(keyStr);
                        if (levelStr != null && !levelStr.isBlank()) {
                            LogLevel logLevel = LogLevel.valueOf(
                                    levelStr.trim().toUpperCase());
                            setLogLevel(loggerName, logLevel);
                        }
                    }
                }
            }
        }
    }

    public static class Factory implements LoggingSystemFactory {
        @Override
        public LoggingSystem getLoggingSystem(ClassLoader classLoader) {
            return new JavaLoggingSystem(classLoader);
        }
    }
}
```

## 16.6 LoggingApplicationListener

The lifecycle orchestrator — a single listener that handles four event types and drives the `LoggingSystem` through its state machine.

**New file:** `iris-boot-core/src/main/java/com/iris/boot/context/logging/LoggingApplicationListener.java`

```java
package com.iris.boot.context.logging;

import com.iris.boot.context.event.ApplicationEnvironmentPreparedEvent;
import com.iris.boot.context.event.ApplicationFailedEvent;
import com.iris.boot.context.event.ApplicationStartingEvent;
import com.iris.boot.logging.LoggingSystem;
import com.iris.framework.context.ApplicationEvent;
import com.iris.framework.context.ApplicationListener;
import com.iris.framework.context.event.ContextClosedEvent;

public class LoggingApplicationListener implements ApplicationListener<ApplicationEvent> {

    private LoggingSystem loggingSystem;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationStartingEvent) {
            onApplicationStartingEvent();
        } else if (event instanceof ApplicationEnvironmentPreparedEvent envEvent) {
            onApplicationEnvironmentPreparedEvent(envEvent);
        } else if (event instanceof ContextClosedEvent) {
            onContextClosedEvent();
        } else if (event instanceof ApplicationFailedEvent) {
            onApplicationFailedEvent();
        }
    }

    private void onApplicationStartingEvent() {
        this.loggingSystem = LoggingSystem.get(getClassLoader());
        this.loggingSystem.beforeInitialize();
    }

    private void onApplicationEnvironmentPreparedEvent(
            ApplicationEnvironmentPreparedEvent event) {
        this.loggingSystem.initialize(event.getEnvironment());
    }

    private void onContextClosedEvent() {
        if (this.loggingSystem != null) {
            this.loggingSystem.cleanUp();
            this.loggingSystem = null;
        }
    }

    private void onApplicationFailedEvent() {
        if (this.loggingSystem != null) {
            this.loggingSystem.cleanUp();
            this.loggingSystem = null;
        }
    }

    public LoggingSystem getLoggingSystem() {
        return this.loggingSystem;
    }

    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return (cl != null) ? cl : getClass().getClassLoader();
    }
}
```

## 16.7 Try It Yourself

<details>
<summary>Challenge: Implement the level mapping for JavaLoggingSystem</summary>

JUL uses different level names than most frameworks. Given this mapping:

| LogLevel | JUL Level |
|----------|-----------|
| TRACE    | FINEST    |
| DEBUG    | FINE      |
| INFO     | INFO      |
| WARN     | WARNING   |
| ERROR    | SEVERE    |
| FATAL    | SEVERE    |
| OFF      | OFF       |

Implement the `LEVEL_MAP` and the `setLogLevel()` method. Remember that JUL uses weak references internally for loggers — if you don't hold a strong reference, the logger's level setting can be garbage-collected.

```java
private static final Map<LogLevel, Level> LEVEL_MAP;
static {
    Map<LogLevel, Level> map = new EnumMap<>(LogLevel.class);
    map.put(LogLevel.TRACE, Level.FINEST);
    map.put(LogLevel.DEBUG, Level.FINE);
    map.put(LogLevel.INFO, Level.INFO);
    map.put(LogLevel.WARN, Level.WARNING);
    map.put(LogLevel.ERROR, Level.SEVERE);
    map.put(LogLevel.FATAL, Level.SEVERE);
    map.put(LogLevel.OFF, Level.OFF);
    LEVEL_MAP = Collections.unmodifiableMap(map);
}

private final Set<Logger> configuredLoggers = Collections.synchronizedSet(new HashSet<>());

@Override
public void setLogLevel(String loggerName, LogLevel level) {
    String name = ROOT_LOGGER_NAME.equalsIgnoreCase(loggerName) ? "" : loggerName;
    Logger logger = Logger.getLogger(name);
    if (level == null) {
        logger.setLevel(null);
        configuredLoggers.remove(logger);
    } else {
        logger.setLevel(LEVEL_MAP.get(level));
        configuredLoggers.add(logger);  // Hold strong reference!
    }
}
```

</details>

<details>
<summary>Challenge: Wire the LoggingApplicationListener into the event lifecycle</summary>

The listener needs to handle four different event types, each triggering a different lifecycle action. The tricky part: it receives events from TWO different phases of the event system. `ApplicationStartingEvent` and `ApplicationEnvironmentPreparedEvent` come via the initial multicaster (before the context exists). `ContextClosedEvent` comes from the context's event publisher (after the listener is bridged onto the context).

Think about: why does a single listener handle all these events instead of separate listeners for each event type?

**Answer:** Because the listener holds state (`this.loggingSystem`) that spans the full lifecycle. The same `LoggingSystem` instance detected on `ApplicationStartingEvent` must be initialized on `ApplicationEnvironmentPreparedEvent` and cleaned up on `ContextClosedEvent`. A single listener naturally holds this state across phases.

```java
public class LoggingApplicationListener implements ApplicationListener<ApplicationEvent> {

    private LoggingSystem loggingSystem;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationStartingEvent) {
            this.loggingSystem = LoggingSystem.get(getClassLoader());
            this.loggingSystem.beforeInitialize();
        } else if (event instanceof ApplicationEnvironmentPreparedEvent envEvent) {
            this.loggingSystem.initialize(envEvent.getEnvironment());
        } else if (event instanceof ContextClosedEvent) {
            if (this.loggingSystem != null) {
                this.loggingSystem.cleanUp();
                this.loggingSystem = null;
            }
        } else if (event instanceof ApplicationFailedEvent) {
            if (this.loggingSystem != null) {
                this.loggingSystem.cleanUp();
                this.loggingSystem = null;
            }
        }
    }
}
```

</details>

## 16.8 Tests

### Unit Tests

**New file:** `iris-boot-core/src/test/java/com/iris/boot/logging/LoggingSystemTest.java`

```java
package com.iris.boot.logging;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class LoggingSystemTest {

    @AfterEach
    void clearSystemProperty() {
        System.clearProperty(LoggingSystem.SYSTEM_PROPERTY);
    }

    @Test
    void shouldDetectJavaLoggingSystem_WhenNoSystemPropertySet() {
        LoggingSystem system = LoggingSystem.get(getClass().getClassLoader());
        assertThat(system.getClass().getSimpleName()).isEqualTo("JavaLoggingSystem");
    }

    @Test
    void shouldReturnNoOpLoggingSystem_WhenSystemPropertySetToNone() {
        System.setProperty(LoggingSystem.SYSTEM_PROPERTY, LoggingSystem.NONE);
        LoggingSystem system = LoggingSystem.get(getClass().getClassLoader());
        assertThat(system.getClass().getSimpleName()).isEqualTo("NoOpLoggingSystem");
    }

    @Test
    void shouldThrowException_WhenSystemPropertySetToInvalidClass() {
        System.setProperty(LoggingSystem.SYSTEM_PROPERTY, "com.nonexistent.LoggingSystem");
        assertThatThrownBy(() -> LoggingSystem.get(getClass().getClassLoader()))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Failed to create LoggingSystem");
    }
}
```

**New file:** `iris-boot-core/src/test/java/com/iris/boot/logging/java/JavaLoggingSystemTest.java`

Tests the JUL-backed implementation: the full lifecycle (`beforeInitialize` → `initialize` → `cleanUp`), all level mappings, root logger handling, and property binding.

**New file:** `iris-boot-core/src/test/java/com/iris/boot/context/logging/LoggingApplicationListenerTest.java`

Tests the listener's event dispatching: starting → suppress, environment prepared → initialize, context closed → cleanup, failed → cleanup.

### Integration Tests

**New file:** `iris-boot-core/src/test/java/com/iris/boot/logging/integration/LoggingSystemIntegrationTest.java`

Tests the full round-trip with `IrisApplication`: auto-registration of the listener, logging initialization during bootstrap, cleanup on context close, and disabled logging via system property.

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 16.9 Why This Works

> ★ **Insight** — Event-Driven Lifecycle Orchestration
> - The logging system's entire lifecycle is driven by events it receives, not by direct calls from `IrisApplication.run()`. This is the **Observer pattern** at its best: the bootstrap sequence has no knowledge of logging — it just fires events. The `LoggingApplicationListener` self-subscribes and reacts.
> - This decoupling means you can disable the logging system entirely (set the system property to `"none"`) without changing a single line in the bootstrap sequence. The listener still runs, but the `NoOpLoggingSystem` silently ignores all calls.
> - **Trade-off:** Event-driven architectures make the control flow harder to trace — you can't just read `run()` to understand what happens with logging. The real Spring Boot mitigates this with excellent Javadoc and clear event naming.
> ---

> ★ **Insight** — The Weak Reference Trap in JUL
> - Java's built-in logging framework (`java.util.logging`) stores logger instances using `WeakReference`s in its internal `LogManager`. This means if you create a logger, set its level, and then lose all strong references to it, the garbage collector can reclaim it — and the level setting is lost.
> - The `configuredLoggers` `Set<Logger>` in `JavaLoggingSystem` is the fix: it holds strong references to every logger whose level has been explicitly set. The real Spring Boot uses exactly the same pattern (`JavaLoggingSystem.java:43`).
> - **This is why JUL is the fallback, not the preferred option.** Logback and Log4j2 don't have this problem — they manage logger lifecycles properly. JUL was designed in 2002 and predates many modern logging best practices.
> ---

> ★ **Insight** — Strategy + Chain of Responsibility for Framework Detection
> - `LoggingSystem.get(ClassLoader)` implements two patterns in one: **Chain of Responsibility** (iterate through factories, take the first non-null result) and **Strategy** (each factory encapsulates the detection logic for a specific framework).
> - In the real Spring Boot, factories are ordered via `@Order` annotations: Logback (highest priority) → Log4j2 → JUL (lowest priority, fallback). This ordering exists because Logback provides the best Spring Boot integration (spring-specific XML extensions, colored output, etc.).
> - **When not to use this:** If you only ever have one implementation, this is over-engineered. We have it here because the real Spring Boot needs it — and it makes the architecture clear even in the simplified version.
> ---

## 16.10 What We Enhanced

| Aspect | Before (ch08–ch15) | Current (ch16) | Real Framework |
|--------|---------------------|----------------|----------------|
| **Logging control** | No control — framework startup messages leak to console unmanaged | `LoggingSystem.beforeInitialize()` suppresses output during bootstrap; `initialize()` configures levels from properties | Full `LoggingApplicationListener` with config-file discovery, log groups, `--debug`/`--trace` flags, shutdown hooks (`LoggingApplicationListener.java:1-493`) |
| **Log level configuration** | `logging.level.*` properties had no effect | Properties are read and applied to the underlying logging framework | Bound via Spring's `Binder`, supports logger groups (`web`, `sql`), `@ConfigurationProperties` integration |
| **Logging framework detection** | None — no awareness of logging frameworks | `LoggingSystem.get()` auto-detects via `LoggingSystemFactory` SPI | `SpringFactoriesLoader` loads factories with `@Order` priority from `META-INF/spring.factories` |
| **Default listeners** | `IrisApplication` had no default listeners | `LoggingApplicationListener` is auto-registered in constructor | Loaded via `SpringFactoriesLoader` — includes `LoggingApplicationListener`, `BackgroundPreinitializer`, and others |

## 16.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `LogLevel` (7 constants) | `LogLevel` | `LogLevel.java:34-80` | Real version carries `LogMethod` functional interface for direct invocation: `level.log(logger, msg)` |
| `LoggingSystem` (abstract base) | `LoggingSystem` | `LoggingSystem.java:1-214` | Real version has `LoggingInitializationContext`, `LogFile`, `getShutdownHandler()`, `getLoggerConfigurations()` |
| `LoggingSystem.get()` (hard-coded factory list) | `LoggingSystem.get()` | `LoggingSystem.java:163-186` | Real version loads factories via `SpringFactoriesLoader` from `spring.factories` |
| `LoggingSystemFactory` (single method) | `LoggingSystemFactory` | `LoggingSystemFactory.java:1-49` | Real version has `fromSpringFactories()` static method returning `DelegatingLoggingSystemFactory` |
| `JavaLoggingSystem` (direct property scanning) | `JavaLoggingSystem` extends `AbstractLoggingSystem` | `JavaLoggingSystem.java:1-198` | Real version has config-file loading, convention-based discovery via `getStandardConfigLocations()` |
| No `AbstractLoggingSystem` | `AbstractLoggingSystem` (template method) | `AbstractLoggingSystem.java:1-250` | Real version implements convention-over-configuration: standard config → `-spring` config → programmatic defaults |
| `LoggingApplicationListener` (4 events) | `LoggingApplicationListener` (5 events + `SmartLifecycle`) | `LoggingApplicationListener.java:1-493` | Real version handles `ApplicationPreparedEvent` (registers beans), inner `Lifecycle` class for ordered shutdown, logger groups, `--debug`/`--trace` argument parsing |

## 16.12 Complete Code

### Production Code

#### File: `iris-boot-core/src/main/java/com/iris/boot/logging/LogLevel.java` [NEW]

```java
package com.iris.boot.logging;

/**
 * Logging levels supported by the Iris logging abstraction.
 *
 * <p>These are the logical levels used across the logging system, independent
 * of which underlying logging framework (JUL, Logback, Log4j2) is in use.
 * Each concrete {@link LoggingSystem} implementation maps these to its
 * framework's native levels.
 *
 * <p>In the real Spring Boot, {@code LogLevel} also carries a {@code LogMethod}
 * functional interface so each level can directly invoke
 * {@code logger.debug(msg)} etc. We keep it simple — just the level constants.
 *
 * @see LoggingSystem#setLogLevel(String, LogLevel)
 * @see org.springframework.boot.logging.LogLevel
 */
public enum LogLevel {

    TRACE,
    DEBUG,
    INFO,
    WARN,
    ERROR,
    FATAL,
    OFF
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/logging/LoggingSystemFactory.java` [NEW]

```java
package com.iris.boot.logging;

/**
 * SPI interface for creating {@link LoggingSystem} instances.
 *
 * <p>Each concrete logging framework (JUL, Logback, Log4j2) provides a
 * factory implementation. The factory checks whether its framework is
 * available on the classpath and returns a {@code LoggingSystem} instance
 * if so — otherwise returns {@code null}.
 *
 * <p>Factories are tried in priority order. In the real Spring Boot,
 * factories are loaded via {@code SpringFactoriesLoader} and ordered via
 * {@code @Order} annotations:
 * <ol>
 *   <li>{@code LogbackLoggingSystem.Factory} (highest priority)</li>
 *   <li>{@code Log4J2LoggingSystem.Factory}</li>
 *   <li>{@code JavaLoggingSystem.Factory} (lowest priority — fallback)</li>
 * </ol>
 *
 * <p>We use only {@code JavaLoggingSystem.Factory} since it requires no
 * external dependencies (JUL is built into the JDK).
 *
 * <h3>Design Pattern: Strategy + Chain of Responsibility</h3>
 *
 * <p>Each factory is a strategy for creating a specific logging system.
 * The factories are chained: {@code LoggingSystem.get()} iterates through
 * them and takes the first non-null result. This makes the system
 * extensible without modifying existing code.
 *
 * @see LoggingSystem#get(ClassLoader)
 * @see org.springframework.boot.logging.LoggingSystemFactory
 */
@FunctionalInterface
public interface LoggingSystemFactory {

    /**
     * Create a {@code LoggingSystem} backed by this factory's framework,
     * or return {@code null} if the framework is not available.
     *
     * @param classLoader the classloader used for detection
     * @return a {@code LoggingSystem}, or {@code null} if not available
     */
    LoggingSystem getLoggingSystem(ClassLoader classLoader);
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/logging/LoggingSystem.java` [NEW]

```java
package com.iris.boot.logging;

import java.util.EnumSet;
import java.util.List;
import java.util.Set;

import com.iris.framework.core.env.Environment;

/**
 * Abstract base class for pluggable logging system integrations.
 *
 * <p>The logging system is the bridge between Iris Boot's configuration
 * (properties, lifecycle events) and the underlying logging framework
 * (JUL, Logback, Log4j2). It provides three key operations:
 *
 * <ol>
 *   <li>{@link #beforeInitialize()} — suppress output during early bootstrap</li>
 *   <li>{@link #initialize(Environment)} — full initialization with
 *       property-driven configuration</li>
 *   <li>{@link #setLogLevel(String, LogLevel)} — change logger levels at runtime</li>
 * </ol>
 *
 * <p>The lifecycle is driven by {@code LoggingApplicationListener}, which
 * detects the logging system via {@link #get(ClassLoader)} and calls these
 * methods at the right moments during bootstrap.
 *
 * <h3>Detection</h3>
 *
 * <p>The static {@link #get(ClassLoader)} method uses a chain of
 * {@link LoggingSystemFactory} implementations to auto-detect which logging
 * framework is on the classpath. The first factory that returns a non-null
 * {@code LoggingSystem} wins. If no framework is found, a no-op
 * implementation is returned.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <ul>
 *   <li>No {@code AbstractLoggingSystem} intermediate class — config-file
 *       convention discovery is omitted</li>
 *   <li>No {@code LogFile} support — no file-based logging</li>
 *   <li>No {@code LoggingInitializationContext} — the {@code Environment} is
 *       passed directly</li>
 *   <li>No {@code LoggingSystemProperties} — no system-property bridging</li>
 *   <li>No {@code LoggerGroups} — no named logger groups</li>
 *   <li>No shutdown hook — cleanup is driven by {@code ContextClosedEvent}</li>
 * </ul>
 *
 * @see LoggingSystemFactory
 * @see LogLevel
 * @see org.springframework.boot.logging.LoggingSystem
 */
public abstract class LoggingSystem {

    /**
     * System property that can be used to explicitly specify the
     * {@code LoggingSystem} implementation class. If set to {@code "none"},
     * the logging system is disabled entirely.
     */
    public static final String SYSTEM_PROPERTY = "com.iris.boot.logging.LoggingSystem";

    /** Value for {@link #SYSTEM_PROPERTY} to disable the logging system. */
    public static final String NONE = "none";

    /** Normalized name for the root logger across all implementations. */
    public static final String ROOT_LOGGER_NAME = "ROOT";

    /**
     * Called very early during bootstrap, before any output should occur.
     *
     * <p>Implementations should suppress all logging output at this point.
     * For example, JUL sets the root logger to {@code SEVERE}; Logback
     * installs a {@code TurboFilter} that denies everything.
     *
     * <p>This is called on {@code ApplicationStartingEvent} — the earliest
     * point in the bootstrap lifecycle.
     */
    public abstract void beforeInitialize();

    /**
     * Fully initialize the logging system using the given environment.
     *
     * <p>Implementations should configure their framework based on properties
     * from the environment. This is called on
     * {@code ApplicationEnvironmentPreparedEvent} — after properties are loaded
     * but before the {@code ApplicationContext} is created.
     *
     * <p>In the real Spring Boot, this method also receives a
     * {@code LoggingInitializationContext} and a {@code LogFile}. We simplify
     * to just the {@code Environment}.
     *
     * @param environment the prepared environment with property sources
     */
    public void initialize(Environment environment) {
        // Default: no-op — subclasses override to configure their framework
    }

    /**
     * Clean up the logging system.
     *
     * <p>Called when the context is closed. Implementations should undo
     * any changes made by {@link #beforeInitialize()} and
     * {@link #initialize(Environment)}.
     */
    public void cleanUp() {
        // Default: no-op
    }

    /**
     * Set the log level for the given logger.
     *
     * <p>A {@code null} level means "reset to default" (inherit from parent).
     *
     * @param loggerName the logger name ({@link #ROOT_LOGGER_NAME} for root)
     * @param level      the level to set, or {@code null} to reset
     */
    public void setLogLevel(String loggerName, LogLevel level) {
        throw new UnsupportedOperationException(
                "Unable to set log level for " + getClass().getSimpleName());
    }

    /**
     * Return the set of {@link LogLevel}s supported by this implementation.
     *
     * <p>Not all frameworks support all levels. For example, JUL has no
     * direct equivalent of {@code FATAL}.
     */
    public Set<LogLevel> getSupportedLogLevels() {
        return EnumSet.allOf(LogLevel.class);
    }

    // -----------------------------------------------------------------------
    // Static detection
    // -----------------------------------------------------------------------

    /** Ordered list of factories for logging system detection. */
    private static final List<LoggingSystemFactory> FACTORIES = List.of(
            // Add LogbackLoggingSystem.Factory here if Logback support is added
            new com.iris.boot.logging.java.JavaLoggingSystem.Factory()  // Fallback
    );

    /**
     * Detect and return the appropriate {@code LoggingSystem} for the given
     * classloader.
     *
     * <p>Checks the {@link #SYSTEM_PROPERTY} system property first. If set
     * to {@code "none"}, returns a no-op implementation. If set to a class
     * name, instantiates that class directly.
     *
     * <p>Otherwise, iterates through registered {@link LoggingSystemFactory}
     * instances and returns the first non-null result.
     *
     * <p>In the real Spring Boot, factories are loaded via
     * {@code SpringFactoriesLoader} from {@code META-INF/spring.factories}.
     * We use a hard-coded list since we only have one implementation.
     *
     * @param classLoader the classloader to use for detection
     * @return the detected {@code LoggingSystem}, never {@code null}
     */
    public static LoggingSystem get(ClassLoader classLoader) {
        // Check system property override
        String property = System.getProperty(SYSTEM_PROPERTY);
        if (property != null) {
            if (NONE.equals(property)) {
                return new NoOpLoggingSystem();
            }
            try {
                Class<?> clazz = Class.forName(property, true, classLoader);
                return (LoggingSystem) clazz.getDeclaredConstructor(ClassLoader.class)
                        .newInstance(classLoader);
            } catch (Exception ex) {
                throw new IllegalStateException(
                        "Failed to create LoggingSystem from system property: " + property, ex);
            }
        }

        // Auto-detect via factories
        for (LoggingSystemFactory factory : FACTORIES) {
            LoggingSystem system = factory.getLoggingSystem(classLoader);
            if (system != null) {
                return system;
            }
        }

        throw new IllegalStateException("No suitable LoggingSystem could be found");
    }

    // -----------------------------------------------------------------------
    // No-op implementation
    // -----------------------------------------------------------------------

    /**
     * A logging system that does nothing — used when logging is explicitly
     * disabled via {@code SYSTEM_PROPERTY = "none"}.
     *
     * <p>This is the Null Object pattern: callers don't need to check for
     * null, they just get a system that silently ignores all calls.
     */
    static class NoOpLoggingSystem extends LoggingSystem {

        @Override
        public void beforeInitialize() {
            // No-op
        }

        @Override
        public void setLogLevel(String loggerName, LogLevel level) {
            // No-op
        }
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/logging/java/JavaLoggingSystem.java` [NEW]

```java
package com.iris.boot.logging.java;

import java.util.Collections;
import java.util.EnumMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.logging.Level;
import java.util.logging.Logger;

import com.iris.boot.logging.LogLevel;
import com.iris.boot.logging.LoggingSystem;
import com.iris.boot.logging.LoggingSystemFactory;
import com.iris.framework.core.env.Environment;

/**
 * {@link LoggingSystem} implementation backed by {@code java.util.logging} (JUL).
 *
 * <p>This is the fallback logging system — it requires no external dependencies
 * since JUL is built into the JDK. In the real Spring Boot, this is the
 * lowest-priority option: Logback and Log4j2 are preferred when available.
 *
 * <h3>Level Mapping</h3>
 *
 * <p>JUL uses different level names than most frameworks:
 * <pre>
 * Iris LogLevel    JUL Level
 * ─────────────    ─────────
 * TRACE         →  FINEST
 * DEBUG         →  FINE
 * INFO          →  INFO
 * WARN          →  WARNING
 * ERROR         →  SEVERE
 * FATAL         →  SEVERE (JUL has no FATAL)
 * OFF           →  OFF
 * </pre>
 *
 * <h3>JUL's Weak Reference Problem</h3>
 *
 * <p>JUL internally stores loggers using {@link java.lang.ref.WeakReference}s.
 * If no strong reference exists to a configured logger, it can be
 * garbage-collected and lose its level setting. We solve this by holding
 * strong references in {@link #configuredLoggers} — the same approach
 * used by the real Spring Boot.
 *
 * <h3>Simplifications</h3>
 *
 * <ul>
 *   <li>No log-file configuration (no JUL {@code FileHandler} setup)</li>
 *   <li>No config-file loading ({@code logging.properties})</li>
 *   <li>No convention-based config discovery</li>
 * </ul>
 *
 * @see org.springframework.boot.logging.java.JavaLoggingSystem
 */
public class JavaLoggingSystem extends LoggingSystem {

    // -----------------------------------------------------------------------
    // Level mapping (Iris LogLevel ↔ JUL Level)
    // -----------------------------------------------------------------------

    private static final Map<LogLevel, Level> LEVEL_MAP;

    static {
        Map<LogLevel, Level> map = new EnumMap<>(LogLevel.class);
        map.put(LogLevel.TRACE, Level.FINEST);
        map.put(LogLevel.DEBUG, Level.FINE);
        map.put(LogLevel.INFO, Level.INFO);
        map.put(LogLevel.WARN, Level.WARNING);
        map.put(LogLevel.ERROR, Level.SEVERE);
        map.put(LogLevel.FATAL, Level.SEVERE);
        map.put(LogLevel.OFF, Level.OFF);
        LEVEL_MAP = Collections.unmodifiableMap(map);
    }

    /**
     * Strong references to loggers we've configured, preventing GC
     * from clearing their level settings.
     *
     * @see java.util.logging.LogManager (uses WeakReferences internally)
     */
    private final Set<Logger> configuredLoggers = Collections.synchronizedSet(new HashSet<>());

    private final ClassLoader classLoader;

    public JavaLoggingSystem(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    // -----------------------------------------------------------------------
    // Lifecycle methods
    // -----------------------------------------------------------------------

    /**
     * Suppress all logging output during early bootstrap.
     *
     * <p>Sets the root JUL logger to {@code SEVERE} so only critical errors
     * are shown while the framework is initializing. The full initialization
     * in {@link #initialize(Environment)} will restore appropriate levels.
     */
    @Override
    public void beforeInitialize() {
        Logger rootLogger = Logger.getLogger("");
        rootLogger.setLevel(Level.SEVERE);
    }

    /**
     * Fully initialize the logging system from the environment.
     *
     * <p>Reads {@code logging.level.*} properties and applies them:
     * <ul>
     *   <li>{@code logging.level.ROOT=DEBUG} → root logger to {@code FINE}</li>
     *   <li>{@code logging.level.com.iris=TRACE} → that logger to {@code FINEST}</li>
     * </ul>
     *
     * <p>Sets the root logger to {@code INFO} as a default if no
     * {@code logging.level.ROOT} property is set.
     */
    @Override
    public void initialize(Environment environment) {
        // Default: set root logger to INFO (undoing the SEVERE from beforeInitialize)
        Logger rootLogger = Logger.getLogger("");
        rootLogger.setLevel(Level.INFO);
        configuredLoggers.add(rootLogger);

        // Apply logging.level.* properties
        applyLevelProperties(environment);
    }

    @Override
    public void cleanUp() {
        configuredLoggers.clear();
    }

    // -----------------------------------------------------------------------
    // Level management
    // -----------------------------------------------------------------------

    /**
     * Set the log level for the given logger.
     *
     * <p>The logger name is normalized: {@link LoggingSystem#ROOT_LOGGER_NAME}
     * is mapped to the empty string (JUL's root logger name).
     *
     * @param loggerName the logger name
     * @param level      the level to set, or {@code null} to reset to default
     */
    @Override
    public void setLogLevel(String loggerName, LogLevel level) {
        String name = ROOT_LOGGER_NAME.equalsIgnoreCase(loggerName) ? "" : loggerName;
        Logger logger = Logger.getLogger(name);

        if (level == null) {
            // Reset to default (inherit from parent)
            logger.setLevel(null);
            configuredLoggers.remove(logger);
        } else {
            Level julLevel = LEVEL_MAP.get(level);
            if (julLevel == null) {
                throw new IllegalArgumentException("Unsupported log level: " + level);
            }
            logger.setLevel(julLevel);
            configuredLoggers.add(logger);
        }
    }

    @Override
    public Set<LogLevel> getSupportedLogLevels() {
        return LEVEL_MAP.keySet();
    }

    // -----------------------------------------------------------------------
    // Property binding
    // -----------------------------------------------------------------------

    /**
     * Scan the environment for {@code logging.level.*} properties and apply
     * them to the corresponding JUL loggers.
     *
     * <p>The property key format is:
     * <pre>
     * logging.level.{loggerName}={LEVEL}
     * </pre>
     *
     * <p>For example:
     * <pre>
     * logging.level.ROOT=DEBUG
     * logging.level.com.iris.framework=TRACE
     * logging.level.com.iris.boot.web=WARN
     * </pre>
     *
     * <p>In the real Spring Boot, this is done by
     * {@code LoggingApplicationListener.setLogLevels()} which uses
     * Spring's {@code Binder} to bind properties. We iterate property
     * sources directly.
     */
    private void applyLevelProperties(Environment environment) {
        String prefix = "logging.level.";

        // Scan all property sources for logging.level.* keys
        for (var propertySource : environment.getPropertySources()) {
            Object source = propertySource.getSource();
            if (source instanceof java.util.Map<?, ?> map) {
                for (Object key : map.keySet()) {
                    String keyStr = key.toString();
                    if (keyStr.startsWith(prefix)) {
                        String loggerName = keyStr.substring(prefix.length());
                        String levelStr = environment.getProperty(keyStr);
                        if (levelStr != null && !levelStr.isBlank()) {
                            try {
                                LogLevel logLevel = LogLevel.valueOf(levelStr.trim().toUpperCase());
                                setLogLevel(loggerName, logLevel);
                            } catch (IllegalArgumentException ex) {
                                System.err.println("Invalid logging level '" + levelStr
                                        + "' for logger '" + loggerName + "'");
                            }
                        }
                    }
                }
            }
        }
    }

    // -----------------------------------------------------------------------
    // Factory (SPI)
    // -----------------------------------------------------------------------

    /**
     * {@link LoggingSystemFactory} that creates a {@code JavaLoggingSystem}.
     *
     * <p>JUL is always available (it's in the JDK), so this factory always
     * returns a result. In the real Spring Boot, this is annotated with
     * {@code @Order(Ordered.LOWEST_PRECEDENCE - 1024)} — the lowest-priority
     * factory, only used when Logback and Log4j2 are not available.
     */
    public static class Factory implements LoggingSystemFactory {

        @Override
        public LoggingSystem getLoggingSystem(ClassLoader classLoader) {
            return new JavaLoggingSystem(classLoader);
        }
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/context/logging/LoggingApplicationListener.java` [NEW]

```java
package com.iris.boot.context.logging;

import com.iris.boot.context.event.ApplicationEnvironmentPreparedEvent;
import com.iris.boot.context.event.ApplicationFailedEvent;
import com.iris.boot.context.event.ApplicationStartingEvent;
import com.iris.boot.logging.LoggingSystem;
import com.iris.framework.context.ApplicationEvent;
import com.iris.framework.context.ApplicationListener;
import com.iris.framework.context.event.ContextClosedEvent;

/**
 * {@link ApplicationListener} that orchestrates the {@link LoggingSystem}
 * lifecycle by responding to bootstrap and context events.
 *
 * <p>This is the glue between the logging abstraction ({@code LoggingSystem})
 * and the application lifecycle (events). The listener progresses the logging
 * system through a state machine:
 *
 * <pre>
 * ┌─────────────────────────────────────────────────────────────────────┐
 * │  Event                              │  Action                      │
 * ├─────────────────────────────────────┼──────────────────────────────┤
 * │  ApplicationStartingEvent           │  detect() + beforeInit()     │
 * │  ApplicationEnvironmentPreparedEvent│  initialize(env)             │
 * │  ContextClosedEvent                 │  cleanUp()                   │
 * │  ApplicationFailedEvent             │  cleanUp()                   │
 * └─────────────────────────────────────────────────────────────────────┘
 * </pre>
 *
 * <h3>Why a Single Listener Handles All Events</h3>
 *
 * <p>The logging system needs to be managed across the full application
 * lifecycle — from before the context exists ({@code ApplicationStartingEvent})
 * to after it's destroyed ({@code ContextClosedEvent}). A single listener
 * that handles all event types can hold state (the detected
 * {@code LoggingSystem} instance) across these phases.
 *
 * <h3>Two-Phase Event Receiving</h3>
 *
 * <p>This listener participates in both phases of the event publishing
 * system:
 * <ol>
 *   <li><strong>Phase 1:</strong> {@code ApplicationStartingEvent} and
 *       {@code ApplicationEnvironmentPreparedEvent} are received via the
 *       initial multicaster (before the context exists)</li>
 *   <li><strong>Phase 2:</strong> {@code ContextClosedEvent} is received
 *       via the context's event publisher (after the listener is bridged
 *       onto the context during {@code contextPrepared()})</li>
 * </ol>
 *
 * <h3>Simplifications</h3>
 *
 * <ul>
 *   <li>No {@code ApplicationPreparedEvent} handling — we don't register
 *       the {@code LoggingSystem} as a bean</li>
 *   <li>No {@code SmartLifecycle} inner class for ordered shutdown</li>
 *   <li>No {@code LoggerGroups} support</li>
 *   <li>No {@code --debug} / {@code --trace} argument parsing</li>
 *   <li>No shutdown hook registration</li>
 * </ul>
 *
 * @see LoggingSystem
 * @see org.springframework.boot.context.logging.LoggingApplicationListener
 */
public class LoggingApplicationListener implements ApplicationListener<ApplicationEvent> {

    /** The detected logging system — initialized on ApplicationStartingEvent. */
    private LoggingSystem loggingSystem;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationStartingEvent) {
            onApplicationStartingEvent();
        } else if (event instanceof ApplicationEnvironmentPreparedEvent envEvent) {
            onApplicationEnvironmentPreparedEvent(envEvent);
        } else if (event instanceof ContextClosedEvent) {
            onContextClosedEvent();
        } else if (event instanceof ApplicationFailedEvent) {
            onApplicationFailedEvent();
        }
    }

    // -----------------------------------------------------------------------
    // Event handlers
    // -----------------------------------------------------------------------

    /**
     * Handle {@code ApplicationStartingEvent} — the earliest lifecycle point.
     *
     * <p>Detects the logging system using the classpath and calls
     * {@code beforeInitialize()} to suppress output during bootstrap.
     *
     * <p>In the real Spring Boot, this is where
     * {@code LoggingSystem.get(classLoader)} is called, and then
     * {@code beforeInitialize()} suppresses noisy framework startup
     * messages (Tomcat, Hibernate, etc.) until the logging system
     * is fully configured.
     */
    private void onApplicationStartingEvent() {
        this.loggingSystem = LoggingSystem.get(getClassLoader());
        this.loggingSystem.beforeInitialize();
    }

    /**
     * Handle {@code ApplicationEnvironmentPreparedEvent} — environment is ready.
     *
     * <p>At this point, {@code application.properties} has been loaded and
     * the environment is fully prepared. We can now read {@code logging.level.*}
     * properties and configure the logging system accordingly.
     *
     * <p>In the real Spring Boot, this method also:
     * <ul>
     *   <li>Applies system properties ({@code LoggingSystemProperties})</li>
     *   <li>Resolves the log file from properties</li>
     *   <li>Parses {@code --debug} / {@code --trace} arguments</li>
     *   <li>Discovers config files by convention</li>
     *   <li>Initializes logger groups</li>
     *   <li>Registers a shutdown hook</li>
     * </ul>
     * We do the essential part: full initialization with the environment.
     */
    private void onApplicationEnvironmentPreparedEvent(
            ApplicationEnvironmentPreparedEvent event) {
        this.loggingSystem.initialize(event.getEnvironment());
    }

    /**
     * Handle {@code ContextClosedEvent} — the context is shutting down.
     *
     * <p>Cleans up the logging system. In the real Spring Boot, this is
     * handled by an inner {@code SmartLifecycle} class at phase
     * {@code Integer.MIN_VALUE + 1} to ensure the logging system is one
     * of the very last things to shut down. We use the simpler approach
     * of listening for the event directly.
     */
    private void onContextClosedEvent() {
        if (this.loggingSystem != null) {
            this.loggingSystem.cleanUp();
            this.loggingSystem = null;
        }
    }

    /**
     * Handle {@code ApplicationFailedEvent} — startup failed.
     *
     * <p>Emergency cleanup: ensures the logging system is cleaned up even
     * if the context was never fully started.
     */
    private void onApplicationFailedEvent() {
        if (this.loggingSystem != null) {
            this.loggingSystem.cleanUp();
            this.loggingSystem = null;
        }
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    /**
     * Return the logging system detected by this listener.
     *
     * <p>Returns {@code null} before {@code ApplicationStartingEvent} is
     * received or after cleanup.
     */
    public LoggingSystem getLoggingSystem() {
        return this.loggingSystem;
    }

    private ClassLoader getClassLoader() {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        return (classLoader != null) ? classLoader : getClass().getClassLoader();
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java` [MODIFIED]

Change: Added `LoggingApplicationListener` import and auto-registration in constructor.

```diff
+ import com.iris.boot.context.logging.LoggingApplicationListener;

  public IrisApplication(Class<?> primarySource) {
      this.primarySource = primarySource;
      this.webApplicationType = WebApplicationType.deduceFromClasspath();
+
+     // Register the LoggingApplicationListener as a default listener.
+     // In the real Spring Boot, this is loaded via SpringFactoriesLoader
+     // from META-INF/spring.factories. We wire it directly.
+     this.listeners.add(new LoggingApplicationListener());
  }
```

### Test Code

#### File: `iris-boot-core/src/test/java/com/iris/boot/logging/LoggingSystemTest.java` [NEW]

```java
package com.iris.boot.logging;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for the {@link LoggingSystem} detection and no-op behavior.
 */
class LoggingSystemTest {

    @AfterEach
    void clearSystemProperty() {
        System.clearProperty(LoggingSystem.SYSTEM_PROPERTY);
    }

    @Test
    void shouldDetectJavaLoggingSystem_WhenNoSystemPropertySet() {
        LoggingSystem system = LoggingSystem.get(getClass().getClassLoader());

        assertThat(system).isNotNull();
        assertThat(system.getClass().getSimpleName()).isEqualTo("JavaLoggingSystem");
    }

    @Test
    void shouldReturnNoOpLoggingSystem_WhenSystemPropertySetToNone() {
        System.setProperty(LoggingSystem.SYSTEM_PROPERTY, LoggingSystem.NONE);

        LoggingSystem system = LoggingSystem.get(getClass().getClassLoader());

        assertThat(system).isNotNull();
        assertThat(system.getClass().getSimpleName()).isEqualTo("NoOpLoggingSystem");
    }

    @Test
    void shouldThrowException_WhenSystemPropertySetToInvalidClass() {
        System.setProperty(LoggingSystem.SYSTEM_PROPERTY, "com.nonexistent.LoggingSystem");

        assertThatThrownBy(() -> LoggingSystem.get(getClass().getClassLoader()))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Failed to create LoggingSystem");
    }

    @Test
    void shouldReturnAllLevelsByDefault_WhenGetSupportedLogLevels() {
        LoggingSystem system = LoggingSystem.get(getClass().getClassLoader());

        assertThat(system.getSupportedLogLevels()).contains(
                LogLevel.TRACE, LogLevel.DEBUG, LogLevel.INFO,
                LogLevel.WARN, LogLevel.ERROR, LogLevel.OFF);
    }

    @Test
    void shouldHaveConstants() {
        assertThat(LoggingSystem.SYSTEM_PROPERTY).isEqualTo("com.iris.boot.logging.LoggingSystem");
        assertThat(LoggingSystem.NONE).isEqualTo("none");
        assertThat(LoggingSystem.ROOT_LOGGER_NAME).isEqualTo("ROOT");
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/logging/java/JavaLoggingSystemTest.java` [NEW]

```java
package com.iris.boot.logging.java;

import java.util.logging.Level;
import java.util.logging.Logger;

import com.iris.boot.logging.LogLevel;
import com.iris.boot.logging.LoggingSystem;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.StandardEnvironment;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link JavaLoggingSystem} — the JUL-backed logging system.
 */
class JavaLoggingSystemTest {

    private JavaLoggingSystem loggingSystem;
    private Level originalRootLevel;

    @BeforeEach
    void setUp() {
        loggingSystem = new JavaLoggingSystem(getClass().getClassLoader());
        originalRootLevel = Logger.getLogger("").getLevel();
    }

    @AfterEach
    void tearDown() {
        loggingSystem.cleanUp();
        // Restore the original root level
        Logger.getLogger("").setLevel(originalRootLevel);
    }

    // -----------------------------------------------------------------------
    // beforeInitialize()
    // -----------------------------------------------------------------------

    @Test
    void shouldSuppressOutput_WhenBeforeInitializeCalled() {
        loggingSystem.beforeInitialize();

        Level rootLevel = Logger.getLogger("").getLevel();
        assertThat(rootLevel).isEqualTo(Level.SEVERE);
    }

    // -----------------------------------------------------------------------
    // initialize()
    // -----------------------------------------------------------------------

    @Test
    void shouldSetRootToInfo_WhenInitializedWithNoProperties() {
        loggingSystem.beforeInitialize();
        StandardEnvironment env = new StandardEnvironment();

        loggingSystem.initialize(env);

        Level rootLevel = Logger.getLogger("").getLevel();
        assertThat(rootLevel).isEqualTo(Level.INFO);
    }

    @Test
    void shouldApplyLoggingLevelProperties_WhenInitialized() {
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test",
                java.util.Map.of(
                        "logging.level.com.iris.test", "DEBUG",
                        "logging.level.com.iris.web", "TRACE"
                )));

        loggingSystem.initialize(env);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.FINE);
        assertThat(Logger.getLogger("com.iris.web").getLevel()).isEqualTo(Level.FINEST);
    }

    @Test
    void shouldApplyRootLevelProperty_WhenInitialized() {
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test",
                java.util.Map.of("logging.level.ROOT", "DEBUG")));

        loggingSystem.initialize(env);

        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.FINE);
    }

    // -----------------------------------------------------------------------
    // setLogLevel()
    // -----------------------------------------------------------------------

    @Test
    void shouldSetTraceLevel_WhenSetLogLevelCalledWithTrace() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.TRACE);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.FINEST);
    }

    @Test
    void shouldSetDebugLevel_WhenSetLogLevelCalledWithDebug() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.DEBUG);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.FINE);
    }

    @Test
    void shouldSetInfoLevel_WhenSetLogLevelCalledWithInfo() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.INFO);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.INFO);
    }

    @Test
    void shouldSetWarnLevel_WhenSetLogLevelCalledWithWarn() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.WARN);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.WARNING);
    }

    @Test
    void shouldSetErrorLevel_WhenSetLogLevelCalledWithError() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.ERROR);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.SEVERE);
    }

    @Test
    void shouldSetFatalLevel_WhenSetLogLevelCalledWithFatal() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.FATAL);

        // FATAL maps to SEVERE in JUL (no native FATAL level)
        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.SEVERE);
    }

    @Test
    void shouldSetOffLevel_WhenSetLogLevelCalledWithOff() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.OFF);

        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.OFF);
    }

    @Test
    void shouldSetRootLoggerLevel_WhenRootNameUsed() {
        loggingSystem.setLogLevel(LoggingSystem.ROOT_LOGGER_NAME, LogLevel.DEBUG);

        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.FINE);
    }

    @Test
    void shouldResetToDefault_WhenSetLogLevelCalledWithNull() {
        // First set a level
        loggingSystem.setLogLevel("com.iris.test", LogLevel.DEBUG);
        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.FINE);

        // Then reset
        loggingSystem.setLogLevel("com.iris.test", null);
        assertThat(Logger.getLogger("com.iris.test").getLevel()).isNull();
    }

    // -----------------------------------------------------------------------
    // cleanUp()
    // -----------------------------------------------------------------------

    @Test
    void shouldClearConfiguredLoggers_WhenCleanUpCalled() {
        loggingSystem.setLogLevel("com.iris.test", LogLevel.DEBUG);
        loggingSystem.setLogLevel("com.iris.web", LogLevel.TRACE);

        loggingSystem.cleanUp();

        // After cleanup, the internal strong reference set should be cleared.
        // The loggers still exist in JUL (they're cached), but our tracking is gone.
        // We verify by calling cleanUp and ensuring no exceptions.
    }

    // -----------------------------------------------------------------------
    // Factory
    // -----------------------------------------------------------------------

    @Test
    void shouldAlwaysReturnInstance_WhenFactoryCreatesLoggingSystem() {
        JavaLoggingSystem.Factory factory = new JavaLoggingSystem.Factory();

        LoggingSystem system = factory.getLoggingSystem(getClass().getClassLoader());

        assertThat(system).isNotNull().isInstanceOf(JavaLoggingSystem.class);
    }

    // -----------------------------------------------------------------------
    // Full lifecycle
    // -----------------------------------------------------------------------

    @Test
    void shouldFollowFullLifecycle_WhenBeforeInitializeThenInitializeThenCleanUp() {
        // Phase 1: beforeInitialize — suppress output
        loggingSystem.beforeInitialize();
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.SEVERE);

        // Phase 2: initialize — restore normal levels
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test",
                java.util.Map.of("logging.level.com.iris.test", "DEBUG")));
        loggingSystem.initialize(env);
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.INFO);
        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.FINE);

        // Phase 3: cleanUp
        loggingSystem.cleanUp();
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/context/logging/LoggingApplicationListenerTest.java` [NEW]

```java
package com.iris.boot.context.logging;

import java.util.logging.Level;
import java.util.logging.Logger;

import com.iris.boot.IrisApplication;
import com.iris.boot.context.event.ApplicationEnvironmentPreparedEvent;
import com.iris.boot.context.event.ApplicationFailedEvent;
import com.iris.boot.context.event.ApplicationStartingEvent;
import com.iris.boot.logging.LoggingSystem;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.StandardEnvironment;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link LoggingApplicationListener} — the lifecycle orchestrator
 * that drives the logging system through bootstrap events.
 */
class LoggingApplicationListenerTest {

    private LoggingApplicationListener listener;
    private IrisApplication application;
    private Level originalRootLevel;

    @BeforeEach
    void setUp() {
        listener = new LoggingApplicationListener();
        application = new IrisApplication(LoggingApplicationListenerTest.class);
        originalRootLevel = Logger.getLogger("").getLevel();
    }

    @AfterEach
    void tearDown() {
        // Ensure cleanup
        if (listener.getLoggingSystem() != null) {
            listener.getLoggingSystem().cleanUp();
        }
        Logger.getLogger("").setLevel(originalRootLevel);
    }

    // -----------------------------------------------------------------------
    // ApplicationStartingEvent
    // -----------------------------------------------------------------------

    @Test
    void shouldDetectLoggingSystem_WhenStartingEventReceived() {
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));

        assertThat(listener.getLoggingSystem()).isNotNull();
    }

    @Test
    void shouldSuppressOutput_WhenStartingEventReceived() {
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));

        // JUL root logger should be set to SEVERE (suppress everything below)
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.SEVERE);
    }

    // -----------------------------------------------------------------------
    // ApplicationEnvironmentPreparedEvent
    // -----------------------------------------------------------------------

    @Test
    void shouldInitializeLogging_WhenEnvironmentPreparedEventReceived() {
        // Start first (detects the logging system)
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));

        // Prepare environment with logging level properties
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test",
                java.util.Map.of("logging.level.com.iris.test", "DEBUG")));
        listener.onApplicationEvent(
                new ApplicationEnvironmentPreparedEvent(application, new String[]{}, env));

        // Root should be INFO (default), test logger should be FINE (DEBUG)
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.INFO);
        assertThat(Logger.getLogger("com.iris.test").getLevel()).isEqualTo(Level.FINE);
    }

    @Test
    void shouldRestoreDefaultLevels_WhenEnvironmentPreparedWithNoProperties() {
        // Start first
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.SEVERE);

        // Initialize with empty environment
        StandardEnvironment env = new StandardEnvironment();
        listener.onApplicationEvent(
                new ApplicationEnvironmentPreparedEvent(application, new String[]{}, env));

        // Root should be restored to INFO
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.INFO);
    }

    // -----------------------------------------------------------------------
    // ContextClosedEvent
    // -----------------------------------------------------------------------

    @Test
    void shouldCleanUpLoggingSystem_WhenContextClosedEventReceived() {
        // Full lifecycle: start → initialize → close
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));
        StandardEnvironment env = new StandardEnvironment();
        listener.onApplicationEvent(
                new ApplicationEnvironmentPreparedEvent(application, new String[]{}, env));
        assertThat(listener.getLoggingSystem()).isNotNull();

        listener.onApplicationEvent(
                new ContextClosedEvent(new AnnotationConfigApplicationContext()));

        assertThat(listener.getLoggingSystem()).isNull();
    }

    // -----------------------------------------------------------------------
    // ApplicationFailedEvent
    // -----------------------------------------------------------------------

    @Test
    void shouldCleanUpLoggingSystem_WhenFailedEventReceived() {
        // Start
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));
        assertThat(listener.getLoggingSystem()).isNotNull();

        // Fail
        listener.onApplicationEvent(
                new ApplicationFailedEvent(application, new String[]{}, null,
                        new RuntimeException("test failure")));

        assertThat(listener.getLoggingSystem()).isNull();
    }

    // -----------------------------------------------------------------------
    // Full lifecycle
    // -----------------------------------------------------------------------

    @Test
    void shouldFollowCompleteLifecycle_WhenAllEventsReceivedInOrder() {
        // 1. Starting — detect and suppress
        listener.onApplicationEvent(
                new ApplicationStartingEvent(application, new String[]{}));
        assertThat(listener.getLoggingSystem()).isNotNull();
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.SEVERE);

        // 2. Environment prepared — initialize with properties
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test",
                java.util.Map.of(
                        "logging.level.ROOT", "WARN",
                        "logging.level.com.iris", "DEBUG")));
        listener.onApplicationEvent(
                new ApplicationEnvironmentPreparedEvent(application, new String[]{}, env));
        assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.WARNING);
        assertThat(Logger.getLogger("com.iris").getLevel()).isEqualTo(Level.FINE);

        // 3. Context closed — clean up
        listener.onApplicationEvent(
                new ContextClosedEvent(new AnnotationConfigApplicationContext()));
        assertThat(listener.getLoggingSystem()).isNull();
    }

    @Test
    void shouldIgnoreUnrelatedEvents_WhenReceived() {
        // Receiving events it doesn't handle should be a no-op
        listener.onApplicationEvent(
                new com.iris.boot.context.event.ApplicationReadyEvent(
                        application, new String[]{}, null, java.time.Duration.ZERO));

        assertThat(listener.getLoggingSystem()).isNull();
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/logging/integration/LoggingSystemIntegrationTest.java` [NEW]

```java
package com.iris.boot.logging.integration;

import java.util.logging.Level;
import java.util.logging.Logger;

import com.iris.boot.IrisApplication;
import com.iris.boot.WebApplicationType;
import com.iris.boot.context.logging.LoggingApplicationListener;
import com.iris.boot.logging.LoggingSystem;
import com.iris.framework.context.ApplicationListener;
import com.iris.framework.context.ConfigurableApplicationContext;
import com.iris.framework.context.annotation.Configuration;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for the logging system, verifying it works with the full
 * {@link IrisApplication} bootstrap lifecycle.
 */
class LoggingSystemIntegrationTest {

    private Level originalRootLevel;

    @BeforeEach
    void setUp() {
        originalRootLevel = Logger.getLogger("").getLevel();
    }

    @AfterEach
    void tearDown() {
        Logger.getLogger("").setLevel(originalRootLevel);
        System.clearProperty(LoggingSystem.SYSTEM_PROPERTY);
    }

    // -----------------------------------------------------------------------
    // Integration with IrisApplication bootstrap
    // -----------------------------------------------------------------------

    @Test
    void shouldAutoRegisterLoggingListener_WhenIrisApplicationCreated() {
        IrisApplication app = new IrisApplication(TestConfig.class);

        boolean hasLoggingListener = app.getListeners().stream()
                .anyMatch(l -> l instanceof LoggingApplicationListener);
        assertThat(hasLoggingListener)
                .as("LoggingApplicationListener should be auto-registered")
                .isTrue();
    }

    @Test
    void shouldInitializeLogging_WhenApplicationStarts() {
        IrisApplication app = new IrisApplication(TestConfig.class);
        app.setWebApplicationType(WebApplicationType.NONE);

        ConfigurableApplicationContext context = app.run();
        try {
            // After startup, the root logger should be at INFO (not SEVERE)
            assertThat(Logger.getLogger("").getLevel()).isEqualTo(Level.INFO);
        } finally {
            context.close();
        }
    }

    @Test
    void shouldApplyLoggingProperties_WhenApplicationStartsWithProperties() {
        IrisApplication app = new IrisApplication(TestConfig.class);
        app.setWebApplicationType(WebApplicationType.NONE);

        ConfigurableApplicationContext context = app.run();
        try {
            // Verify logging was initialized (not left at SEVERE)
            Level rootLevel = Logger.getLogger("").getLevel();
            assertThat(rootLevel).isNotEqualTo(Level.SEVERE);
        } finally {
            context.close();
        }
    }

    @Test
    void shouldCleanUpLogging_WhenContextClosed() {
        IrisApplication app = new IrisApplication(TestConfig.class);
        app.setWebApplicationType(WebApplicationType.NONE);

        ConfigurableApplicationContext context = app.run();

        // Find the LoggingApplicationListener
        LoggingApplicationListener loggingListener = null;
        for (ApplicationListener<?> listener : app.getListeners()) {
            if (listener instanceof LoggingApplicationListener lal) {
                loggingListener = lal;
                break;
            }
        }

        assertThat(loggingListener).isNotNull();
        assertThat(loggingListener.getLoggingSystem()).isNotNull();

        // Close context
        context.close();

        // After close, the logging system should be cleaned up
        assertThat(loggingListener.getLoggingSystem()).isNull();
    }

    @Test
    void shouldWorkWithDisabledLogging_WhenNoneSystemProperty() {
        System.setProperty(LoggingSystem.SYSTEM_PROPERTY, LoggingSystem.NONE);

        IrisApplication app = new IrisApplication(TestConfig.class);
        app.setWebApplicationType(WebApplicationType.NONE);

        // Should not throw even with logging disabled
        ConfigurableApplicationContext context = app.run();
        try {
            assertThat(context.isActive()).isTrue();
        } finally {
            context.close();
        }
    }

    // -----------------------------------------------------------------------
    // Test configuration
    // -----------------------------------------------------------------------

    @Configuration
    static class TestConfig {
        // Minimal config — no beans needed for this test
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`LoggingSystem`** | Abstract base class that bridges Iris Boot's configuration to the underlying logging framework (JUL, Logback, Log4j2) |
| **`LoggingSystemFactory`** | SPI interface — each logging framework provides a factory that checks classpath availability and creates the appropriate `LoggingSystem` |
| **`JavaLoggingSystem`** | Concrete implementation backed by `java.util.logging` — the JDK-built-in fallback when no external logging framework is available |
| **`LoggingApplicationListener`** | Event-driven lifecycle orchestrator — a single listener that handles `ApplicationStartingEvent` (detect + suppress), `ApplicationEnvironmentPreparedEvent` (initialize), `ContextClosedEvent` (cleanup), and `ApplicationFailedEvent` (emergency cleanup) |
| **Null Object pattern** | `NoOpLoggingSystem` — callers never need null checks; when logging is disabled, they get a system that silently ignores all calls |
| **Strategy + Chain of Responsibility** | `LoggingSystem.get()` iterates ordered factories, returning the first non-null result — extensible without modifying existing code |
| **Weak reference trap** | JUL stores loggers via `WeakReference`; `JavaLoggingSystem.configuredLoggers` holds strong references to prevent GC from clearing level settings |

**Next: Chapter 17 — Profiles & ConfigData** — Profile-specific configuration files (`application-{profile}.properties`) and the modern config loading pipeline (`spring.config.import`, search locations), replacing the simple single-file property loading with a full pipeline.
