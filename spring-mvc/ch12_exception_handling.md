# Chapter 12: Exception Handling

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Handler methods can throw exceptions, but any exception causes a raw 500 error from the servlet container with a Tomcat HTML error page | No way for controllers to define structured error responses or for the application to centralize error handling logic | Build an `@ExceptionHandler` + `@ControllerAdvice` system that catches handler exceptions and routes them to dedicated error-handling methods that produce clean JSON/text error responses |

---

## 12.1 The Integration Point

The integration point is `SimpleDispatcherServlet.doDispatch()` — specifically the gap between catching the `dispatchException` (line 270) and re-throwing it (previously line 232). Currently, any exception from a handler method bubbles up as a `ServletException` to Tomcat, which renders its default HTML error page.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Add `processHandlerException()` call between the catch and re-throw, plus a new `exceptionResolvers` field and `initExceptionResolvers()` during startup.

```java
// NEW: Exception resolvers field
private final List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<>();

// In initStrategies():
initExceptionResolvers();  // NEW — scans for @ControllerAdvice beans

// In doDispatch() — replaces the old "if (dispatchException != null) { throw dispatchException; }"
if (dispatchException != null) {
    Object handler = (mappedHandler != null) ? mappedHandler.getHandler() : null;
    if (!processHandlerException(request, response, handler, dispatchException)) {
        throw dispatchException;  // No resolver handled it — re-throw to container
    }
}

// NEW method:
private boolean processHandlerException(HttpServletRequest request, HttpServletResponse response,
                                        Object handler, Exception ex) {
    for (HandlerExceptionResolver resolver : exceptionResolvers) {
        if (resolver.resolveException(request, response, handler, ex)) {
            return true;  // Exception consumed — response already written
        }
    }
    return false;  // No resolver handled it
}
```

Two key decisions here:

1. **Chain of Responsibility over single resolver** — the real framework supports multiple `HandlerExceptionResolver` beans (ExceptionHandler, ResponseStatus, DefaultHandler). We iterate a list even though we only have one resolver now, keeping the extension point open.
2. **Resolve AFTER afterCompletion** — interceptors still get their cleanup callbacks even when the exception is resolved. The real framework does the same: interceptor lifecycle is independent of exception handling.

This connects the **dispatch pipeline** to the **exception resolution subsystem**. To make it work, we need to build:
- `HandlerExceptionResolver` — the strategy interface (Chain of Responsibility)
- `ExceptionHandlerMethodResolver` — discovers and matches `@ExceptionHandler` methods within a single class
- `ExceptionHandlerExceptionResolver` — the concrete resolver with two-phase lookup (local controller → global `@ControllerAdvice`)
- `@ExceptionHandler` and `@ControllerAdvice` annotations

## 12.2 The Strategy Interface: `HandlerExceptionResolver`

**New file:** `src/main/java/com/simplespringmvc/exception/HandlerExceptionResolver.java`

```java
package com.simplespringmvc.exception;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public interface HandlerExceptionResolver {

    /**
     * Try to resolve the given exception.
     * Returns true if handled (response written), false to pass to next resolver.
     *
     * Real version returns ModelAndView instead of boolean:
     *   - non-null ModelAndView = handled (render view or empty = response already written)
     *   - null = not handled, try next resolver
     * We simplify to boolean since we don't have ModelAndView yet (ch13).
     */
    boolean resolveException(HttpServletRequest request, HttpServletResponse response,
                             Object handler, Exception ex);
}
```

## 12.3 The Annotations: `@ExceptionHandler` and `@ControllerAdvice`

**New file:** `src/main/java/com/simplespringmvc/annotation/ExceptionHandler.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {
    /**
     * Exception types handled by this method.
     * If empty, inferred from Throwable parameter types.
     */
    Class<? extends Throwable>[] value() default {};
}
```

**New file:** `src/main/java/com/simplespringmvc/annotation/ControllerAdvice.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ControllerAdvice {
}
```

## 12.4 The Matching Engine: `ExceptionHandlerMethodResolver`

This class is responsible for two jobs within a **single class**: (1) discover all `@ExceptionHandler` methods at construction time, and (2) match thrown exceptions to the best handler at runtime.

**New file:** `src/main/java/com/simplespringmvc/exception/ExceptionHandlerMethodResolver.java`

```java
package com.simplespringmvc.exception;

import com.simplespringmvc.annotation.ExceptionHandler;
import java.lang.reflect.Method;
import java.util.*;

public class ExceptionHandlerMethodResolver {

    private final Map<Class<? extends Throwable>, Method> mappedMethods = new HashMap<>();

    public ExceptionHandlerMethodResolver(Class<?> beanType) {
        for (Method method : beanType.getMethods()) {
            ExceptionHandler annotation = method.getAnnotation(ExceptionHandler.class);
            if (annotation != null) {
                List<Class<? extends Throwable>> types = detectExceptionTypes(annotation, method);
                for (Class<? extends Throwable> type : types) {
                    mappedMethods.put(type, method);
                }
            }
        }
    }

    public Method resolveMethod(Throwable exception) {
        // Try the exception itself
        Method method = resolveByExceptionType(exception.getClass());
        if (method != null) return method;

        // Walk the cause chain
        Throwable cause = exception.getCause();
        while (cause != null) {
            method = resolveByExceptionType(cause.getClass());
            if (method != null) return method;
            cause = cause.getCause();
        }
        return null;
    }

    private Method resolveByExceptionType(Class<? extends Throwable> exceptionType) {
        Method bestMatch = null;
        int bestDepth = Integer.MAX_VALUE;
        for (var entry : mappedMethods.entrySet()) {
            if (entry.getKey().isAssignableFrom(exceptionType)) {
                int depth = getDepth(exceptionType, entry.getKey());
                if (depth < bestDepth) {
                    bestDepth = depth;
                    bestMatch = entry.getValue();
                }
            }
        }
        return bestMatch;
    }

    private int getDepth(Class<?> exceptionType, Class<?> declaredType) {
        int depth = 0;
        Class<?> current = exceptionType;
        while (current != null) {
            if (current == declaredType) return depth;
            current = current.getSuperclass();
            depth++;
        }
        return Integer.MAX_VALUE;
    }

    public boolean hasExceptionMappings() { return !mappedMethods.isEmpty(); }
}
```

The **exception depth** algorithm is the key insight. When multiple `@ExceptionHandler` methods could match (e.g., one for `RuntimeException`, another for `Exception`), the closest match in the class hierarchy wins. This mirrors `ExceptionDepthComparator` in the real framework.

## 12.5 The Orchestrator: `ExceptionHandlerExceptionResolver`

This is the class that implements the **two-phase lookup** — the core design of Spring's exception handling:

**New file:** `src/main/java/com/simplespringmvc/exception/ExceptionHandlerExceptionResolver.java`

```java
package com.simplespringmvc.exception;

import com.simplespringmvc.adapter.*;
import com.simplespringmvc.annotation.ControllerAdvice;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.converter.*;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.lang.reflect.Method;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class ExceptionHandlerExceptionResolver implements HandlerExceptionResolver {

    // Phase 1 cache: controller class → resolver (lazily populated)
    private final Map<Class<?>, ExceptionHandlerMethodResolver> exceptionHandlerCache =
            new ConcurrentHashMap<>(64);

    // Phase 2 cache: @ControllerAdvice bean → resolver (eagerly populated at init)
    private final Map<Object, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache =
            new LinkedHashMap<>();

    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();

    public void init(BeanContainer beanContainer) {
        // Scan for @ControllerAdvice beans
        for (String name : beanContainer.getBeanNames()) {
            Object bean = beanContainer.getBean(name);
            if (bean.getClass().isAnnotationPresent(ControllerAdvice.class)) {
                ExceptionHandlerMethodResolver resolver =
                        new ExceptionHandlerMethodResolver(bean.getClass());
                if (resolver.hasExceptionMappings()) {
                    exceptionHandlerAdviceCache.put(bean, resolver);
                }
            }
        }
        // Set up return value handlers (same chain as SimpleHandlerAdapter)
        List<HttpMessageConverter> converters = List.of(new JacksonMessageConverter());
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(converters));
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    @Override
    public boolean resolveException(HttpServletRequest request, HttpServletResponse response,
                                    Object handler, Exception ex) {
        HandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handler, ex);
        if (exceptionHandlerMethod == null) return false;

        try {
            if (!response.isCommitted()) response.reset();
            Method method = exceptionHandlerMethod.getMethod();
            Object[] args = buildExceptionHandlerArgs(method, ex, request, response);
            method.setAccessible(true);
            Object result = method.invoke(exceptionHandlerMethod.getBean(), args);
            returnValueHandlers.handleReturnValue(result, exceptionHandlerMethod, request, response);
            return true;
        } catch (Exception resolverEx) {
            return false;
        }
    }

    /**
     * TWO-PHASE LOOKUP — the heart of exception handling:
     * Phase 1: Check the controller that threw (local handlers)
     * Phase 2: Check @ControllerAdvice beans (global handlers)
     */
    private HandlerMethod getExceptionHandlerMethod(Object handler, Exception ex) {
        // Phase 1: Local
        if (handler instanceof HandlerMethod handlerMethod) {
            ExceptionHandlerMethodResolver resolver = exceptionHandlerCache.computeIfAbsent(
                    handlerMethod.getBeanType(), ExceptionHandlerMethodResolver::new);
            Method method = resolver.resolveMethod(ex);
            if (method != null) return new HandlerMethod(handlerMethod.getBean(), method);
        }
        // Phase 2: Global @ControllerAdvice
        for (var entry : exceptionHandlerAdviceCache.entrySet()) {
            Method method = entry.getValue().resolveMethod(ex);
            if (method != null) return new HandlerMethod(entry.getKey(), method);
        }
        return null;
    }

    private Object[] buildExceptionHandlerArgs(Method method, Exception ex,
                                                HttpServletRequest request,
                                                HttpServletResponse response) {
        Class<?>[] paramTypes = method.getParameterTypes();
        Object[] args = new Object[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++) {
            if (Throwable.class.isAssignableFrom(paramTypes[i]))
                args[i] = findMatchingException(ex, paramTypes[i]);
            else if (HttpServletRequest.class.isAssignableFrom(paramTypes[i]))
                args[i] = request;
            else if (HttpServletResponse.class.isAssignableFrom(paramTypes[i]))
                args[i] = response;
        }
        return args;
    }

    private Throwable findMatchingException(Throwable ex, Class<?> declaredType) {
        Throwable current = ex;
        while (current != null) {
            if (declaredType.isInstance(current)) return current;
            current = current.getCause();
        }
        return ex;
    }
}
```

## 12.6 Try It Yourself

<details>
<summary>Challenge 1: Implement exception depth matching</summary>

Given this controller:
```java
@Controller
class MyController {
    @ExceptionHandler(RuntimeException.class)
    @ResponseBody
    public String handleRuntime(RuntimeException ex) { return "runtime"; }

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseBody
    public String handleBadArg(IllegalArgumentException ex) { return "bad arg"; }
}
```

When a `NumberFormatException` (extends `IllegalArgumentException` extends `RuntimeException`) is thrown, which handler should be selected and why?

**Answer:** `handleBadArg` — because `IllegalArgumentException` is at depth 1 from `NumberFormatException`, while `RuntimeException` is at depth 2. The closest match in the exception hierarchy always wins.

The depth calculation walks up the class hierarchy:
```java
private int getDepth(Class<?> exceptionType, Class<?> declaredType) {
    int depth = 0;
    Class<?> current = exceptionType;
    while (current != null) {
        if (current == declaredType) return depth;
        current = current.getSuperclass();
        depth++;
    }
    return Integer.MAX_VALUE;
}
```

</details>

<details>
<summary>Challenge 2: Implement the two-phase lookup</summary>

Given:
- `ProductController` with a local `@ExceptionHandler(IllegalArgumentException.class)`
- `@ControllerAdvice GlobalAdvice` with `@ExceptionHandler(RuntimeException.class)`

When `ProductController` throws an `IllegalArgumentException`, which handler runs?
When `ProductController` throws an `IllegalStateException`, which handler runs?

**Answer:**
1. `IllegalArgumentException` → **local handler** wins (Phase 1 match, never reaches Phase 2)
2. `IllegalStateException` → Phase 1 finds no match (local only handles `IllegalArgumentException`), so Phase 2 kicks in → **global handler** (`IllegalStateException` IS a `RuntimeException`)

```java
private HandlerMethod getExceptionHandlerMethod(Object handler, Exception ex) {
    // Phase 1: Local — always checked first
    if (handler instanceof HandlerMethod handlerMethod) {
        ExceptionHandlerMethodResolver resolver = exceptionHandlerCache.computeIfAbsent(
                handlerMethod.getBeanType(), ExceptionHandlerMethodResolver::new);
        Method method = resolver.resolveMethod(ex);
        if (method != null) return new HandlerMethod(handlerMethod.getBean(), method);
    }
    // Phase 2: Global — only if Phase 1 had no match
    for (var entry : exceptionHandlerAdviceCache.entrySet()) {
        Method method = entry.getValue().resolveMethod(ex);
        if (method != null) return new HandlerMethod(entry.getKey(), method);
    }
    return null;
}
```

</details>

## 12.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/exception/ExceptionHandlerMethodResolverTest.java`

```java
// Key test methods:
shouldDiscoverExplicitExceptionType()        — @ExceptionHandler(IllegalArgumentException.class)
shouldDiscoverInferredExceptionType()        — @ExceptionHandler on method(RuntimeException ex)
shouldResolveExactMatch()                    — exact type match returns correct handler
shouldResolveSubclass_WhenExactMatchNotAvailable() — NumberFormatException matches IllegalArgumentException handler
shouldResolveClosestMatch_WhenMultipleHandlersApply() — depth 0 beats depth 1
shouldWalkCauseChain_WhenTopLevelDoesNotMatch()    — RuntimeException wrapping IOException finds IO handler
shouldReturnNull_WhenCauseChainHasNoMatch()  — no match in chain returns null
```

**New file:** `src/test/java/com/simplespringmvc/exception/ExceptionHandlerExceptionResolverTest.java`

```java
// Key test methods:
shouldDiscoverControllerAdviceBeans()                    — init finds @ControllerAdvice classes
shouldResolveFromLocalController_WhenControllerHasHandler() — Phase 1 works
shouldFallbackToGlobalAdvice_WhenControllerHasNoHandler()  — Phase 2 works
shouldPreferLocalHandler_OverGlobalAdvice()               — Phase 1 priority over Phase 2
shouldReturnFalse_WhenNoHandlerMatches()                 — unresolvable returns false
shouldPassRequestAndResponse_WhenHandlerDeclaresThemAsParameters() — argument injection
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/ExceptionHandlingIntegrationTest.java`

```java
// Key test methods (full HTTP stack via EmbeddedTomcat):
shouldHandleNormalRequest_WhenNoExceptionThrown()        — exception handling doesn't break normal flow
shouldUseLocalExceptionHandler_WhenControllerThrows()    — local @ExceptionHandler produces JSON error
shouldUseGlobalAdvice_WhenControllerHasNoLocalHandler()  — @ControllerAdvice catches unhandled exceptions
shouldPreferLocalHandler_OverGlobalAdvice_WhenBothMatch() — local wins when both could match
shouldFallToGlobal_WhenLocalHandlerDoesNotMatchExceptionType() — type mismatch falls to global
shouldReturn500_WhenNoResolverHandlesException()         — unresolvable → Tomcat 500
shouldInitializeExceptionResolvers()                      — DispatcherServlet has resolvers after init
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 12.8 Why This Works

> ★ **Insight: Two-Phase Lookup is Intentional Layering** ─────────────
> - Controller-local `@ExceptionHandler` methods always win because they represent **domain-specific** error handling — the controller author knows best how to handle errors from their own endpoints.
> - `@ControllerAdvice` provides the **safety net** — global, application-wide error handling for anything not caught locally. This is the Exception Handling equivalent of CSS specificity: inline styles (local) beat stylesheets (global).
> - The real framework adds a third dimension: `@ControllerAdvice` supports filtering by `basePackages`, `assignableTypes`, and `annotations`, enabling middle-ground handlers (e.g., one for REST controllers, another for HTML controllers). We skip this to keep the two-phase design clear.
> ────────────────────────────────────────────────────────────────────

> ★ **Insight: Exception Depth as Best-Fit Resolution** ───────────────
> - When multiple `@ExceptionHandler` methods match, the framework picks the **closest type** in the hierarchy. This is the same principle as Java method overloading resolution — the most specific match wins.
> - The real framework uses `ExceptionDepthComparator` and also considers media type specificity (since 6.2). Our simplified depth calculation captures the essential algorithm: walk up from the thrown type counting steps.
> - **Trade-off:** This means a handler for `Exception.class` is a "catch-all" — it matches everything. While convenient, it can mask bugs if developers aren't careful. The real framework mitigates this with ordering (`@Order` on `@ControllerAdvice`) and the media type dimension.
> ────────────────────────────────────────────────────────────────────

> ★ **Insight: Return Value Handlers Reuse is Key** ───────────────────
> - Exception handler methods use the **same return value handler chain** as normal handler methods. This means `@ResponseBody` on an `@ExceptionHandler` method works exactly as expected — Jackson serializes the return value to JSON. No special error-response handling needed.
> - The real framework takes this further: `@ExceptionHandler` methods support the full argument resolver chain (Exception, Model, WebRequest, HttpServletRequest, etc.) and the full return value handler chain (ResponseEntity, ModelAndView, @ResponseBody, etc.). Our simplified version supports Exception + Request/Response parameters and the two return value handlers we already have.
> ────────────────────────────────────────────────────────────────────

## 12.9 What We Enhanced

| Aspect | Before (ch11) | Current (ch12) | Real Framework |
|--------|---------------|----------------|----------------|
| Exception handling | Exceptions re-thrown as `ServletException` → Tomcat HTML error page | `@ExceptionHandler` methods produce structured error responses; `@ControllerAdvice` provides global fallback | Full resolver chain with `ExceptionHandlerExceptionResolver`, `ResponseStatusExceptionResolver`, `DefaultHandlerExceptionResolver` — `DispatcherServlet.java:1207` |
| DispatcherServlet pipeline | `doDispatch()` catches exception then re-throws | `doDispatch()` catches exception, calls `processHandlerException()`, only re-throws if unresolved | `processHandlerException()` resets response buffer, iterates resolvers, handles `ModelAndView` rendering — `DispatcherServlet.java:1207` |
| Strategy initialization | `initStrategies()` initializes handler mapping + adapter | `initStrategies()` also initializes exception resolvers | `initHandlerExceptionResolvers()` finds beans or loads from `DispatcherServlet.properties` — `DispatcherServlet.java:608` |

## 12.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `HandlerExceptionResolver` (returns boolean) | `HandlerExceptionResolver` (returns `ModelAndView`) | `HandlerExceptionResolver.java:51` | Real version returns ModelAndView for view rendering; null means "not handled" |
| `ExceptionHandlerMethodResolver` | `ExceptionHandlerMethodResolver` | `ExceptionHandlerMethodResolver.java:93` | Real version uses `ExceptionMapping` record with media type, `ConcurrentLruCache`, and `ExceptionDepthComparator` |
| `ExceptionHandlerExceptionResolver.getExceptionHandlerMethod()` | `ExceptionHandlerExceptionResolver.getExceptionHandlerMethod()` | `ExceptionHandlerExceptionResolver.java:505` | Real version respects `@ControllerAdvice` filtering (basePackages, assignableTypes, annotations) and media type matching |
| `ExceptionHandlerExceptionResolver.resolveException()` | `ExceptionHandlerExceptionResolver.doResolveHandlerMethodException()` | `ExceptionHandlerExceptionResolver.java:426` | Real version goes through `AbstractHandlerExceptionResolver` → `AbstractHandlerMethodExceptionResolver` template method hierarchy |
| `@ControllerAdvice` (no filtering) | `@ControllerAdvice` (with filtering) | `ControllerAdvice.java:97` | Real version supports `basePackages`, `assignableTypes`, `annotations` for targeting specific controllers |
| `buildExceptionHandlerArgs()` (3 types) | Full argument resolver chain | `ExceptionHandlerExceptionResolver.java:275` | Real version uses configured `HandlerMethodArgumentResolver` list supporting Exception, Model, WebRequest, Locale, etc. |
| `SimpleDispatcherServlet.processHandlerException()` | `DispatcherServlet.processHandlerException()` | `DispatcherServlet.java:1207` | Real version resets `PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE`, calls `response.resetBuffer()`, handles `ModelAndView` rendering |

## 12.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/ExceptionHandler.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a method as an exception handler within a {@link Controller} or
 * {@link ControllerAdvice} class. When a handler method throws an exception
 * that matches one of the declared types, this method is invoked to produce
 * the error response.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.ExceptionHandler}
 *
 * <h3>Exception type matching:</h3>
 * <pre>
 *   1. Explicit: @ExceptionHandler(IllegalArgumentException.class)
 *      → matches IllegalArgumentException and all subclasses
 *   2. Inferred: @ExceptionHandler on method(SomeException ex)
 *      → exception type inferred from the Throwable parameter
 * </pre>
 *
 * <h3>Two-phase lookup:</h3>
 * <pre>
 *   Phase 1: Search the throwing controller's own @ExceptionHandler methods
 *   Phase 2: Search @ControllerAdvice beans (global handlers)
 *   → Controller-local handlers always take priority
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No {@code produces} attribute (no content negotiation — ch14)</li>
 *   <li>No support for multiple exception types in a single handler
 *       (use one @ExceptionHandler per type for simplicity)</li>
 * </ul>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {

    /**
     * The exception types handled by this method. If empty, the exception type
     * is inferred from the method's {@code Throwable} parameter type.
     *
     * Maps to: {@code ExceptionHandler.value()} / {@code ExceptionHandler.exception()}
     * (they are aliases of each other in real Spring)
     */
    Class<? extends Throwable>[] value() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/ControllerAdvice.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as a global exception handler (and future model-attribute /
 * init-binder provider) for all {@link Controller} classes.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.ControllerAdvice}
 *
 * In the real framework, {@code @ControllerAdvice} is meta-annotated with
 * {@code @Component}, so it's discovered by component scanning. It also
 * supports filtering by base packages, assignable types, and target annotations.
 *
 * <h3>How it works:</h3>
 * <pre>
 *   @ControllerAdvice
 *   public class GlobalExceptionHandler {
 *
 *       @ExceptionHandler(IllegalArgumentException.class)
 *       @ResponseBody
 *       public Map&lt;String, String&gt; handleBadRequest(IllegalArgumentException ex) {
 *           return Map.of("error", ex.getMessage());
 *       }
 *   }
 * </pre>
 *
 * The {@code ExceptionHandlerExceptionResolver} discovers these beans and
 * consults them after checking the controller's own {@code @ExceptionHandler}
 * methods — controller-local handlers always take priority.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No {@code @Component} meta-annotation (no component scanning — ch17)</li>
 *   <li>No filtering by basePackages, assignableTypes, or annotations</li>
 *   <li>No ordering via {@code @Order} / {@code Ordered}</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ControllerAdvice {
}
```

#### File: `src/main/java/com/simplespringmvc/exception/HandlerExceptionResolver.java` [NEW]

```java
package com.simplespringmvc.exception;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for resolving exceptions thrown during handler execution.
 *
 * Maps to: {@code org.springframework.web.servlet.HandlerExceptionResolver}
 *
 * The real interface returns {@code ModelAndView}: a non-null value means the
 * exception was handled (empty ModelAndView = response already written, non-empty
 * = render a view). A null return means "I can't handle this, try the next resolver."
 *
 * We simplify by returning a boolean: {@code true} = handled (response written),
 * {@code false} = not handled (pass to next resolver or re-throw). This avoids
 * introducing ModelAndView before ch13 while preserving the Chain of Responsibility
 * pattern.
 *
 * <h3>Chain of Responsibility:</h3>
 * <pre>
 *   DispatcherServlet.processHandlerException():
 *     for each resolver in handlerExceptionResolvers:
 *       if resolver.resolveException(req, resp, handler, ex):
 *         return  ← exception handled, stop iterating
 *     throw ex    ← no resolver handled it, re-throw
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Returns boolean instead of ModelAndView</li>
 *   <li>No AbstractHandlerExceptionResolver template method hierarchy</li>
 *   <li>No Ordered interface for resolver prioritization</li>
 * </ul>
 */
public interface HandlerExceptionResolver {

    /**
     * Try to resolve the given exception thrown during handler execution.
     *
     * Maps to: {@code HandlerExceptionResolver.resolveException()} (line 51)
     *
     * @param request  the current HTTP request
     * @param response the current HTTP response
     * @param handler  the handler that threw the exception (may be null)
     * @param ex       the exception that was thrown
     * @return {@code true} if the exception was handled and a response was written;
     *         {@code false} if this resolver cannot handle the exception
     */
    boolean resolveException(HttpServletRequest request, HttpServletResponse response,
                             Object handler, Exception ex);
}
```

#### File: `src/main/java/com/simplespringmvc/exception/ExceptionHandlerMethodResolver.java` [NEW]

```java
package com.simplespringmvc.exception;

import com.simplespringmvc.annotation.ExceptionHandler;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Discovers {@link ExceptionHandler @ExceptionHandler} methods within a single class
 * and matches thrown exceptions to the best handler at runtime.
 *
 * Maps to: {@code org.springframework.web.method.annotation.ExceptionHandlerMethodResolver}
 *
 * <h3>Discovery (at construction time):</h3>
 * <pre>
 *   For each method on the class:
 *     if annotated with @ExceptionHandler:
 *       1. Read exception types from @ExceptionHandler.value()
 *       2. If empty, infer from Throwable parameter types
 *       3. Register: exceptionType → method in the mappedMethods map
 * </pre>
 *
 * <h3>Matching (at runtime):</h3>
 * <pre>
 *   Given a thrown exception:
 *     1. Walk the class hierarchy of the exception (RuntimeException → Exception → Throwable)
 *     2. Check mappedMethods for each level
 *     3. If no direct match, walk the cause chain (ex.getCause())
 *     4. Return the closest match (shallowest depth in the type hierarchy)
 * </pre>
 *
 * The "closest match" algorithm uses exception depth — the number of levels between
 * the thrown exception type and the declared handler type. For example:
 * <pre>
 *   IllegalArgumentException extends RuntimeException extends Exception
 *   Depth for IllegalArgumentException:
 *     → IllegalArgumentException.class = depth 0 (exact match)
 *     → RuntimeException.class        = depth 1
 *     → Exception.class               = depth 2
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No media type matching on the exception handler</li>
 *   <li>No caching of resolved results (ConcurrentLruCache in real version)</li>
 *   <li>No cause-chain walking (real version walks ex.getCause() recursively)</li>
 *   <li>Simpler depth comparison without ExceptionDepthComparator class</li>
 * </ul>
 */
public class ExceptionHandlerMethodResolver {

    /**
     * Exception type → handler method mapping, populated at construction time.
     *
     * Maps to: {@code ExceptionHandlerMethodResolver.mappedMethods}
     * (LinkedHashMap in real version, keyed by ExceptionMapping record)
     */
    private final Map<Class<? extends Throwable>, Method> mappedMethods = new HashMap<>();

    /**
     * Discover all @ExceptionHandler methods in the given class.
     *
     * Maps to: {@code ExceptionHandlerMethodResolver(Class)} constructor (line 93)
     * Real version uses MethodIntrospector.selectMethods() with a MetadataLookup filter.
     *
     * @param beanType the controller or @ControllerAdvice class to scan
     */
    public ExceptionHandlerMethodResolver(Class<?> beanType) {
        for (Method method : beanType.getMethods()) {
            ExceptionHandler annotation = method.getAnnotation(ExceptionHandler.class);
            if (annotation != null) {
                List<Class<? extends Throwable>> exceptionTypes = detectExceptionTypes(annotation, method);
                for (Class<? extends Throwable> exType : exceptionTypes) {
                    mappedMethods.put(exType, method);
                }
            }
        }
    }

    /**
     * Determine the exception types this handler method should match.
     *
     * Maps to: {@code ExceptionHandlerMethodResolver.detectExceptionMappings()} (line 113)
     *
     * Two strategies:
     * 1. Explicit: @ExceptionHandler(IllegalArgumentException.class)
     * 2. Inferred: @ExceptionHandler on method(SomeException ex) → infer from parameter
     */
    @SuppressWarnings("unchecked")
    private List<Class<? extends Throwable>> detectExceptionTypes(ExceptionHandler annotation, Method method) {
        List<Class<? extends Throwable>> result = new ArrayList<>();

        // Strategy 1: Explicit types from annotation
        Class<? extends Throwable>[] declaredTypes = annotation.value();
        if (declaredTypes.length > 0) {
            for (Class<? extends Throwable> type : declaredTypes) {
                result.add(type);
            }
            return result;
        }

        // Strategy 2: Infer from Throwable parameter types
        for (Class<?> paramType : method.getParameterTypes()) {
            if (Throwable.class.isAssignableFrom(paramType)) {
                result.add((Class<? extends Throwable>) paramType);
            }
        }

        if (result.isEmpty()) {
            throw new IllegalStateException(
                    "@ExceptionHandler method " + method.getName() + " in " + method.getDeclaringClass().getName()
                            + " has no exception types declared and no Throwable parameter to infer from");
        }

        return result;
    }

    /**
     * Find the best handler method for the given exception.
     *
     * Maps to: {@code ExceptionHandlerMethodResolver.resolveExceptionMapping()} (line 191)
     * → {@code getMappedMethod()} (line 230)
     *
     * The "best match" is determined by exception depth — the number of steps
     * from the thrown exception type up the class hierarchy to the declared type.
     * An exact match (depth 0) always wins over a superclass match.
     *
     * @param exception the thrown exception
     * @return the best matching handler method, or null if none match
     */
    public Method resolveMethod(Throwable exception) {
        // Try matching the exception itself
        Method method = resolveByExceptionType(exception.getClass());
        if (method != null) {
            return method;
        }

        // Walk the cause chain — real framework does this recursively
        Throwable cause = exception.getCause();
        while (cause != null) {
            method = resolveByExceptionType(cause.getClass());
            if (method != null) {
                return method;
            }
            cause = cause.getCause();
        }

        return null;
    }

    /**
     * Find the best handler method for a specific exception class.
     *
     * Iterates all registered mappings and selects the one with the smallest
     * "depth" — the distance between the thrown exception type and the
     * declared handler type in the class hierarchy.
     *
     * Maps to: {@code ExceptionHandlerMethodResolver.getMappedMethod()} (line 230)
     * Real version uses ExceptionDepthComparator to sort candidates.
     */
    private Method resolveByExceptionType(Class<? extends Throwable> exceptionType) {
        Method bestMatch = null;
        int bestDepth = Integer.MAX_VALUE;

        for (Map.Entry<Class<? extends Throwable>, Method> entry : mappedMethods.entrySet()) {
            Class<? extends Throwable> declaredType = entry.getKey();
            if (declaredType.isAssignableFrom(exceptionType)) {
                int depth = getDepth(exceptionType, declaredType);
                if (depth < bestDepth) {
                    bestDepth = depth;
                    bestMatch = entry.getValue();
                }
            }
        }

        return bestMatch;
    }

    /**
     * Calculate the depth between two exception types in the class hierarchy.
     *
     * Maps to: {@code ExceptionDepthComparator.getDepth()} in
     * {@code org.springframework.core.ExceptionDepthComparator}
     *
     * Examples:
     * <pre>
     *   getDepth(IllegalArgumentException.class, IllegalArgumentException.class) = 0
     *   getDepth(IllegalArgumentException.class, RuntimeException.class) = 1
     *   getDepth(IllegalArgumentException.class, Exception.class) = 2
     * </pre>
     */
    private int getDepth(Class<?> exceptionType, Class<?> declaredType) {
        int depth = 0;
        Class<?> current = exceptionType;
        while (current != null) {
            if (current == declaredType) {
                return depth;
            }
            current = current.getSuperclass();
            depth++;
        }
        return Integer.MAX_VALUE;
    }

    /**
     * Check if this resolver has any @ExceptionHandler methods.
     */
    public boolean hasExceptionMappings() {
        return !mappedMethods.isEmpty();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exception/ExceptionHandlerExceptionResolver.java` [NEW]

```java
package com.simplespringmvc.exception;

import com.simplespringmvc.adapter.HandlerMethodReturnValueHandler;
import com.simplespringmvc.adapter.HandlerMethodReturnValueHandlerComposite;
import com.simplespringmvc.adapter.ResponseBodyReturnValueHandler;
import com.simplespringmvc.adapter.StringReturnValueHandler;
import com.simplespringmvc.annotation.ControllerAdvice;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * The concrete resolver that finds {@code @ExceptionHandler} methods on the
 * throwing controller (local) or on {@code @ControllerAdvice} beans (global)
 * and invokes them to produce an error response.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver}
 *
 * <h3>Two-phase lookup (the core design):</h3>
 * <pre>
 *   Phase 1 — LOCAL (controller-level):
 *     Look at the controller class that threw the exception.
 *     If it has an @ExceptionHandler method matching the exception type → use it.
 *     Controller-local handlers always win because they're closest to the action.
 *
 *   Phase 2 — GLOBAL (@ControllerAdvice):
 *     Iterate all @ControllerAdvice beans (in registration order).
 *     First matching @ExceptionHandler method wins.
 *     These are "catch-all" handlers for exceptions not handled locally.
 * </pre>
 *
 * <h3>Invocation:</h3>
 * <pre>
 *   Once a matching method is found:
 *     1. Build arguments: pass the exception if the method declares a Throwable parameter
 *     2. Invoke via reflection
 *     3. Process the return value through the return value handler chain
 *        (same handlers as normal handler methods — @ResponseBody → JSON, String → text)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No AbstractHandlerExceptionResolver template method hierarchy</li>
 *   <li>No {@code shouldApplyTo()} filtering</li>
 *   <li>No {@code @ControllerAdvice} filtering (basePackages, assignableTypes, etc.)</li>
 *   <li>No argument resolvers for exception handler methods — only the exception itself</li>
 *   <li>No ModelAndView return — return values go through the same handler chain</li>
 *   <li>No response reset before writing error response (real version resets buffer)</li>
 * </ul>
 */
public class ExceptionHandlerExceptionResolver implements HandlerExceptionResolver {

    /**
     * Lazily-populated cache: controller class → its ExceptionHandlerMethodResolver.
     *
     * Maps to: {@code ExceptionHandlerExceptionResolver.exceptionHandlerCache}
     * (ConcurrentHashMap<Class<?>, ExceptionHandlerMethodResolver>)
     *
     * Populated on first exception from each controller class.
     */
    private final Map<Class<?>, ExceptionHandlerMethodResolver> exceptionHandlerCache =
            new ConcurrentHashMap<>(64);

    /**
     * Eagerly-populated at init: @ControllerAdvice bean → its ExceptionHandlerMethodResolver.
     *
     * Maps to: {@code ExceptionHandlerExceptionResolver.exceptionHandlerAdviceCache}
     * (LinkedHashMap to preserve registration/discovery order)
     *
     * Populated during {@link #init(BeanContainer)} by scanning all beans annotated
     * with @ControllerAdvice.
     */
    private final Map<Object, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache =
            new LinkedHashMap<>();

    /**
     * Return value handlers for processing exception handler method results.
     * Mirrors the same chain used for normal handler methods: @ResponseBody → JSON,
     * String → text/plain.
     */
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();

    /**
     * Initialize: scan for @ControllerAdvice beans and set up return value handlers.
     *
     * Maps to: {@code ExceptionHandlerExceptionResolver.afterPropertiesSet()} (line 275)
     * → {@code initExceptionHandlerAdviceCache()} (line 299)
     *
     * The real version is called via InitializingBean.afterPropertiesSet(). We call
     * it explicitly during DispatcherServlet.initStrategies().
     *
     * @param beanContainer the container to scan for @ControllerAdvice beans
     */
    public void init(BeanContainer beanContainer) {
        initExceptionHandlerAdviceCache(beanContainer);
        initReturnValueHandlers();
    }

    /**
     * Scan all beans for @ControllerAdvice and pre-build their resolvers.
     *
     * Maps to: {@code ExceptionHandlerExceptionResolver.initExceptionHandlerAdviceCache()} (line 299)
     * Real version uses ControllerAdviceBean.findAnnotatedBeans() which respects ordering.
     */
    private void initExceptionHandlerAdviceCache(BeanContainer beanContainer) {
        for (String name : beanContainer.getBeanNames()) {
            Object bean = beanContainer.getBean(name);
            Class<?> beanType = bean.getClass();

            if (beanType.isAnnotationPresent(ControllerAdvice.class)) {
                ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(beanType);
                if (resolver.hasExceptionMappings()) {
                    exceptionHandlerAdviceCache.put(bean, resolver);
                }
            }
        }
    }

    /**
     * Set up the same return value handler chain used by SimpleHandlerAdapter.
     * Exception handler methods can use @ResponseBody for JSON or return Strings.
     */
    private void initReturnValueHandlers() {
        List<HttpMessageConverter> converters = new ArrayList<>();
        converters.add(new JacksonMessageConverter());

        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(converters));
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    /**
     * Try to resolve the exception by finding and invoking an @ExceptionHandler method.
     *
     * Maps to: {@code ExceptionHandlerExceptionResolver.doResolveHandlerMethodException()} (line 426)
     *
     * @return true if the exception was handled (response written), false otherwise
     */
    @Override
    public boolean resolveException(HttpServletRequest request, HttpServletResponse response,
                                    Object handler, Exception ex) {
        // Find a matching @ExceptionHandler method (two-phase lookup)
        HandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handler, ex);
        if (exceptionHandlerMethod == null) {
            return false;
        }

        try {
            // Reset response for error output (real framework resets buffer at line 1215)
            // Only reset if not yet committed — once committed, headers are already sent
            if (!response.isCommitted()) {
                response.reset();
            }

            // Build arguments for the exception handler method.
            // Real version uses a full argument resolver chain that supports
            // Exception, HttpServletRequest, HttpServletResponse, Model, etc.
            // We support: the exception itself and request/response.
            java.lang.reflect.Method method = exceptionHandlerMethod.getMethod();
            Object[] args = buildExceptionHandlerArgs(method, ex, request, response);

            // Ensure the method is accessible — required when the declaring class
            // is not public (e.g., package-private inner classes in tests).
            // The real framework does this in InvocableHandlerMethod (line 133):
            //   ReflectionUtils.makeAccessible(getBridgedMethod())
            method.setAccessible(true);

            // Invoke the exception handler method
            Object result = method.invoke(exceptionHandlerMethod.getBean(), args);

            // Process the return value through the same handler chain as normal methods.
            // This means @ResponseBody on exception handler methods → JSON output,
            // String return → text/plain.
            returnValueHandlers.handleReturnValue(result, exceptionHandlerMethod, request, response);

            return true;

        } catch (Exception resolverEx) {
            // If the exception handler itself throws, log and fall through
            System.err.println("Exception handler method threw exception: " + resolverEx);
            return false;
        }
    }

    /**
     * Two-phase lookup for the best @ExceptionHandler method.
     *
     * Maps to: {@code ExceptionHandlerExceptionResolver.getExceptionHandlerMethod()} (line 505)
     *
     * Phase 1: Check the controller that threw the exception (local handlers).
     * Phase 2: Check @ControllerAdvice beans (global handlers).
     *
     * Controller-local handlers always take priority — this is a deliberate design
     * choice that lets controllers override global error handling for their specific cases.
     */
    private HandlerMethod getExceptionHandlerMethod(Object handler, Exception ex) {
        // Phase 1: Local — check the controller's own @ExceptionHandler methods
        if (handler instanceof HandlerMethod handlerMethod) {
            Class<?> controllerType = handlerMethod.getBeanType();

            // Lazily compute and cache the resolver for this controller class
            ExceptionHandlerMethodResolver resolver = exceptionHandlerCache.computeIfAbsent(
                    controllerType, ExceptionHandlerMethodResolver::new);

            Method method = resolver.resolveMethod(ex);
            if (method != null) {
                return new HandlerMethod(handlerMethod.getBean(), method);
            }
        }

        // Phase 2: Global — check @ControllerAdvice beans
        for (Map.Entry<Object, ExceptionHandlerMethodResolver> entry : exceptionHandlerAdviceCache.entrySet()) {
            Object adviceBean = entry.getKey();
            ExceptionHandlerMethodResolver resolver = entry.getValue();

            Method method = resolver.resolveMethod(ex);
            if (method != null) {
                return new HandlerMethod(adviceBean, method);
            }
        }

        return null;
    }

    /**
     * Build the argument array for an exception handler method by matching
     * parameter types to available objects.
     *
     * Maps to: The real framework uses a full argument resolver chain configured
     * in {@code ExceptionHandlerExceptionResolver.afterPropertiesSet()} with
     * resolvers for Exception, WebRequest, HttpServletRequest/Response, Model, etc.
     *
     * We support three parameter types:
     * <ul>
     *   <li>{@code Throwable} (or subclass) — receives the exception</li>
     *   <li>{@code HttpServletRequest} — receives the current request</li>
     *   <li>{@code HttpServletResponse} — receives the current response</li>
     * </ul>
     */
    private Object[] buildExceptionHandlerArgs(Method method, Exception ex,
                                                HttpServletRequest request,
                                                HttpServletResponse response) {
        Class<?>[] paramTypes = method.getParameterTypes();
        Object[] args = new Object[paramTypes.length];

        for (int i = 0; i < paramTypes.length; i++) {
            if (Throwable.class.isAssignableFrom(paramTypes[i])) {
                // Find the most specific exception in the cause chain that matches
                args[i] = findMatchingException(ex, paramTypes[i]);
            } else if (HttpServletRequest.class.isAssignableFrom(paramTypes[i])) {
                args[i] = request;
            } else if (HttpServletResponse.class.isAssignableFrom(paramTypes[i])) {
                args[i] = response;
            }
        }

        return args;
    }

    /**
     * Find the exception in the cause chain that matches the declared parameter type.
     * If the top-level exception matches, return it. Otherwise, walk the cause chain.
     */
    private Throwable findMatchingException(Throwable ex, Class<?> declaredType) {
        Throwable current = ex;
        while (current != null) {
            if (declaredType.isInstance(current)) {
                return current;
            }
            current = current.getCause();
        }
        // Fallback to the top-level exception
        return ex;
    }

    /**
     * Provides access to the advice cache for testing.
     */
    public Map<Object, ExceptionHandlerMethodResolver> getExceptionHandlerAdviceCache() {
        return exceptionHandlerAdviceCache;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.exception.ExceptionHandlerExceptionResolver;
import com.simplespringmvc.exception.HandlerExceptionResolver;
import com.simplespringmvc.interceptor.HandlerExecutionChain;
import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;
    private final List<HandlerInterceptor> interceptors = new ArrayList<>();
    private final List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<>();  // ch12

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    protected void initStrategies() {
        initHandlerMapping();
        initHandlerAdapter();
        initExceptionResolvers();  // ch12
    }

    private void initHandlerMapping() { /* unchanged */ }
    private void initHandlerAdapter() { /* unchanged */ }

    // ch12: NEW method
    private void initExceptionResolvers() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(beanContainer);
        exceptionResolvers.add(resolver);
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException { /* unchanged */ }

    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerExecutionChain mappedHandler = null;
        Exception dispatchException = null;

        try {
            mappedHandler = getHandler(request);
            if (mappedHandler == null) {
                response.sendError(HttpServletResponse.SC_NOT_FOUND, /* ... */);
                return;
            }
            if (!mappedHandler.applyPreHandle(request, response)) return;
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
            adapter.handle(request, response, mappedHandler.getHandler());
            mappedHandler.applyPostHandle(request, response);
        } catch (Exception ex) {
            dispatchException = ex;
        }

        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, dispatchException);
        }

        // ch12: Try exception resolution before re-throwing
        if (dispatchException != null) {
            Object handler = (mappedHandler != null) ? mappedHandler.getHandler() : null;
            if (!processHandlerException(request, response, handler, dispatchException)) {
                throw dispatchException;
            }
        }
    }

    // ch12: NEW method
    private boolean processHandlerException(HttpServletRequest request, HttpServletResponse response,
                                            Object handler, Exception ex) {
        for (HandlerExceptionResolver resolver : exceptionResolvers) {
            if (resolver.resolveException(request, response, handler, ex)) return true;
        }
        return false;
    }

    // ... getHandler(), getHandlerAdapter(), addInterceptor(), getters — unchanged ...

    // ch12: NEW accessor
    public List<HandlerExceptionResolver> getExceptionResolvers() {
        return List.copyOf(exceptionResolvers);
    }
}
```

> **Note:** The `[MODIFIED]` listing above shows the structural changes. See the actual file in `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` for the complete implementation with full Javadoc.

### Test Code

#### File: `src/test/java/com/simplespringmvc/exception/ExceptionHandlerMethodResolverTest.java` [NEW]

```java
package com.simplespringmvc.exception;

import com.simplespringmvc.annotation.ExceptionHandler;
import com.simplespringmvc.annotation.ResponseBody;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.lang.reflect.Method;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class ExceptionHandlerMethodResolverTest {

    static class ExplicitTypeController {
        @ExceptionHandler(IllegalArgumentException.class)
        @ResponseBody
        public Map<String, String> handleBadRequest(IllegalArgumentException ex) {
            return Map.of("error", ex.getMessage());
        }
    }

    static class InferredTypeController {
        @ExceptionHandler
        @ResponseBody
        public String handleRuntime(RuntimeException ex) {
            return ex.getMessage();
        }
    }

    static class MultipleHandlerController {
        @ExceptionHandler(IllegalArgumentException.class)
        @ResponseBody
        public String handleBadArg(IllegalArgumentException ex) { return "bad arg"; }

        @ExceptionHandler(RuntimeException.class)
        @ResponseBody
        public String handleRuntime(RuntimeException ex) { return "runtime"; }

        @ExceptionHandler(Exception.class)
        @ResponseBody
        public String handleAll(Exception ex) { return "all"; }
    }

    static class IOExceptionController {
        @ExceptionHandler(IOException.class)
        @ResponseBody
        public String handleIO(IOException ex) { return "io"; }
    }

    static class NoHandlerController {
        public String normalMethod() { return "hello"; }
    }

    @Test
    void shouldDiscoverExplicitExceptionType() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(ExplicitTypeController.class);
        assertTrue(resolver.hasExceptionMappings());
    }

    @Test
    void shouldDiscoverInferredExceptionType() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(InferredTypeController.class);
        assertTrue(resolver.hasExceptionMappings());
    }

    @Test
    void shouldReportNoMappings_WhenClassHasNoExceptionHandlers() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(NoHandlerController.class);
        assertFalse(resolver.hasExceptionMappings());
    }

    @Test
    void shouldResolveExactMatch() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(ExplicitTypeController.class);
        Method method = resolver.resolveMethod(new IllegalArgumentException("bad"));
        assertNotNull(method);
        assertEquals("handleBadRequest", method.getName());
    }

    @Test
    void shouldResolveSubclass_WhenExactMatchNotAvailable() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(ExplicitTypeController.class);
        Method method = resolver.resolveMethod(new NumberFormatException("not a number"));
        assertNotNull(method);
        assertEquals("handleBadRequest", method.getName());
    }

    @Test
    void shouldReturnNull_WhenNoMatchingHandler() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(IOExceptionController.class);
        Method method = resolver.resolveMethod(new IllegalArgumentException("bad"));
        assertNull(method);
    }

    @Test
    void shouldResolveClosestMatch_WhenMultipleHandlersApply() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(MultipleHandlerController.class);
        assertEquals("handleBadArg", resolver.resolveMethod(new IllegalArgumentException("bad")).getName());
        assertEquals("handleRuntime", resolver.resolveMethod(new RuntimeException("runtime")).getName());
        assertEquals("handleAll", resolver.resolveMethod(new IOException("io")).getName());
    }

    @Test
    void shouldResolveInferredType_FromMethodParameter() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(InferredTypeController.class);
        Method method = resolver.resolveMethod(new RuntimeException("inferred"));
        assertNotNull(method);
        assertEquals("handleRuntime", method.getName());
    }

    @Test
    void shouldWalkCauseChain_WhenTopLevelDoesNotMatch() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(IOExceptionController.class);
        RuntimeException wrapper = new RuntimeException("wrapper", new IOException("cause"));
        Method method = resolver.resolveMethod(wrapper);
        assertNotNull(method);
        assertEquals("handleIO", method.getName());
    }

    @Test
    void shouldReturnNull_WhenCauseChainHasNoMatch() {
        ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(IOExceptionController.class);
        Method method = resolver.resolveMethod(new IllegalStateException("no match"));
        assertNull(method);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/exception/ExceptionHandlerExceptionResolverTest.java` [NEW]

```java
package com.simplespringmvc.exception;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.ControllerAdvice;
import com.simplespringmvc.annotation.ExceptionHandler;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.lang.reflect.Method;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ExceptionHandlerExceptionResolverTest {

    private HttpServletRequest request;
    private HttpServletResponse response;
    private StringWriter responseBody;

    @Controller
    static class TestController {
        public String throwingMethod() { throw new IllegalArgumentException("bad input"); }

        @ExceptionHandler(IllegalArgumentException.class)
        @ResponseBody
        public Map<String, String> handleBadRequest(IllegalArgumentException ex) {
            return Map.of("error", ex.getMessage());
        }
    }

    @Controller
    static class ControllerWithNoHandler {
        public String throwingMethod() { throw new RuntimeException("oops"); }
    }

    @ControllerAdvice
    static class GlobalExceptionHandler {
        @ExceptionHandler(RuntimeException.class)
        @ResponseBody
        public Map<String, String> handleRuntime(RuntimeException ex) {
            return Map.of("globalError", ex.getMessage());
        }
    }

    @ControllerAdvice
    static class AnotherGlobalHandler {
        @ExceptionHandler(Exception.class)
        @ResponseBody
        public Map<String, String> handleAll(Exception ex) {
            return Map.of("fallback", ex.getMessage());
        }
    }

    @BeforeEach
    void setUp() throws Exception {
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    @Test
    void shouldDiscoverControllerAdviceBeans() {
        BeanContainer container = new SimpleBeanContainer();
        container.registerBean(new GlobalExceptionHandler());
        container.registerBean(new AnotherGlobalHandler());
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        assertThat(resolver.getExceptionHandlerAdviceCache()).hasSize(2);
    }

    @Test
    void shouldIgnoreBeansWithoutControllerAdvice() {
        BeanContainer container = new SimpleBeanContainer();
        container.registerBean(new TestController());
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        assertThat(resolver.getExceptionHandlerAdviceCache()).isEmpty();
    }

    @Test
    void shouldResolveFromLocalController_WhenControllerHasHandler() throws Exception {
        BeanContainer container = new SimpleBeanContainer();
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        TestController controller = new TestController();
        Method throwingMethod = TestController.class.getMethod("throwingMethod");
        HandlerMethod handler = new HandlerMethod(controller, throwingMethod);
        boolean handled = resolver.resolveException(request, response, handler,
                new IllegalArgumentException("bad input"));
        assertThat(handled).isTrue();
        assertThat(responseBody.toString()).contains("bad input");
    }

    @Test
    void shouldFallbackToGlobalAdvice_WhenControllerHasNoHandler() throws Exception {
        BeanContainer container = new SimpleBeanContainer();
        container.registerBean(new GlobalExceptionHandler());
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        ControllerWithNoHandler controller = new ControllerWithNoHandler();
        Method throwingMethod = ControllerWithNoHandler.class.getMethod("throwingMethod");
        HandlerMethod handler = new HandlerMethod(controller, throwingMethod);
        boolean handled = resolver.resolveException(request, response, handler,
                new RuntimeException("oops"));
        assertThat(handled).isTrue();
        assertThat(responseBody.toString()).contains("globalError");
    }

    @Test
    void shouldPreferLocalHandler_OverGlobalAdvice() throws Exception {
        BeanContainer container = new SimpleBeanContainer();
        container.registerBean(new GlobalExceptionHandler());
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        TestController controller = new TestController();
        Method throwingMethod = TestController.class.getMethod("throwingMethod");
        HandlerMethod handler = new HandlerMethod(controller, throwingMethod);
        boolean handled = resolver.resolveException(request, response, handler,
                new IllegalArgumentException("local wins"));
        assertThat(handled).isTrue();
        assertThat(responseBody.toString()).contains("\"error\"");
        assertThat(responseBody.toString()).doesNotContain("globalError");
    }

    @Test
    void shouldReturnFalse_WhenNoHandlerMatches() {
        BeanContainer container = new SimpleBeanContainer();
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        assertThat(resolver.resolveException(request, response, null,
                new RuntimeException("unhandled"))).isFalse();
    }

    @Controller
    static class ControllerWithRequestParam {
        public String throwingMethod() { throw new RuntimeException("test"); }

        @ExceptionHandler(RuntimeException.class)
        public void handleWithResponse(RuntimeException ex, HttpServletResponse resp) throws Exception {
            resp.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            resp.setContentType("text/plain");
            resp.getWriter().write("Error: " + ex.getMessage());
        }
    }

    @Test
    void shouldPassRequestAndResponse_WhenHandlerDeclaresThemAsParameters() throws Exception {
        BeanContainer container = new SimpleBeanContainer();
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(container);
        ControllerWithRequestParam controller = new ControllerWithRequestParam();
        Method throwingMethod = ControllerWithRequestParam.class.getMethod("throwingMethod");
        HandlerMethod handler = new HandlerMethod(controller, throwingMethod);
        boolean handled = resolver.resolveException(request, response, handler,
                new RuntimeException("test"));
        assertThat(handled).isTrue();
        verify(response).setStatus(HttpServletResponse.SC_BAD_REQUEST);
        assertThat(responseBody.toString()).contains("Error: test");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/ExceptionHandlingIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.*;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ExceptionHandlingIntegrationTest {

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @RestController
    static class ProductController {
        @GetMapping("/products")
        public Map<String, Object> getProducts() { return Map.of("products", "all"); }

        @GetMapping("/products/bad-request")
        public String throwBadRequest() { throw new IllegalArgumentException("Invalid product ID"); }

        @GetMapping("/products/server-error")
        public String throwServerError() { throw new IllegalStateException("Database down"); }

        @ExceptionHandler(IllegalArgumentException.class)
        @ResponseBody
        public Map<String, String> handleBadRequest(IllegalArgumentException ex) {
            return Map.of("error", "Bad Request", "message", ex.getMessage());
        }
    }

    @RestController
    static class OrderController {
        @GetMapping("/orders/error")
        public String throwError() { throw new RuntimeException("Order processing failed"); }
    }

    @ControllerAdvice
    static class GlobalExceptionAdvice {
        @ExceptionHandler(RuntimeException.class)
        @ResponseBody
        public Map<String, String> handleRuntime(RuntimeException ex) {
            return Map.of("error", "Internal Error", "message", ex.getMessage());
        }
    }

    @AfterEach
    void tearDown() throws Exception { if (tomcat != null) tomcat.stop(); }

    private void startServer(SimpleBeanContainer container) throws Exception {
        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();
        baseUrl = "http://localhost:" + tomcat.getPort();
        httpClient = HttpClient.newHttpClient();
    }

    private HttpResponse<String> get(String path) throws Exception {
        return httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + path)).GET().build(),
                HttpResponse.BodyHandlers.ofString());
    }

    @Test @DisplayName("normal request works")
    void shouldHandleNormalRequest_WhenNoExceptionThrown() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new ProductController()); c.registerBean(new GlobalExceptionAdvice());
        startServer(c);
        assertThat(get("/products").statusCode()).isEqualTo(200);
    }

    @Test @DisplayName("local @ExceptionHandler")
    void shouldUseLocalExceptionHandler_WhenControllerThrows() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new ProductController()); c.registerBean(new GlobalExceptionAdvice());
        startServer(c);
        HttpResponse<String> r = get("/products/bad-request");
        assertThat(r.statusCode()).isEqualTo(200);
        assertThat(r.body()).contains("Bad Request").contains("Invalid product ID");
    }

    @Test @DisplayName("global @ControllerAdvice")
    void shouldUseGlobalAdvice_WhenControllerHasNoLocalHandler() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new OrderController()); c.registerBean(new GlobalExceptionAdvice());
        startServer(c);
        HttpResponse<String> r = get("/orders/error");
        assertThat(r.statusCode()).isEqualTo(200);
        assertThat(r.body()).contains("Internal Error");
    }

    @Test @DisplayName("local wins over global")
    void shouldPreferLocalHandler_OverGlobalAdvice_WhenBothMatch() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new ProductController()); c.registerBean(new GlobalExceptionAdvice());
        startServer(c);
        assertThat(get("/products/bad-request").body()).contains("Bad Request")
                .doesNotContain("Internal Error");
    }

    @Test @DisplayName("fallthrough to global")
    void shouldFallToGlobal_WhenLocalHandlerDoesNotMatchExceptionType() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new ProductController()); c.registerBean(new GlobalExceptionAdvice());
        startServer(c);
        assertThat(get("/products/server-error").body()).contains("Internal Error").contains("Database down");
    }

    @RestController
    static class UnhandledController {
        @GetMapping("/unhandled")
        public String throwChecked() throws Exception { throw new Exception("checked"); }
    }

    @Test @DisplayName("unhandled → 500")
    void shouldReturn500_WhenNoResolverHandlesException() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new UnhandledController());
        startServer(c);
        assertThat(get("/unhandled").statusCode()).isEqualTo(500);
    }

    @Test @DisplayName("resolvers initialized")
    void shouldInitializeExceptionResolvers() throws Exception {
        SimpleBeanContainer c = new SimpleBeanContainer();
        c.registerBean(new ProductController());
        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(c);
        servlet.init();
        assertThat(servlet.getExceptionResolvers()).isNotEmpty();
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`HandlerExceptionResolver`** | Strategy interface — resolves exceptions thrown during handler execution via Chain of Responsibility |
| **`@ExceptionHandler`** | Method-level annotation marking a method as an error handler for specific exception types |
| **`@ControllerAdvice`** | Class-level annotation marking a global exception handler bean consulted after controller-local handlers |
| **Two-phase lookup** | Local (controller) handlers checked first, global (`@ControllerAdvice`) second — specificity wins |
| **Exception depth** | When multiple handlers match, the one closest to the thrown exception type in the class hierarchy wins |
| **Chain of Responsibility** | `DispatcherServlet` iterates resolvers in order; first to return "handled" wins |

**Next: Chapter 13 — View Resolution & Rendering** — Add template-based HTML rendering as an alternative to `@ResponseBody` JSON, introducing `ViewResolver`, `View`, and `ModelAndView` for when controllers need to produce HTML pages instead of API responses.
