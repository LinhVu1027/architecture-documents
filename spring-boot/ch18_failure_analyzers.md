# Chapter 18: Failure Analyzers

## Build Challenge

| | |
|---|---|
| **Current State** | When `IrisApplication.run()` fails, the user sees a raw stack trace. Circular bean dependencies cause a `StackOverflowError`. |
| **Limitation** | No human-readable error messages. No circular dependency detection. No "APPLICATION FAILED TO START" banner with description and suggested action. |
| **Objective** | Transform cryptic startup exceptions into clear error messages via a pluggable `FailureAnalyzer` system. Add circular dependency detection to `DefaultBeanFactory`. Produce the iconic "APPLICATION FAILED TO START" banner. |

---

## 18.1 The Integration Point

The new feature plugs into **one exact method**: `IrisApplication.handleRunFailure()`.

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java`
**Change:** Add failure analysis between event publication and context close

Before (Feature 8):
```java
private void handleRunFailure(ConfigurableApplicationContext context,
                                IrisApplicationRunListeners listeners,
                                Throwable exception) {
    try {
        listeners.failed(context, exception);
    } catch (Exception ex) {
        System.err.println("Failed to publish ApplicationFailedEvent: " + ex.getMessage());
    }
    if (context != null) {
        context.close();
    }
}
```

After (Feature 18):
```java
private void handleRunFailure(ConfigurableApplicationContext context,
                                IrisApplicationRunListeners listeners,
                                Throwable exception) {
    try {
        listeners.failed(context, exception);
    } catch (Exception ex) {
        System.err.println("Failed to publish ApplicationFailedEvent: " + ex.getMessage());
    }
    reportFailure(exception);
    if (context != null) {
        context.close();
    }
}

private void reportFailure(Throwable exception) {
    try {
        FailureAnalyzers analyzers = new FailureAnalyzers(getClassLoader());
        analyzers.reportException(exception);
    } catch (Throwable ex) {
        System.err.println("Unable to report failure: " + ex.getMessage());
    }
}
```

**Direction:** The `FailureAnalyzers` orchestrator is the bridge between the bootstrap catch block and the individual analyzers. It loads analyzer implementations via SPI (from `.imports` files), iterates them against the failure, and reports the first match through a `LoggingFailureAnalysisReporter`. Everything we build supports this single integration point.

We also enhance `DefaultBeanFactory` with circular dependency detection — adding a `beansCurrentlyInCreation` set so that circular references produce a recognizable `BeanCurrentlyInCreationException` instead of a `StackOverflowError`.

---

## 18.2 The Core Abstractions

### FailureAnalysis — the result

A simple record carrying what went wrong, what to do about it, and the underlying cause.

```java
package com.iris.boot.diagnostics;

public record FailureAnalysis(String description, String action, Throwable cause) {
}
```

### FailureAnalyzer — the interface

Each analyzer inspects a failure and returns a `FailureAnalysis` if it recognizes it, or `null` to pass.

```java
package com.iris.boot.diagnostics;

@FunctionalInterface
public interface FailureAnalyzer {
    FailureAnalysis analyze(Throwable failure);
}
```

### AbstractFailureAnalyzer — the cause chain walker

The key abstraction: parameterized by the exception type it handles, it walks the cause chain looking for a match.

```java
package com.iris.boot.diagnostics;

public abstract class AbstractFailureAnalyzer<T extends Throwable> implements FailureAnalyzer {

    @Override
    public FailureAnalysis analyze(Throwable failure) {
        T cause = findCause(failure, getCauseType());
        if (cause != null) {
            return analyze(failure, cause);
        }
        return null;
    }

    protected abstract FailureAnalysis analyze(Throwable rootFailure, T cause);

    @SuppressWarnings("unchecked")
    protected Class<T> getCauseType() {
        Type superclass = getClass().getGenericSuperclass();
        while (superclass != null) {
            if (superclass instanceof ParameterizedType pt) {
                Type raw = pt.getRawType();
                if (raw == AbstractFailureAnalyzer.class) {
                    Type arg = pt.getActualTypeArguments()[0];
                    if (arg instanceof Class<?>) {
                        return (Class<T>) arg;
                    }
                }
            }
            if (superclass instanceof Class<?> cls) {
                superclass = cls.getGenericSuperclass();
            } else {
                break;
            }
        }
        throw new IllegalStateException(
                "Unable to resolve cause type for " + getClass().getName());
    }

    @SuppressWarnings("unchecked")
    protected <E extends Throwable> E findCause(Throwable failure, Class<E> type) {
        Throwable candidate = failure;
        while (candidate != null) {
            if (type.isInstance(candidate)) {
                return (E) candidate;
            }
            candidate = candidate.getCause();
        }
        return null;
    }
}
```

The `getCauseType()` method resolves the generic parameter `T` at runtime using `getGenericSuperclass()`. When you write `class MyAnalyzer extends AbstractFailureAnalyzer<MyException>`, the method finds `MyException.class`. In the real Spring Boot, this uses `ResolvableType` — a full generic type resolution utility. We inline the essential logic since we only need one generic parameter from a direct subclass.

---

## 18.3 The Orchestrator and Reporter

### FailureAnalyzers — load, iterate, report

```java
package com.iris.boot.diagnostics;

public class FailureAnalyzers {

    private final List<FailureAnalyzer> analyzers;

    public FailureAnalyzers(ClassLoader classLoader) {
        this.analyzers = loadFailureAnalyzers(classLoader);
    }

    public boolean reportException(Throwable failure) {
        FailureAnalysis analysis = analyze(failure);
        if (analysis != null) {
            new LoggingFailureAnalysisReporter().report(analysis);
            return true;
        }
        return false;
    }

    private FailureAnalysis analyze(Throwable failure) {
        for (FailureAnalyzer analyzer : this.analyzers) {
            try {
                FailureAnalysis analysis = analyzer.analyze(failure);
                if (analysis != null) {
                    return analysis;
                }
            } catch (Throwable ex) {
                // A broken analyzer must never mask the original failure
                logger.log(Level.FINE, "FailureAnalyzer " + analyzer.getClass().getName() + " failed", ex);
            }
        }
        return null;
    }

    private static List<FailureAnalyzer> loadFailureAnalyzers(ClassLoader classLoader) {
        ImportCandidates candidates = ImportCandidates.load(FailureAnalyzer.class, classLoader);
        List<FailureAnalyzer> analyzers = new ArrayList<>();
        for (String className : candidates.getCandidates()) {
            try {
                Class<?> clazz = Class.forName(className, true, classLoader);
                Object instance = clazz.getDeclaredConstructor().newInstance();
                if (instance instanceof FailureAnalyzer fa) {
                    analyzers.add(fa);
                }
            } catch (Throwable ex) {
                logger.log(Level.FINE, "Unable to instantiate FailureAnalyzer: " + className, ex);
            }
        }
        return analyzers;
    }
}
```

This reuses the `ImportCandidates` SPI from Feature 13 (Auto-Configuration Engine) — the same `.imports` file mechanism. Analyzer class names are listed in `META-INF/iris/com.iris.boot.diagnostics.FailureAnalyzer.imports`.

### LoggingFailureAnalysisReporter — the banner

```java
package com.iris.boot.diagnostics;

public class LoggingFailureAnalysisReporter implements FailureAnalysisReporter {

    @Override
    public void report(FailureAnalysis analysis) {
        if (logger.isLoggable(Level.FINE)) {
            logger.log(Level.FINE, "Application startup failed due to an exception",
                    analysis.cause());
        }
        String message = buildMessage(analysis);
        System.err.println(message);
    }

    private String buildMessage(FailureAnalysis analysis) {
        StringBuilder sb = new StringBuilder();
        sb.append("\n***************************\n");
        sb.append("APPLICATION FAILED TO START\n");
        sb.append("***************************\n\n");
        sb.append("Description:\n\n");
        sb.append(analysis.description());
        sb.append("\n");
        if (analysis.action() != null) {
            sb.append("\nAction:\n\n");
            sb.append(analysis.action());
            sb.append("\n");
        }
        return sb.toString();
    }
}
```

---

## 18.4 The Concrete Analyzers

### BeanCurrentlyInCreationFailureAnalyzer — circular dependencies

First, we need a recognizable exception. Add to `iris-framework`:

```java
package com.iris.framework.beans.factory;

public class BeanCurrentlyInCreationException extends BeanCreationException {
    public BeanCurrentlyInCreationException(String beanName) {
        super(beanName, "Requested bean is currently in creation: "
                + "Is there an unresolvable circular reference?");
    }
}
```

**Modifying:** `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`
**Change:** Add `beansCurrentlyInCreation` set and detection logic in `doGetBean()`

```java
// New field:
private final Set<String> beansCurrentlyInCreation =
        Collections.newSetFromMap(new ConcurrentHashMap<>());

// Modified doGetBean:
private Object doGetBean(String name) {
    Object singleton = this.singletonObjects.get(name);
    if (singleton != null) {
        return singleton;
    }

    BeanDefinition bd = getBeanDefinition(name);

    // Circular dependency check — if this bean is already being
    // created further up the call stack, we have a cycle.
    if (bd.isSingleton() && !this.beansCurrentlyInCreation.add(name)) {
        throw new BeanCurrentlyInCreationException(name);
    }

    try {
        Object bean = createBean(name, bd);
        if (bd.isSingleton()) {
            this.singletonObjects.put(name, bean);
        }
        return bean;
    } finally {
        this.beansCurrentlyInCreation.remove(name);
    }
}
```

The detection is elegant: `Set.add()` returns `false` if the element was already present. A bean that's already in the set means we've re-entered `doGetBean()` for the same bean — a cycle.

Now the analyzer that produces a readable message from the cause chain:

```java
package com.iris.boot.diagnostics.analyzer;

public class BeanCurrentlyInCreationFailureAnalyzer
        extends AbstractFailureAnalyzer<BeanCurrentlyInCreationException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure,
                                       BeanCurrentlyInCreationException cause) {
        List<String> cycle = findCycle(rootFailure);
        String description = buildDescription(cycle);
        String action = "Redesign your beans to remove the circular dependency. "
                + "Consider using setter injection or extracting a common "
                + "dependency into a separate bean.";
        return new FailureAnalysis(description, action, cause);
    }

    private List<String> findCycle(Throwable rootFailure) {
        List<String> beanNames = new ArrayList<>();
        Throwable current = rootFailure;
        while (current != null) {
            if (current instanceof BeanCreationException bce) {
                String name = bce.getBeanName();
                if (name != null && !beanNames.contains(name)) {
                    beanNames.add(name);
                } else if (name != null) {
                    break;  // Found the cycle start
                }
            }
            current = current.getCause();
        }
        return beanNames;
    }
}
```

The `findCycle` method walks the exception cause chain collecting bean names from `BeanCreationException`s. When a name repeats, that's where the cycle closes. The description renders an ASCII diagram using box-drawing characters.

### PortInUseFailureAnalyzer — port conflicts

First, we add a `PortInUseException` to make port-in-use errors recognizable:

```java
package com.iris.boot.web.server;

public class PortInUseException extends WebServerException {
    private final int port;

    public PortInUseException(int port, Throwable cause) {
        super("Port " + port + " is already in use", cause);
        this.port = port;
    }

    public int getPort() { return this.port; }
}
```

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatWebServer.java`
**Change:** Detect `java.net.BindException` in the catch block and throw `PortInUseException`

```java
} catch (LifecycleException ex) {
    this.started.set(false);
    Connector connector = this.tomcat.getConnector();
    if (connector != null && isPortInUse(ex)) {
        throw new PortInUseException(connector.getPort(), ex);
    }
    throw new WebServerException("Unable to start embedded Tomcat", ex);
}
```

The analyzer is the simplest of the three:

```java
package com.iris.boot.diagnostics.analyzer;

public class PortInUseFailureAnalyzer extends AbstractFailureAnalyzer<PortInUseException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, PortInUseException cause) {
        return new FailureAnalysis(
                "Web server failed to start. Port " + cause.getPort() + " was already in use.",
                "Identify and stop the process that's listening on port " + cause.getPort()
                        + " or configure this application to listen on another port.",
                cause);
    }
}
```

### BindFailureAnalyzer — property binding errors

First, add a `BindException` to make binding errors recognizable:

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBinder.java`
**Change:** Wrap conversion errors in `BindException` in the `bindSimpleField` method

```java
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
```

The analyzer extracts the property name and provides actionable guidance:

```java
package com.iris.boot.diagnostics.analyzer;

public class BindFailureAnalyzer extends AbstractFailureAnalyzer<BindException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, BindException cause) {
        String description = "Failed to bind properties under '"
                + cause.getPropertyName() + "' to " + cause.getTargetClass().getName()
                + ":\n\n    Reason: " + getRootMessage(cause);
        String action = "Update your application's configuration.\n"
                + "Check the property value for '" + cause.getPropertyName()
                + "' and ensure it is compatible with the target type.";
        return new FailureAnalysis(description, action, cause);
    }
}
```

### The SPI file

**New file:** `iris-boot-core/src/main/resources/META-INF/iris/com.iris.boot.diagnostics.FailureAnalyzer.imports`

```
# Failure Analyzers — ordered from most specific to most general
com.iris.boot.diagnostics.analyzer.BeanCurrentlyInCreationFailureAnalyzer
com.iris.boot.diagnostics.analyzer.PortInUseFailureAnalyzer
com.iris.boot.diagnostics.analyzer.BindFailureAnalyzer
```

---

## 18.5 Try It Yourself

<details><summary>Challenge 1: Implement a NoUniqueBeanDefinitionFailureAnalyzer</summary>

When the container finds multiple beans of the same type and no `@Primary` is declared, it throws `NoUniqueBeanDefinitionException`. Write an analyzer that:
- Lists the conflicting bean names
- Suggests using `@Primary` or making one of the beans `@ConditionalOnMissingBean`

```java
package com.iris.boot.diagnostics.analyzer;

import com.iris.boot.diagnostics.AbstractFailureAnalyzer;
import com.iris.boot.diagnostics.FailureAnalysis;
import com.iris.framework.beans.factory.NoUniqueBeanDefinitionException;

public class NoUniqueBeanDefinitionFailureAnalyzer
        extends AbstractFailureAnalyzer<NoUniqueBeanDefinitionException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure,
                                       NoUniqueBeanDefinitionException cause) {
        String description = "No qualifying bean of type available: "
                + "expected single matching bean but found "
                + cause.getNumberOfBeansFound() + ": "
                + cause.getBeanNamesFound();
        String action = "Consider marking one of the beans as @Primary, "
                + "or using @ConditionalOnMissingBean to ensure only one "
                + "of the conflicting beans is registered.";
        return new FailureAnalysis(description, action, cause);
    }
}
```

</details>

<details><summary>Challenge 2: Write a custom FailureAnalyzer for missing @Value properties</summary>

When `@Value("${missing.property}")` can't be resolved, it throws a `BeanCreationException` with a message containing "Could not resolve placeholder". Write an analyzer that:
- Extracts the property name from the error message
- Suggests adding the property to `application.properties`

Hint: You'll need to use `AbstractFailureAnalyzer<BeanCreationException>` and check the message.

</details>

---

## 18.6 Tests

### Unit tests — AbstractFailureAnalyzer

```java
@Test
void shouldReturnAnalysis_WhenCauseMatchesDirectly() {
    TestException cause = new TestException("direct hit");
    TestAnalyzer analyzer = new TestAnalyzer();
    FailureAnalysis result = analyzer.analyze(cause);
    assertThat(result).isNotNull();
    assertThat(result.description()).isEqualTo("Found: direct hit");
}

@Test
void shouldReturnAnalysis_WhenCauseIsNested() {
    TestException nested = new TestException("nested");
    RuntimeException wrapper = new RuntimeException("outer",
            new IllegalStateException("mid", nested));
    FailureAnalysis result = new TestAnalyzer().analyze(wrapper);
    assertThat(result).isNotNull();
    assertThat(result.description()).isEqualTo("Found: nested");
}

@Test
void shouldReturnNull_WhenCauseNotInChain() {
    RuntimeException unrelated = new RuntimeException("no match");
    assertThat(new TestAnalyzer().analyze(unrelated)).isNull();
}
```

### Unit tests — FailureAnalyzers orchestrator

```java
@Test
void shouldReportTrue_WhenAnalyzerRecognizesFailure() {
    FailureAnalyzer matchingAnalyzer = failure ->
            new FailureAnalysis("matched", "fix it", failure);
    FailureAnalyzers analyzers = new FailureAnalyzers(List.of(matchingAnalyzer));
    assertThat(analyzers.reportException(new RuntimeException())).isTrue();
}

@Test
void shouldSkipBrokenAnalyzer_WhenItThrowsException() {
    FailureAnalyzer broken = failure -> { throw new RuntimeException("broke"); };
    FailureAnalyzer working = failure -> new FailureAnalysis("works", "action", failure);
    FailureAnalyzers analyzers = new FailureAnalyzers(List.of(broken, working));
    assertThat(analyzers.reportException(new RuntimeException())).isTrue();
}
```

### Unit tests — concrete analyzers

```java
@Test
void shouldAnalyze_WhenCircularDependencyDetected() {
    BeanCurrentlyInCreationException cycle = new BeanCurrentlyInCreationException("serviceA");
    BeanCreationException inner = new BeanCreationException("serviceB", "Unsatisfied", cycle);
    BeanCreationException outer = new BeanCreationException("serviceA", "Unsatisfied", inner);

    FailureAnalysis analysis = new BeanCurrentlyInCreationFailureAnalyzer().analyze(outer);
    assertThat(analysis.description()).contains("form a cycle");
    assertThat(analysis.description()).contains("serviceA");
}

@Test
void shouldAnalyze_WhenPortInUseExceptionDetected() {
    FailureAnalysis analysis = new PortInUseFailureAnalyzer()
            .analyze(new PortInUseException(8080));
    assertThat(analysis.description()).contains("Port 8080");
    assertThat(analysis.action()).contains("another port");
}
```

### Integration test — circular dependency in DefaultBeanFactory

```java
@Test
void shouldThrowBeanCurrentlyInCreationException_WhenCircularDependencyExists() {
    DefaultBeanFactory factory = new DefaultBeanFactory();
    factory.registerBeanDefinition("alpha", new BeanDefinition(AlphaService.class));
    factory.registerBeanDefinition("beta", new BeanDefinition(BetaService.class));

    assertThatThrownBy(() -> factory.getBean("alpha"))
            .isInstanceOf(BeanCreationException.class);
    // Prior to Feature 18, this would have been a StackOverflowError
}
```

Run the tests:
```bash
./gradlew test
```

---

## 18.7 Why This Works

`★ Insight ─────────────────────────────────────`
**Chain of Responsibility pattern:** Each `FailureAnalyzer` gets a chance to inspect the exception. The first one that returns a non-null `FailureAnalysis` wins. This is why `AbstractFailureAnalyzer` returns `null` when the cause type isn't in the chain — it's a "pass" signal. A broken analyzer (one that throws) is silently skipped. The system is **additive** — adding a new analyzer never breaks existing ones, and the order only matters when two analyzers could match the same exception.

**Cause chain walking in AbstractFailureAnalyzer:** Java exceptions form a linked list via `getCause()`. A single startup failure might be wrapped 5+ times: `IllegalStateException` → `BeanCreationException("serviceA")` → `BeanCreationException("serviceB")` → `BeanCurrentlyInCreationException("serviceA")`. The `findCause()` method walks this chain to find the specific exception type each analyzer handles. This is why the generic type parameter matters — it determines WHERE in the chain to look.

**Circular dependency detection via Set.add():** The `beansCurrentlyInCreation` set uses `Set.add()`'s boolean return value: `true` if the element was added (first time), `false` if already present (cycle!). This is a single-instruction check-and-mark — no separate "contains" + "add" — which is both cleaner and thread-safe with the `ConcurrentHashMap` backing. The real Spring Boot uses the same pattern in `DefaultSingletonBeanRegistry.beforeSingletonCreation()`.
`─────────────────────────────────────────────────`

---

## 18.8 What We Enhanced

| Component | Before (Feature 8/1) | After (Feature 18) | Why |
|-----------|---------------------|---------------------|-----|
| `IrisApplication.handleRunFailure()` | Only fires `ApplicationFailedEvent` and closes context | Also runs `FailureAnalyzers` to produce human-readable error messages | Users see actionable messages instead of raw stack traces |
| `DefaultBeanFactory.doGetBean()` | No circular dependency detection — `StackOverflowError` on cycles | Tracks `beansCurrentlyInCreation`, throws `BeanCurrentlyInCreationException` | Recognizable exception that `BeanCurrentlyInCreationFailureAnalyzer` can analyze |
| `TomcatWebServer.start()` | Catches `LifecycleException`, wraps in `WebServerException` | Detects `java.net.BindException` in cause chain, throws `PortInUseException` | Recognizable exception that `PortInUseFailureAnalyzer` can analyze |
| `ConfigurationPropertiesBinder.bindSimpleField()` | Raw `RuntimeException` on conversion failure | Wraps in `BindException` with property name and target class | Recognizable exception that `BindFailureAnalyzer` can analyze |

---

## 18.9 Connection to Real Spring Boot

| Iris Class | Spring Boot Class | File (commit `5922311a95a`) |
|-----------|-------------------|----------------------------|
| `FailureAnalysis` | `FailureAnalysis` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/FailureAnalysis.java` |
| `FailureAnalyzer` | `FailureAnalyzer` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/FailureAnalyzer.java` |
| `AbstractFailureAnalyzer` | `AbstractFailureAnalyzer` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/AbstractFailureAnalyzer.java` |
| `FailureAnalyzers` | `FailureAnalyzers` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/FailureAnalyzers.java` |
| `LoggingFailureAnalysisReporter` | `LoggingFailureAnalysisReporter` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/LoggingFailureAnalysisReporter.java` |
| `BeanCurrentlyInCreationFailureAnalyzer` | `BeanCurrentlyInCreationFailureAnalyzer` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/analyzer/BeanCurrentlyInCreationFailureAnalyzer.java` |
| `PortInUseFailureAnalyzer` | `PortInUseFailureAnalyzer` | `module/spring-boot-web-server/src/main/java/org/springframework/boot/web/server/PortInUseFailureAnalyzer.java` |
| `BindFailureAnalyzer` | `BindFailureAnalyzer` | `core/spring-boot/src/main/java/org/springframework/boot/diagnostics/analyzer/BindFailureAnalyzer.java` |
| `BeanCurrentlyInCreationException` | `BeanCurrentlyInCreationException` | `spring-framework: spring-beans/src/main/java/org/springframework/beans/factory/BeanCurrentlyInCreationException.java` |
| `PortInUseException` | `PortInUseException` | `module/spring-boot-web-server/src/main/java/org/springframework/boot/web/server/PortInUseException.java` |

**Simplifications from the real implementation:**

1. **No `SpringBootExceptionReporter` indirection:** In real Spring Boot, `FailureAnalyzers` implements `SpringBootExceptionReporter`, which is loaded via `spring.factories`. We call `FailureAnalyzers` directly from `handleRunFailure()`.

2. **No constructor injection into analyzers:** Real Spring Boot passes `BeanFactory` and `Environment` to analyzers that need them (e.g., `BeanCurrentlyInCreationFailureAnalyzer` takes a `BeanFactory` to check `isAllowCircularReferences()`). We use no-arg constructors.

3. **No `ResolvableType`:** The real `AbstractFailureAnalyzer.getCauseType()` uses `ResolvableType.forClass()` for robust generic resolution across complex class hierarchies. We inline the common case: direct subclass with one type parameter.

4. **Simpler cycle diagram:** Real Spring Boot's cycle diagram includes resource descriptions and injection point details. Ours shows bean names only.

---

## 18.10 Complete Code

### [NEW] `iris-framework/src/main/java/com/iris/framework/beans/factory/BeanCurrentlyInCreationException.java`

```java
package com.iris.framework.beans.factory;

/**
 * Thrown when a bean is requested while it is still being created,
 * indicating a circular dependency.
 *
 * <p>In the real Spring Framework, this extends {@code BeanCreationException}
 * and is thrown by {@code DefaultSingletonBeanRegistry.beforeSingletonCreation()}
 * when a bean name already exists in the "currently in creation" set.
 *
 * <p>The exception message includes the bean name involved in the cycle.
 * The full cycle can be reconstructed by walking the cause chain — each
 * nested {@code BeanCreationException} names the bean that triggered the
 * next lookup.
 *
 * @see org.springframework.beans.factory.BeanCurrentlyInCreationException
 */
public class BeanCurrentlyInCreationException extends BeanCreationException {

    public BeanCurrentlyInCreationException(String beanName) {
        super(beanName, "Requested bean is currently in creation: "
                + "Is there an unresolvable circular reference?");
    }
}
```

### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/beans/factory/support/DefaultBeanFactory.java`

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

public class DefaultBeanFactory implements BeanFactory, BeanDefinitionRegistry {

    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
    private final List<String> beanDefinitionNames = new ArrayList<>(256);
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();

    /**
     * Names of beans that are currently in the creation process.
     * Detects circular dependencies: if doGetBean() is called for a bean
     * that's already in this set, a circular reference exists.
     */
    private final Set<String> beansCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>());

    // ... (BeanPostProcessor registration, BeanDefinitionRegistry, BeanFactory
    //      implementations unchanged from ch01-ch06) ...

    private Object doGetBean(String name) {
        // 1. Check singleton cache
        Object singleton = this.singletonObjects.get(name);
        if (singleton != null) {
            return singleton;
        }

        // 2. Get the definition
        BeanDefinition bd = getBeanDefinition(name);

        // 3. Circular dependency check
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

    // ... (rest of DefaultBeanFactory unchanged) ...
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/FailureAnalysis.java`

```java
package com.iris.boot.diagnostics;

public record FailureAnalysis(String description, String action, Throwable cause) {
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/FailureAnalyzer.java`

```java
package com.iris.boot.diagnostics;

@FunctionalInterface
public interface FailureAnalyzer {
    FailureAnalysis analyze(Throwable failure);
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/AbstractFailureAnalyzer.java`

```java
package com.iris.boot.diagnostics;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

public abstract class AbstractFailureAnalyzer<T extends Throwable> implements FailureAnalyzer {

    @Override
    public FailureAnalysis analyze(Throwable failure) {
        T cause = findCause(failure, getCauseType());
        if (cause != null) {
            return analyze(failure, cause);
        }
        return null;
    }

    protected abstract FailureAnalysis analyze(Throwable rootFailure, T cause);

    @SuppressWarnings("unchecked")
    protected Class<T> getCauseType() {
        Type superclass = getClass().getGenericSuperclass();
        while (superclass != null) {
            if (superclass instanceof ParameterizedType pt) {
                Type raw = pt.getRawType();
                if (raw == AbstractFailureAnalyzer.class) {
                    Type arg = pt.getActualTypeArguments()[0];
                    if (arg instanceof Class<?>) {
                        return (Class<T>) arg;
                    }
                }
            }
            if (superclass instanceof Class<?> cls) {
                superclass = cls.getGenericSuperclass();
            } else {
                break;
            }
        }
        throw new IllegalStateException(
                "Unable to resolve cause type for " + getClass().getName());
    }

    @SuppressWarnings("unchecked")
    protected <E extends Throwable> E findCause(Throwable failure, Class<E> type) {
        Throwable candidate = failure;
        while (candidate != null) {
            if (type.isInstance(candidate)) {
                return (E) candidate;
            }
            candidate = candidate.getCause();
        }
        return null;
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/FailureAnalysisReporter.java`

```java
package com.iris.boot.diagnostics;

@FunctionalInterface
public interface FailureAnalysisReporter {
    void report(FailureAnalysis analysis);
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/LoggingFailureAnalysisReporter.java`

```java
package com.iris.boot.diagnostics;

import java.util.logging.Level;
import java.util.logging.Logger;

public class LoggingFailureAnalysisReporter implements FailureAnalysisReporter {

    private static final Logger logger = Logger.getLogger(LoggingFailureAnalysisReporter.class.getName());

    @Override
    public void report(FailureAnalysis analysis) {
        if (logger.isLoggable(Level.FINE)) {
            logger.log(Level.FINE, "Application startup failed due to an exception",
                    analysis.cause());
        }
        String message = buildMessage(analysis);
        System.err.println(message);
    }

    private String buildMessage(FailureAnalysis analysis) {
        StringBuilder sb = new StringBuilder();
        sb.append("\n***************************\n");
        sb.append("APPLICATION FAILED TO START\n");
        sb.append("***************************\n\n");
        sb.append("Description:\n\n");
        sb.append(analysis.description());
        sb.append("\n");
        if (analysis.action() != null) {
            sb.append("\nAction:\n\n");
            sb.append(analysis.action());
            sb.append("\n");
        }
        return sb.toString();
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/FailureAnalyzers.java`

```java
package com.iris.boot.diagnostics;

import java.util.ArrayList;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

import com.iris.boot.context.annotation.ImportCandidates;

public class FailureAnalyzers {

    private static final Logger logger = Logger.getLogger(FailureAnalyzers.class.getName());

    private final List<FailureAnalyzer> analyzers;

    public FailureAnalyzers(ClassLoader classLoader) {
        this.analyzers = loadFailureAnalyzers(classLoader);
    }

    FailureAnalyzers(List<FailureAnalyzer> analyzers) {
        this.analyzers = analyzers;
    }

    public boolean reportException(Throwable failure) {
        FailureAnalysis analysis = analyze(failure);
        if (analysis != null) {
            new LoggingFailureAnalysisReporter().report(analysis);
            return true;
        }
        return false;
    }

    private FailureAnalysis analyze(Throwable failure) {
        for (FailureAnalyzer analyzer : this.analyzers) {
            try {
                FailureAnalysis analysis = analyzer.analyze(failure);
                if (analysis != null) {
                    return analysis;
                }
            } catch (Throwable ex) {
                logger.log(Level.FINE,
                        "FailureAnalyzer " + analyzer.getClass().getName() + " failed", ex);
            }
        }
        return null;
    }

    private static List<FailureAnalyzer> loadFailureAnalyzers(ClassLoader classLoader) {
        ImportCandidates candidates = ImportCandidates.load(FailureAnalyzer.class, classLoader);
        List<FailureAnalyzer> analyzers = new ArrayList<>();
        for (String className : candidates.getCandidates()) {
            try {
                Class<?> clazz = Class.forName(className, true, classLoader);
                Object instance = clazz.getDeclaredConstructor().newInstance();
                if (instance instanceof FailureAnalyzer fa) {
                    analyzers.add(fa);
                }
            } catch (Throwable ex) {
                logger.log(Level.FINE,
                        "Unable to instantiate FailureAnalyzer: " + className, ex);
            }
        }
        return analyzers;
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/analyzer/BeanCurrentlyInCreationFailureAnalyzer.java`

```java
package com.iris.boot.diagnostics.analyzer;

import java.util.ArrayList;
import java.util.List;

import com.iris.boot.diagnostics.AbstractFailureAnalyzer;
import com.iris.boot.diagnostics.FailureAnalysis;
import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.BeanCurrentlyInCreationException;

public class BeanCurrentlyInCreationFailureAnalyzer
        extends AbstractFailureAnalyzer<BeanCurrentlyInCreationException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure,
                                       BeanCurrentlyInCreationException cause) {
        List<String> cycle = findCycle(rootFailure);
        String description = buildDescription(cycle);
        String action = "Redesign your beans to remove the circular dependency. "
                + "Consider using setter injection or extracting a common "
                + "dependency into a separate bean.";
        return new FailureAnalysis(description, action, cause);
    }

    private List<String> findCycle(Throwable rootFailure) {
        List<String> beanNames = new ArrayList<>();
        Throwable current = rootFailure;
        while (current != null) {
            if (current instanceof BeanCreationException bce) {
                String name = bce.getBeanName();
                if (name != null && !beanNames.contains(name)) {
                    beanNames.add(name);
                } else if (name != null) {
                    break;
                }
            }
            current = current.getCause();
        }
        return beanNames;
    }

    private String buildDescription(List<String> cycle) {
        StringBuilder sb = new StringBuilder();
        sb.append("The dependencies of some of the beans in the application context "
                + "form a cycle:\n\n");
        if (cycle.isEmpty()) {
            sb.append("   (cycle details unavailable)");
            return sb.toString();
        }
        sb.append("   \u250c\u2500\u2500\u2500\u2500\u2500\u2510\n");
        for (int i = 0; i < cycle.size(); i++) {
            sb.append("   |  ").append(cycle.get(i)).append("\n");
            if (i < cycle.size() - 1) {
                sb.append("   \u2193\n");
            }
        }
        sb.append("   \u2514\u2500\u2500\u2500\u2500\u2500\u2518");
        return sb.toString();
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/analyzer/PortInUseFailureAnalyzer.java`

```java
package com.iris.boot.diagnostics.analyzer;

import com.iris.boot.diagnostics.AbstractFailureAnalyzer;
import com.iris.boot.diagnostics.FailureAnalysis;
import com.iris.boot.web.server.PortInUseException;

public class PortInUseFailureAnalyzer extends AbstractFailureAnalyzer<PortInUseException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, PortInUseException cause) {
        return new FailureAnalysis(
                "Web server failed to start. Port " + cause.getPort()
                        + " was already in use.",
                "Identify and stop the process that's listening on port "
                        + cause.getPort()
                        + " or configure this application to listen on another port.",
                cause);
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/diagnostics/analyzer/BindFailureAnalyzer.java`

```java
package com.iris.boot.diagnostics.analyzer;

import com.iris.boot.context.properties.BindException;
import com.iris.boot.diagnostics.AbstractFailureAnalyzer;
import com.iris.boot.diagnostics.FailureAnalysis;

public class BindFailureAnalyzer extends AbstractFailureAnalyzer<BindException> {

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, BindException cause) {
        String description = "Failed to bind properties under '"
                + cause.getPropertyName() + "' to " + cause.getTargetClass().getName()
                + ":\n\n    Reason: " + getRootMessage(cause);
        String action = "Update your application's configuration.\n"
                + "Check the property value for '" + cause.getPropertyName()
                + "' and ensure it is compatible with the target type.";
        return new FailureAnalysis(description, action, cause);
    }

    private String getRootMessage(Throwable ex) {
        Throwable root = ex;
        while (root.getCause() != null) {
            root = root.getCause();
        }
        return root.getMessage();
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/web/server/PortInUseException.java`

```java
package com.iris.boot.web.server;

public class PortInUseException extends WebServerException {

    private final int port;

    public PortInUseException(int port) {
        super("Port " + port + " is already in use");
        this.port = port;
    }

    public PortInUseException(int port, Throwable cause) {
        super("Port " + port + " is already in use", cause);
        this.port = port;
    }

    public int getPort() {
        return this.port;
    }
}
```

### [NEW] `iris-boot-core/src/main/java/com/iris/boot/context/properties/BindException.java`

```java
package com.iris.boot.context.properties;

public class BindException extends RuntimeException {

    private final String propertyName;
    private final Class<?> targetClass;

    public BindException(String propertyName, Class<?> targetClass, String message) {
        super("Failed to bind property '" + propertyName + "' to "
                + targetClass.getSimpleName() + ": " + message);
        this.propertyName = propertyName;
        this.targetClass = targetClass;
    }

    public BindException(String propertyName, Class<?> targetClass, String message,
                         Throwable cause) {
        super("Failed to bind property '" + propertyName + "' to "
                + targetClass.getSimpleName() + ": " + message, cause);
        this.propertyName = propertyName;
        this.targetClass = targetClass;
    }

    public String getPropertyName() { return this.propertyName; }
    public Class<?> getTargetClass() { return this.targetClass; }
}
```

### [NEW] `iris-boot-core/src/main/resources/META-INF/iris/com.iris.boot.diagnostics.FailureAnalyzer.imports`

```
# Failure Analyzers — ordered from most specific to most general
com.iris.boot.diagnostics.analyzer.BeanCurrentlyInCreationFailureAnalyzer
com.iris.boot.diagnostics.analyzer.PortInUseFailureAnalyzer
com.iris.boot.diagnostics.analyzer.BindFailureAnalyzer
```

### [MODIFIED] `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java` (diff only)

```diff
+ import com.iris.boot.diagnostics.FailureAnalyzers;

  private void handleRunFailure(...) {
      try {
          listeners.failed(context, exception);
      } catch (Exception ex) {
          System.err.println("Failed to publish ApplicationFailedEvent: " + ex.getMessage());
      }
+     reportFailure(exception);
      if (context != null) {
          context.close();
      }
  }

+ private void reportFailure(Throwable exception) {
+     try {
+         FailureAnalyzers analyzers = new FailureAnalyzers(getClassLoader());
+         analyzers.reportException(exception);
+     } catch (Throwable ex) {
+         System.err.println("Unable to report failure: " + ex.getMessage());
+     }
+ }

+ private ClassLoader getClassLoader() {
+     return this.primarySource != null
+             ? this.primarySource.getClassLoader()
+             : Thread.currentThread().getContextClassLoader();
+ }
```

### [MODIFIED] `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatWebServer.java` (diff only)

```diff
+ import com.iris.boot.web.server.PortInUseException;

  } catch (LifecycleException ex) {
      this.started.set(false);
+     Connector connector = this.tomcat.getConnector();
+     if (connector != null && isPortInUse(ex)) {
+         throw new PortInUseException(connector.getPort(), ex);
+     }
      throw new WebServerException("Unable to start embedded Tomcat", ex);
  }

+ private boolean isPortInUse(Throwable ex) {
+     Throwable candidate = ex;
+     while (candidate != null) {
+         if (candidate instanceof java.net.BindException) {
+             return true;
+         }
+         candidate = candidate.getCause();
+     }
+     return false;
+ }
```

### [MODIFIED] `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBinder.java` (diff only)

```diff
  private void bindSimpleField(...) {
      String value = resolveProperty(prefix, fieldName);
      if (value != null) {
+         try {
              Object converted = PropertySourcesPropertyResolver.convertValue(value, fieldType);
              setFieldValue(target, field, converted);
+         } catch (Exception ex) {
+             String propertyName = prefix + "." + fieldName;
+             throw new BindException(propertyName, target.getClass(),
+                     "failed to convert value '" + value + "' to type "
+                             + fieldType.getSimpleName(), ex);
+         }
      }
  }
```

### Test files

See `iris-boot-core/src/test/java/com/iris/boot/diagnostics/` for all test files:
- `FailureAnalysisTest.java` — record behavior
- `AbstractFailureAnalyzerTest.java` — cause chain walking, generic type resolution
- `FailureAnalyzersTest.java` — orchestrator, SPI loading
- `LoggingFailureAnalysisReporterTest.java` — banner formatting
- `analyzer/BeanCurrentlyInCreationFailureAnalyzerTest.java` — cycle detection
- `analyzer/PortInUseFailureAnalyzerTest.java` — port conflict
- `analyzer/BindFailureAnalyzerTest.java` — binding errors
- `integration/FailureAnalyzersIntegrationTest.java` — end-to-end with IrisApplication

And `iris-framework/src/test/java/.../CircularDependencyDetectionTest.java` — DefaultBeanFactory circular dependency detection.

---

## Summary

| What | Detail |
|------|--------|
| **New classes** | `FailureAnalysis`, `FailureAnalyzer`, `AbstractFailureAnalyzer`, `FailureAnalyzers`, `FailureAnalysisReporter`, `LoggingFailureAnalysisReporter`, `BeanCurrentlyInCreationFailureAnalyzer`, `PortInUseFailureAnalyzer`, `BindFailureAnalyzer`, `BeanCurrentlyInCreationException`, `PortInUseException`, `BindException` |
| **Modified classes** | `IrisApplication` (+`reportFailure`), `DefaultBeanFactory` (+circular detection), `TomcatWebServer` (+port-in-use detection), `ConfigurationPropertiesBinder` (+bind exception) |
| **Pattern** | Chain of Responsibility — each analyzer inspects the exception chain, first match wins |
| **SPI** | `META-INF/iris/com.iris.boot.diagnostics.FailureAnalyzer.imports` |
| **Tests** | 8 test files, covering unit, integration, and SPI loading |

**Next chapter:** [Chapter 19: Graceful Shutdown](ch19_graceful_shutdown.md) — drain active HTTP requests before stopping the server via `SmartLifecycle` and `WebServerGracefulShutdownLifecycle`.
