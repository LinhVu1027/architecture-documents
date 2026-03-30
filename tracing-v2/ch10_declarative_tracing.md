# Chapter 10: Declarative Tracing — @NewSpan, @ContinueSpan, @SpanTag Annotations

> **What you'll build**: Three annotations and four processing classes that let clients
> instrument methods declaratively — `@NewSpan` creates a child span around a method,
> `@ContinueSpan` enriches the current span, and `@SpanTag` automatically tags spans
> from method parameters. An AspectJ aspect intercepts annotated methods and delegates
> to a processor that manages the full span lifecycle.

---

## 1. The API Contract

After this chapter, a client can write:

```java
public class UserService {

    @NewSpan("get-user")
    public User getUser(@SpanTag("user.id") String userId) {
        return userRepository.findById(userId);
    }

    @ContinueSpan(log = "validate-user")
    public void validateUser(@SpanTag("user.name") String name) {
        // enriches the current span instead of creating a new one
    }
}
```

**Why this matters**: Up to now, tracing required explicit span creation — calling
`tracer.nextSpan().name("x").start()` and managing scope/end manually. Annotation-based
tracing lets you declaratively say "trace this method" without touching span lifecycle
code. This is the pattern Spring Cloud Sleuth (now Micrometer Tracing) popularized —
add an annotation, get a span.

### The Three Annotations

```
@NewSpan("name")                ← Creates a NEW child span around the method
├── name/value: span name       (defaults to method name in lower-hyphen-case)
│
@ContinueSpan(log = "prefix")  ← ENRICHES the current span (no new span)
├── log: event prefix           (records "prefix.before", "prefix.after" events)
│
@SpanTag("key")                 ← Tags the span from a method PARAMETER
└── value/key: tag key name     (tag value = parameter.toString())
```

### The Processing Flow

```
Client method                  SpanAspect              Processor
─────────────                  ──────────              ─────────
@NewSpan method called  →  @Around intercepts  →  create span, name it, start
                                                   process @SpanTag params
                                                   proceed with method
                                                   end span (or record error)

@ContinueSpan called   →  @Around intercepts  →  get current span (or fallback)
                                                   process @SpanTag params
                                                   log lifecycle events
                                                   proceed with method
                                                   (does NOT end the span)
```

**Key design choice**: The aspect is deliberately thin — it only intercepts, adapts
(ProceedingJoinPoint → MethodInvocation), and delegates. All tracing logic lives in
the `ImperativeMethodInvocationProcessor`. This separation means:
1. The aspect can be swapped for Spring AOP without changing the processor
2. The processor can be tested without AspectJ infrastructure
3. A reactive variant could be added alongside the imperative one

---

## 2. Client Usage & Tests

### 2a. Testing @NewSpan — Create a New Span

```java
@Test
void shouldCreateNewSpan_WhenNewSpanAnnotationPresent() throws Throwable {
    // Arrange — simulate a method annotated with @NewSpan("get-user")
    MethodInvocation invocation = createInvocation(
            AnnotatedService.class, "getUser", new Class<?>[] { String.class },
            new AnnotatedService(), new Object[] { "user-42" });
    NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

    // Act
    Object result = processor.process(invocation, newSpan, null);

    // Assert — a new span was created, named from the annotation, and ended
    assertThat(result).isEqualTo("User:user-42");
    assertThat(tracer.finishedSpans()).hasSize(1);
    TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
    assertThat(span.name).isEqualTo("get-user");
    assertThat(span.tags).containsEntry("user.id", "user-42");
    assertThat(span.tags).containsEntry("class", "AnnotatedService");
    assertThat(span.tags).containsEntry("method", "getUser");
}
```

Notice we test the processor directly, not the aspect. The `createInvocation()` helper
creates a `MethodInvocation` from a real annotated method — mimicking what the aspect
would do at runtime.

### 2b. Testing Method Name Fallback

```java
@Test
void shouldUseLowerHyphenMethodName_WhenNewSpanHasNoName() throws Throwable {
    MethodInvocation invocation = createInvocation(
            AnnotatedService.class, "processOrder", new Class<?>[] {},
            new AnnotatedService(), new Object[] {});
    NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

    processor.process(invocation, newSpan, null);

    // method name "processOrder" → "process-order" via toLowerHyphen()
    assertThat(tracer.finishedSpans().get(0).name).isEqualTo("process-order");
}
```

### 2c. Testing @ContinueSpan — Enrich Existing Span

```java
@Test
void shouldEnrichCurrentSpan_WhenContinueSpanAnnotationPresent() throws Throwable {
    // Start a span, then call a @ContinueSpan method
    Span existing = tracer.nextSpan().name("existing-operation").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(existing)) {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "validateUser", new Class<?>[] { String.class },
                new AnnotatedService(), new Object[] { "Alice" });
        ContinueSpan continueSpan = invocation.getMethod().getAnnotation(ContinueSpan.class);

        processor.process(invocation, null, continueSpan);
    } finally {
        existing.end();
    }

    // No new span — the existing span was enriched with the tag
    assertThat(tracer.finishedSpans()).hasSize(1);
    TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
    assertThat(span.name).isEqualTo("existing-operation");
    assertThat(span.tags).containsEntry("user.name", "Alice");
    assertThat(span.events).contains("validate-user.before", "validate-user.after");
}
```

### 2d. Testing Error Handling

```java
@Test
void shouldRecordErrorAndEndSpan_WhenMethodThrows() throws Throwable {
    MethodInvocation invocation = createInvocation(
            AnnotatedService.class, "failingMethod", new Class<?>[] {},
            new AnnotatedService(), new Object[] {});
    NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

    // The exception propagates to the caller
    assertThatThrownBy(() -> processor.process(invocation, newSpan, null))
            .isInstanceOf(RuntimeException.class)
            .hasMessage("something went wrong");

    // But the span was still ended AND the error recorded
    TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
    assertThat(span.name).isEqualTo("failing-method");
    assertThat(span.error).isNotNull();
    assertThat(span.error.getMessage()).isEqualTo("something went wrong");
}
```

### 2e. The Annotated Service

```java
public static class AnnotatedService {

    @NewSpan("get-user")
    public String getUser(@SpanTag("user.id") String userId) {
        return "User:" + userId;
    }

    @NewSpan
    public String processOrder() {
        return "processed";
    }

    @ContinueSpan(log = "validate-user")
    public void validateUser(@SpanTag("user.name") String name) {
        // enriches the current span
    }

    @NewSpan("failing-method")
    public void failingMethod() {
        throw new RuntimeException("something went wrong");
    }
}
```

---

## 3. Implementing the Call Chain — Top Down

### 3a. API Layer: The Three Annotations

The annotations are pure metadata — no logic. They serve as the **declarative API
contract** that clients use.

**@NewSpan** — Creates a new child span:

```java
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Target(ElementType.METHOD)
public @interface NewSpan {

    String name() default "";

    String value() default "";

}
```

Two synonymous attributes (`name` and `value`) for the span name, with method name as
fallback. `@Inherited` ensures subclass methods inherit the annotation.

**@ContinueSpan** — Enriches the current span:

```java
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Target(ElementType.METHOD)
public @interface ContinueSpan {

    String log() default "";

}
```

The `log` attribute provides a prefix for lifecycle events recorded on the span:
`"validate-user.before"`, `"validate-user.after"`, `"validate-user.afterFailure"`.

**@SpanTag** — Tags the span from a parameter:

```java
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Target(ElementType.PARAMETER)
public @interface SpanTag {

    String value() default "";

    String key() default "";

}
```

Placed on parameters (not methods). The real framework supports custom `ValueResolver`
and SpEL expressions — we simplify to `toString()`.

### 3b. Interception Layer: SpanAspect

The aspect is the **entry point** that intercepts annotated methods:

```java
@Aspect
public class SpanAspect {

    private final MethodInvocationProcessor methodInvocationProcessor;

    public SpanAspect(MethodInvocationProcessor methodInvocationProcessor) {
        this.methodInvocationProcessor = methodInvocationProcessor;
    }

    @Around("@annotation(io.simpletracing.annotation.NewSpan)")
    public Object newSpanMethod(ProceedingJoinPoint pjp) throws Throwable {
        Method method = getMethod(pjp);
        NewSpan newSpan = method.getAnnotation(NewSpan.class);
        return methodInvocationProcessor.process(
                new SpanAspectMethodInvocation(pjp, method), newSpan, null);
    }

    @Around("@annotation(io.simpletracing.annotation.ContinueSpan)")
    public Object continueSpanMethod(ProceedingJoinPoint pjp) throws Throwable {
        Method method = getMethod(pjp);
        ContinueSpan continueSpan = method.getAnnotation(ContinueSpan.class);
        return methodInvocationProcessor.process(
                new SpanAspectMethodInvocation(pjp, method), null, continueSpan);
    }

    private Method getMethod(ProceedingJoinPoint pjp) throws NoSuchMethodException {
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        return pjp.getTarget().getClass().getMethod(method.getName(), method.getParameterTypes());
    }
}
```

**Three key design decisions:**

1. **Two separate `@Around` advices** — one for `@NewSpan`, one for `@ContinueSpan`. The
   distinction is communicated to the processor via which parameter is non-null.

2. **`getMethod()` resolves from the target class** — not the proxy/interface. This
   ensures annotations on the concrete implementation class are found, even behind
   Spring AOP proxies.

3. **`SpanAspectMethodInvocation`** — bridges from AspectJ's `ProceedingJoinPoint` to
   the standard `org.aopalliance.intercept.MethodInvocation`. This decouples the
   processor from AspectJ, so the same processor works with Spring AOP.

### 3c. Processing Interface: MethodInvocationProcessor

```java
public interface MethodInvocationProcessor {

    Object process(MethodInvocation invocation, NewSpan newSpan, ContinueSpan continueSpan)
            throws Throwable;

}
```

A single-method **Strategy interface**. The aspect depends on this, not on the concrete
processor. This allows:
- `ImperativeMethodInvocationProcessor` for synchronous methods
- A reactive variant (not built here) for `Mono`/`Flux` return types

### 3d. Processing Layer: ImperativeMethodInvocationProcessor

This is the **core logic** — where spans are actually created, named, tagged, and ended:

```java
public class ImperativeMethodInvocationProcessor implements MethodInvocationProcessor {

    private final NewSpanParser newSpanParser;
    private final Tracer tracer;
    private final SpanTagAnnotationHandler spanTagAnnotationHandler;

    @Override
    public Object process(MethodInvocation invocation, NewSpan newSpan, ContinueSpan continueSpan)
            throws Throwable {
        Span span = tracer.currentSpan();
        // Create a new span for @NewSpan, or as a fallback when @ContinueSpan has no current span
        boolean startNewSpan = newSpan != null || span == null;
        if (startNewSpan) {
            span = tracer.nextSpan();
            newSpanParser.parse(invocation, newSpan, span);
            span.start();
        }
        String log = logPrefix(continueSpan);
        boolean hasLog = !log.isEmpty();
        try (Tracer.SpanInScope ignored = tracer.withSpan(span)) {
            before(invocation, span, log, hasLog);
            return invocation.proceed();
        } catch (Exception ex) {
            onFailure(span, log, hasLog, ex);
            throw ex;
        } finally {
            after(span, startNewSpan, log, hasLog);
        }
    }
}
```

**The `@ContinueSpan` fallback**: If no span is in scope, the processor creates a new
one rather than NPE-ing. This defensive design means `@ContinueSpan` never silently
loses data — it always attaches tags to *something*.

**The `finally` block always runs**: Whether the method succeeds or throws, `after()` is
called. For `@NewSpan`, this calls `span.end()`. For `@ContinueSpan`, it only logs the
"after" event (the span belongs to the caller).

### 3e. Naming Layer: DefaultNewSpanParser

```java
public class DefaultNewSpanParser implements NewSpanParser {

    @Override
    public void parse(MethodInvocation methodInvocation, NewSpan newSpan, Span span) {
        String name = spanName(newSpan, methodInvocation);
        span.name(toLowerHyphen(name));
    }

    private String spanName(NewSpan newSpan, MethodInvocation invocation) {
        if (newSpan == null) {
            return invocation.getMethod().getName();
        }
        String name = newSpan.name();
        String value = newSpan.value();
        if (!name.isEmpty()) return name;
        if (!value.isEmpty()) return value;
        return invocation.getMethod().getName();
    }

    static String toLowerHyphen(String name) {
        if (name.isEmpty()) return name;
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < name.length(); i++) {
            char c = name.charAt(i);
            if (Character.isUpperCase(c)) {
                if (i > 0) result.append('-');
                result.append(Character.toLowerCase(c));
            } else {
                result.append(c);
            }
        }
        return result.toString();
    }
}
```

**Naming precedence**: `name()` > `value()` > method name. The resolved name is converted
to lower-hyphen-case to match distributed tracing conventions (Zipkin, OpenTelemetry).

### 3f. Tagging Layer: SpanTagAnnotationHandler

```java
public class SpanTagAnnotationHandler {

    public void addAnnotatedParameters(Method method, Object[] arguments, SpanCustomizer customizer) {
        Annotation[][] parameterAnnotations = method.getParameterAnnotations();
        for (int i = 0; i < parameterAnnotations.length; i++) {
            for (Annotation annotation : parameterAnnotations[i]) {
                if (annotation instanceof SpanTag spanTag) {
                    String key = resolveKey(spanTag);
                    if (!key.isEmpty()) {
                        String value = resolveValue(arguments[i]);
                        customizer.tag(key, value);
                    }
                }
            }
        }
    }

    private String resolveKey(SpanTag spanTag) {
        return !spanTag.value().isEmpty() ? spanTag.value() : spanTag.key();
    }

    private String resolveValue(Object argument) {
        return argument == null ? "" : argument.toString();
    }
}
```

Uses `Method.getParameterAnnotations()` to scan each parameter. The tag key comes from
`value()` (preferred) or `key()`. The tag value is always `toString()` — the real
framework adds `ValueResolver` and SpEL expression support on top.

---

## 4. Try It Yourself

1. **Add a `@SpanTag` resolver**: Extend `SpanTagAnnotationHandler` to support a
   `resolver` attribute on `@SpanTag` — a `Function<Object, String>` that can compute
   tag values from complex objects (e.g., extracting an ID from a domain entity).

2. **Add `@NewSpan` on the class level**: Create a class-level `@NewSpan` that applies
   to all public methods, so you don't need to annotate every method individually.

3. **Build a reactive variant**: Create `ReactiveMethodInvocationProcessor` that wraps
   `Mono`/`Flux` return types — starting the span on subscribe and ending on complete.

---

## 5. Why This Works

### The Thin Aspect Pattern
The aspect does three things: intercept, adapt, delegate. All tracing logic lives in the
processor. This means you can swap the interception mechanism (AspectJ ↔ Spring AOP ↔
manual proxy) without changing any tracing logic. The same processor works everywhere.

### Strategy Pattern for Processing
`MethodInvocationProcessor` is a strategy interface. The aspect doesn't know whether
it's dealing with synchronous or reactive code — it just calls `process()`. The concrete
implementation decides how to manage span lifecycle around the method execution.

### Defensive Fallback in @ContinueSpan
Rather than throwing when no current span exists, the processor creates a fallback span.
This prevents silent data loss in cases where `@ContinueSpan` is called outside an
existing trace context (e.g., during testing or misconfiguration). The span still captures
the tags and events — better to have data in the wrong place than no data at all.

### Reflection-Based Tag Resolution
`SpanTagAnnotationHandler` uses `Method.getParameterAnnotations()` — standard Java
reflection. This is called once per method invocation, which is fine for the typical
tracing use case (hundreds, not millions, of traced calls per second). The real framework
adds caching on top for hot paths.

---

## 6. What We Enhanced

| Component | Before Feature 10 | After Feature 10 |
|-----------|-------------------|-------------------|
| Span creation | Explicit `tracer.nextSpan().start()` | Declarative `@NewSpan` |
| Span enrichment | Manual `span.tag()` calls | Declarative `@SpanTag` on params |
| Continue span | Manual `tracer.currentSpan()` | Declarative `@ContinueSpan` |
| Method tracing | ~8 lines of boilerplate per method | 1 annotation per method |
| `simple-tracing-api/build.gradle` | No AOP dependencies | Added aopalliance + aspectjweaver |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Differences |
|------------------|---------------------|-----------------|
| `@NewSpan` | `io.micrometer.tracing.annotation.NewSpan` | Identical |
| `@ContinueSpan` | `io.micrometer.tracing.annotation.ContinueSpan` | Identical |
| `@SpanTag` | `io.micrometer.tracing.annotation.SpanTag` | Simplified: no `resolver` or `expression` |
| `SpanAspect` | `io.micrometer.tracing.annotation.SpanAspect` | Identical structure |
| `SpanAspectMethodInvocation` | `io.micrometer.tracing.annotation.SpanAspectMethodInvocation` | Identical |
| `MethodInvocationProcessor` | `io.micrometer.tracing.annotation.MethodInvocationProcessor` | Identical |
| `ImperativeMethodInvocationProcessor` | `io.micrometer.tracing.annotation.ImperativeMethodInvocationProcessor` | Simplified: no `AbstractMethodInvocationProcessor` base class |
| `NewSpanParser` | `io.micrometer.tracing.annotation.NewSpanParser` | Identical |
| `DefaultNewSpanParser` | `io.micrometer.tracing.annotation.DefaultNewSpanParser` | Simplified: inline `toLowerHyphen` instead of `SpanNameUtil` |
| `SpanTagAnnotationHandler` | `io.micrometer.tracing.annotation.SpanTagAnnotationHandler` | Simplified: no `AnnotationHandler` base class, no `ValueResolver`/`ValueExpressionResolver` |

**What the real framework adds:**
- `AbstractMethodInvocationProcessor` — template-method base class with `before()`, `after()`, `onFailure()` hooks. We inline these directly.
- `ValueResolver` / `ValueExpressionResolver` — pluggable tag value computation (custom beans, SpEL). We use `toString()`.
- `ReactiveMethodInvocationProcessor` — handles `Mono`/`Flux` return types by wrapping the reactive chain.
- `SpanNameUtil` — shared utility for name conversion. We inline a simplified version.
- Integration with `micrometer-common`'s `AnnotationHandler` — generic annotation-scanning base class. We use direct reflection.

---

## 8. Complete Code

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/NewSpan.java`

```java
package io.simpletracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation that creates a <b>new child span</b> around the annotated method.
 * The span is started before the method executes and ended after it completes
 * (or on error). The span name defaults to the method name in lower-hyphen-case,
 * unless overridden by {@link #name()} or {@link #value()}.
 *
 * <p>Example:
 * <pre>{@code
 * @NewSpan("get-user")
 * public User getUser(@SpanTag("user.id") String userId) {
 *     return userRepository.findById(userId);
 * }
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.NewSpan}
 *
 * @see ContinueSpan
 * @see SpanTag
 */
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Target(ElementType.METHOD)
public @interface NewSpan {

    /**
     * The span name. If empty, falls back to {@link #value()}, then to the method name.
     * @return span name, or empty string
     */
    String name() default "";

    /**
     * Alias for {@link #name()}.
     * @return span name, or empty string
     */
    String value() default "";

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/ContinueSpan.java`

```java
package io.simpletracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation that <b>continues the current span</b> rather than creating a new one.
 * Use this to enrich an existing span with tags (via {@link SpanTag}) and lifecycle
 * events, without introducing an extra child span.
 *
 * <p>If no span is currently in scope when the method is invoked, a new span is
 * created as a fallback (same behavior as {@link NewSpan}).
 *
 * <p>The optional {@link #log()} attribute provides a prefix for lifecycle events
 * recorded on the span: {@code "<log>.before"}, {@code "<log>.after"}, and
 * {@code "<log>.afterFailure"}.
 *
 * <p>Example:
 * <pre>{@code
 * @ContinueSpan(log = "validate-user")
 * public void validateUser(@SpanTag("user.name") String name) {
 *     // enriches the current span instead of creating a new one
 * }
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.ContinueSpan}
 *
 * @see NewSpan
 * @see SpanTag
 */
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Target(ElementType.METHOD)
public @interface ContinueSpan {

    /**
     * Optional prefix for lifecycle events. When set, the processor records
     * {@code "<log>.before"}, {@code "<log>.after"}, and on failure
     * {@code "<log>.afterFailure"} events on the span.
     * @return log prefix, or empty string
     */
    String log() default "";

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/SpanTag.java`

```java
package io.simpletracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation placed on method parameters to automatically tag the current span
 * with the parameter's value. Used together with {@link NewSpan} or
 * {@link ContinueSpan}.
 *
 * <p>The tag key is determined by {@link #value()} (preferred) or {@link #key()}.
 * The tag value is the parameter's {@link Object#toString()} result.
 *
 * <p>Example:
 * <pre>{@code
 * @NewSpan("get-user")
 * public User getUser(@SpanTag("user.id") String userId) {
 *     return userRepository.findById(userId);
 * }
 * }</pre>
 *
 * <p><b>Simplification</b>: The real framework supports custom {@code ValueResolver}
 * and SpEL expressions for computing the tag value. This simplified version always
 * uses {@code toString()} — sufficient for learning the annotation processing pattern.
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.SpanTag}
 *
 * @see NewSpan
 * @see ContinueSpan
 */
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Target(ElementType.PARAMETER)
public @interface SpanTag {

    /**
     * The tag key name. Takes precedence over {@link #key()}.
     * @return tag key, or empty string
     */
    String value() default "";

    /**
     * Alias for {@link #value()}.
     * @return tag key, or empty string
     */
    String key() default "";

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/NewSpanParser.java`

```java
package io.simpletracing.annotation;

import io.simpletracing.Span;
import org.aopalliance.intercept.MethodInvocation;

/**
 * Strategy interface for parsing a {@link NewSpan} annotation and applying its
 * configuration (primarily the span name) to a {@link Span}.
 *
 * <p>The default implementation ({@link DefaultNewSpanParser}) converts the annotation's
 * name (or the method name) to lower-hyphen-case. Custom implementations can override
 * this to apply different naming conventions.
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.NewSpanParser}
 *
 * @see DefaultNewSpanParser
 */
public interface NewSpanParser {

    /**
     * Parses the {@link NewSpan} annotation and configures the given span accordingly.
     *
     * @param methodInvocation the intercepted method invocation
     * @param newSpan the annotation (may be null if the span was created as a fallback
     *                for a {@link ContinueSpan} with no current span)
     * @param span the span to configure
     */
    void parse(MethodInvocation methodInvocation, NewSpan newSpan, Span span);

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/DefaultNewSpanParser.java`

```java
package io.simpletracing.annotation;

import io.simpletracing.Span;
import org.aopalliance.intercept.MethodInvocation;

/**
 * Default implementation of {@link NewSpanParser} that names the span using:
 * <ol>
 *   <li>{@link NewSpan#name()} — if set</li>
 *   <li>{@link NewSpan#value()} — if name is empty but value is set</li>
 *   <li>The Java method name — if both are empty</li>
 * </ol>
 *
 * <p>The resolved name is converted to lower-hyphen-case (e.g., {@code "getUserById"}
 * becomes {@code "get-user-by-id"}). This matches the naming convention used by
 * distributed tracing systems like Zipkin and OpenTelemetry.
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.DefaultNewSpanParser}
 *
 * @see NewSpanParser
 */
public class DefaultNewSpanParser implements NewSpanParser {

    @Override
    public void parse(MethodInvocation methodInvocation, NewSpan newSpan, Span span) {
        String name = spanName(newSpan, methodInvocation);
        span.name(toLowerHyphen(name));
    }

    private String spanName(NewSpan newSpan, MethodInvocation invocation) {
        if (newSpan == null) {
            return invocation.getMethod().getName();
        }
        String name = newSpan.name();
        String value = newSpan.value();
        if (!name.isEmpty()) {
            return name;
        }
        if (!value.isEmpty()) {
            return value;
        }
        return invocation.getMethod().getName();
    }

    /**
     * Converts a camelCase or PascalCase string to lower-hyphen-case.
     * For example: "getUserById" → "get-user-by-id".
     *
     * <p>Maps to: {@code io.micrometer.tracing.internal.SpanNameUtil.toLowerHyphen}
     */
    static String toLowerHyphen(String name) {
        if (name.isEmpty()) {
            return name;
        }
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < name.length(); i++) {
            char c = name.charAt(i);
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

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/SpanTagAnnotationHandler.java`

```java
package io.simpletracing.annotation;

import io.simpletracing.SpanCustomizer;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

/**
 * Processes {@link SpanTag} annotations on method parameters, extracting tag key-value
 * pairs and applying them to a {@link SpanCustomizer}.
 *
 * <p>For each parameter annotated with {@code @SpanTag}, this handler:
 * <ol>
 *   <li>Resolves the tag <b>key</b> from {@link SpanTag#value()} (preferred) or
 *       {@link SpanTag#key()}</li>
 *   <li>Resolves the tag <b>value</b> by calling {@code toString()} on the argument
 *       (or {@code ""} if null)</li>
 *   <li>Applies the tag via {@link SpanCustomizer#tag(String, String)}</li>
 * </ol>
 *
 * <p><b>Simplification</b>: The real framework supports custom {@code ValueResolver}
 * and SpEL expressions for computing the tag value. This simplified version always
 * uses {@code toString()}, which is sufficient for learning the annotation-processing
 * pattern.
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.SpanTagAnnotationHandler}
 */
public class SpanTagAnnotationHandler {

    /**
     * Scans the given method's parameters for {@link SpanTag} annotations and applies
     * any found tags to the given span customizer.
     *
     * @param method the annotated method
     * @param arguments the actual argument values passed to the method
     * @param customizer the span customizer to tag
     */
    public void addAnnotatedParameters(Method method, Object[] arguments, SpanCustomizer customizer) {
        Annotation[][] parameterAnnotations = method.getParameterAnnotations();
        for (int i = 0; i < parameterAnnotations.length; i++) {
            for (Annotation annotation : parameterAnnotations[i]) {
                if (annotation instanceof SpanTag spanTag) {
                    String key = resolveKey(spanTag);
                    if (!key.isEmpty()) {
                        String value = resolveValue(arguments[i]);
                        customizer.tag(key, value);
                    }
                }
            }
        }
    }

    private String resolveKey(SpanTag spanTag) {
        if (!spanTag.value().isEmpty()) {
            return spanTag.value();
        }
        return spanTag.key();
    }

    private String resolveValue(Object argument) {
        if (argument == null) {
            return "";
        }
        return argument.toString();
    }

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/MethodInvocationProcessor.java`

```java
package io.simpletracing.annotation;

import org.aopalliance.intercept.MethodInvocation;

/**
 * Strategy interface for processing annotated method invocations intercepted by
 * {@link SpanAspect}. The aspect delegates all tracing logic to this processor,
 * keeping the aspect thin and focused on interception only.
 *
 * <p>Exactly one of {@code newSpan} or {@code continueSpan} will be non-null:
 * <ul>
 *   <li>{@code newSpan} non-null → create a new child span, run the method, end the span</li>
 *   <li>{@code continueSpan} non-null → enrich the current span, run the method</li>
 * </ul>
 *
 * <p>The default implementation is {@link ImperativeMethodInvocationProcessor} for
 * synchronous methods. The real framework also has a reactive variant (not built here).
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.MethodInvocationProcessor}
 *
 * @see ImperativeMethodInvocationProcessor
 */
public interface MethodInvocationProcessor {

    /**
     * Processes the intercepted method invocation under the appropriate tracing context.
     *
     * @param invocation the method invocation to proceed
     * @param newSpan the @NewSpan annotation, or null if this is a @ContinueSpan call
     * @param continueSpan the @ContinueSpan annotation, or null if this is a @NewSpan call
     * @return the method's return value
     * @throws Throwable if the method throws
     */
    Object process(MethodInvocation invocation, NewSpan newSpan, ContinueSpan continueSpan)
            throws Throwable;

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/ImperativeMethodInvocationProcessor.java`

```java
package io.simpletracing.annotation;

import io.simpletracing.Span;
import io.simpletracing.SpanCustomizer;
import io.simpletracing.Tracer;
import org.aopalliance.intercept.MethodInvocation;

/**
 * Synchronous implementation of {@link MethodInvocationProcessor}. Handles both
 * {@link NewSpan} and {@link ContinueSpan} annotated methods by creating, scoping,
 * and ending spans around the method invocation.
 *
 * <p><b>For {@code @NewSpan}</b>:
 * <ol>
 *   <li>Creates a new span via {@code tracer.nextSpan()}</li>
 *   <li>Names it using {@link NewSpanParser}</li>
 *   <li>Starts it and places it in scope</li>
 *   <li>Processes {@link SpanTag} parameters via {@link SpanTagAnnotationHandler}</li>
 *   <li>Proceeds with the method</li>
 *   <li>On success: ends the span. On error: records the error, then ends the span.</li>
 * </ol>
 *
 * <p><b>For {@code @ContinueSpan}</b>:
 * <ol>
 *   <li>Gets the current span from the tracer (falls back to creating a new span if none)</li>
 *   <li>Optionally logs lifecycle events with the {@code log} prefix</li>
 *   <li>Processes {@link SpanTag} parameters</li>
 *   <li>Proceeds with the method</li>
 *   <li>Does NOT end the span (it belongs to the caller)</li>
 * </ol>
 *
 * <p><b>Simplification</b>: The real framework uses an {@code AbstractMethodInvocationProcessor}
 * base class with template methods for before/after/onFailure. We inline all logic here,
 * reducing one level of inheritance.
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.ImperativeMethodInvocationProcessor}
 *
 * @see MethodInvocationProcessor
 * @see SpanAspect
 */
public class ImperativeMethodInvocationProcessor implements MethodInvocationProcessor {

    private final NewSpanParser newSpanParser;

    private final Tracer tracer;

    private final SpanTagAnnotationHandler spanTagAnnotationHandler;

    public ImperativeMethodInvocationProcessor(NewSpanParser newSpanParser, Tracer tracer,
            SpanTagAnnotationHandler spanTagAnnotationHandler) {
        this.newSpanParser = newSpanParser;
        this.tracer = tracer;
        this.spanTagAnnotationHandler = spanTagAnnotationHandler;
    }

    public ImperativeMethodInvocationProcessor(NewSpanParser newSpanParser, Tracer tracer) {
        this(newSpanParser, tracer, new SpanTagAnnotationHandler());
    }

    @Override
    public Object process(MethodInvocation invocation, NewSpan newSpan, ContinueSpan continueSpan)
            throws Throwable {
        Span span = tracer.currentSpan();
        // Create a new span for @NewSpan, or as a fallback when @ContinueSpan has no current span
        boolean startNewSpan = newSpan != null || span == null;
        if (startNewSpan) {
            span = tracer.nextSpan();
            newSpanParser.parse(invocation, newSpan, span);
            span.start();
        }
        String log = logPrefix(continueSpan);
        boolean hasLog = !log.isEmpty();
        try (Tracer.SpanInScope ignored = tracer.withSpan(span)) {
            before(invocation, span, log, hasLog);
            return invocation.proceed();
        } catch (Exception ex) {
            onFailure(span, log, hasLog, ex);
            throw ex;
        } finally {
            after(span, startNewSpan, log, hasLog);
        }
    }

    private void before(MethodInvocation invocation, Span span, String log, boolean hasLog) {
        if (hasLog) {
            span.event(log + ".before");
        }
        addTags(invocation, span);
    }

    private void after(Span span, boolean startNewSpan, String log, boolean hasLog) {
        if (hasLog) {
            span.event(log + ".after");
        }
        if (startNewSpan) {
            span.end();
        }
    }

    private void onFailure(Span span, String log, boolean hasLog, Exception ex) {
        if (hasLog) {
            span.event(log + ".afterFailure");
        }
        span.error(ex);
    }

    private void addTags(MethodInvocation invocation, Span span) {
        spanTagAnnotationHandler.addAnnotatedParameters(
                invocation.getMethod(), invocation.getArguments(), span);
        // Add class and method tags for observability
        if (invocation.getThis() != null) {
            span.tag("class", invocation.getThis().getClass().getSimpleName());
        }
        span.tag("method", invocation.getMethod().getName());
    }

    private String logPrefix(ContinueSpan continueSpan) {
        if (continueSpan == null) {
            return "";
        }
        return continueSpan.log();
    }

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/SpanAspect.java`

```java
package io.simpletracing.annotation;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;

import java.lang.reflect.Method;

/**
 * AspectJ aspect that intercepts methods annotated with {@link NewSpan} or
 * {@link ContinueSpan} and delegates tracing logic to a {@link MethodInvocationProcessor}.
 *
 * <p>This aspect is deliberately thin — it only handles:
 * <ol>
 *   <li><b>Interception</b>: Two {@code @Around} advices match the two annotations</li>
 *   <li><b>Adaptation</b>: Wraps AspectJ's {@link ProceedingJoinPoint} into an
 *       {@link org.aopalliance.intercept.MethodInvocation} via {@link SpanAspectMethodInvocation}</li>
 *   <li><b>Delegation</b>: Passes the adapted invocation + annotation to the processor</li>
 * </ol>
 *
 * <p>All span creation, naming, tagging, and lifecycle management lives in the processor,
 * not here. This separation keeps the AspectJ-specific code minimal and testable.
 *
 * <p><b>Method resolution</b>: The {@link #getMethod(ProceedingJoinPoint)} helper resolves
 * the method from the <b>target class</b> (not the proxy/interface). This ensures that
 * annotations from the concrete class are found, even when AOP proxies are in use.
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.SpanAspect}
 *
 * @see MethodInvocationProcessor
 * @see ImperativeMethodInvocationProcessor
 */
@Aspect
public class SpanAspect {

    private final MethodInvocationProcessor methodInvocationProcessor;

    public SpanAspect(MethodInvocationProcessor methodInvocationProcessor) {
        this.methodInvocationProcessor = methodInvocationProcessor;
    }

    @Around("@annotation(io.simpletracing.annotation.ContinueSpan)")
    public Object continueSpanMethod(ProceedingJoinPoint pjp) throws Throwable {
        Method method = getMethod(pjp);
        ContinueSpan continueSpan = method.getAnnotation(ContinueSpan.class);
        return methodInvocationProcessor.process(
                new SpanAspectMethodInvocation(pjp, method), null, continueSpan);
    }

    @Around("@annotation(io.simpletracing.annotation.NewSpan)")
    public Object newSpanMethod(ProceedingJoinPoint pjp) throws Throwable {
        Method method = getMethod(pjp);
        NewSpan newSpan = method.getAnnotation(NewSpan.class);
        return methodInvocationProcessor.process(
                new SpanAspectMethodInvocation(pjp, method), newSpan, null);
    }

    private Method getMethod(ProceedingJoinPoint pjp) throws NoSuchMethodException {
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        return pjp.getTarget().getClass().getMethod(method.getName(), method.getParameterTypes());
    }

}
```

### `[NEW] simple-tracing-api/src/main/java/io/simpletracing/annotation/SpanAspectMethodInvocation.java`

```java
package io.simpletracing.annotation;

import org.aopalliance.intercept.MethodInvocation;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.reflect.MethodSignature;

import java.lang.reflect.AccessibleObject;
import java.lang.reflect.Method;

/**
 * Adapter that wraps AspectJ's {@link ProceedingJoinPoint} into the AOP Alliance
 * {@link MethodInvocation} interface. This bridges the gap between AspectJ (used by
 * {@link SpanAspect}) and the AOP Alliance abstraction (used by
 * {@link MethodInvocationProcessor}).
 *
 * <p>This adapter is necessary because the real framework's annotation processing
 * is built around the standard {@code aopalliance} API, not the AspectJ API. This
 * allows the same {@link MethodInvocationProcessor} to work with both AspectJ and
 * Spring AOP (which also uses the aopalliance API).
 *
 * <p>Maps to: {@code io.micrometer.tracing.annotation.SpanAspectMethodInvocation}
 */
class SpanAspectMethodInvocation implements MethodInvocation {

    private final ProceedingJoinPoint pjp;

    private final Method method;

    SpanAspectMethodInvocation(ProceedingJoinPoint pjp, Method method) {
        this.pjp = pjp;
        this.method = method;
    }

    @Override
    public Method getMethod() {
        return this.method;
    }

    @Override
    public Object[] getArguments() {
        return this.pjp.getArgs();
    }

    @Override
    public Object proceed() throws Throwable {
        return this.pjp.proceed();
    }

    @Override
    public Object getThis() {
        return this.pjp.getTarget();
    }

    @Override
    public AccessibleObject getStaticPart() {
        return this.method;
    }

}
```

### `[MODIFIED] simple-tracing-api/build.gradle`

```groovy
// Core API module — defines the tracing facade interfaces
// Maps to: micrometer-tracing in the real repository
//
// This module has NO implementation — only interfaces and annotations.
// Concrete implementations live in simple-tracing-test (in-memory)
// and simple-tracing-bridge-brave (Brave adapter).

dependencies {
    // Observation API — needed for TracingObservationHandler (Feature 9)
    api 'io.micrometer:micrometer-observation:1.15.0'

    // AOP Alliance — needed for annotation-based tracing (Feature 10)
    api 'aopalliance:aopalliance:1.0'
    implementation 'org.aspectj:aspectjweaver:1.9.22.1'
}
```

### `[NEW] simple-tracing-api/src/test/java/io/simpletracing/AnnotationTracingTest.java`

```java
package io.simpletracing;

import io.simpletracing.annotation.ContinueSpan;
import io.simpletracing.annotation.DefaultNewSpanParser;
import io.simpletracing.annotation.ImperativeMethodInvocationProcessor;
import io.simpletracing.annotation.NewSpan;
import io.simpletracing.annotation.SpanTag;
import io.simpletracing.annotation.SpanTagAnnotationHandler;
import org.aopalliance.intercept.MethodInvocation;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.AccessibleObject;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Client-perspective tests for the annotation-based tracing system.
 *
 * <p>These tests exercise the {@link ImperativeMethodInvocationProcessor} directly,
 * bypassing the AspectJ aspect. This is the same testing approach used in the real
 * Micrometer Tracing framework — the aspect is thin enough that testing the processor
 * directly validates the complete behavior.
 */
class AnnotationTracingTest {

    private TestTracer tracer;

    private ImperativeMethodInvocationProcessor processor;

    @BeforeEach
    void setUp() {
        tracer = new TestTracer();
        processor = new ImperativeMethodInvocationProcessor(
                new DefaultNewSpanParser(), tracer, new SpanTagAnnotationHandler());
    }

    // ── @NewSpan tests ─────────────────────────────────────────────────────

    @Test
    void shouldCreateNewSpan_WhenNewSpanAnnotationPresent() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "getUser", new Class<?>[] { String.class },
                new AnnotatedService(), new Object[] { "user-42" });
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        Object result = processor.process(invocation, newSpan, null);

        assertThat(result).isEqualTo("User:user-42");
        assertThat(tracer.finishedSpans()).hasSize(1);
        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.name).isEqualTo("get-user");
        assertThat(span.tags).containsEntry("user.id", "user-42");
        assertThat(span.tags).containsEntry("class", "AnnotatedService");
        assertThat(span.tags).containsEntry("method", "getUser");
    }

    @Test
    void shouldUseLowerHyphenMethodName_WhenNewSpanHasNoName() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "processOrder", new Class<?>[] {},
                new AnnotatedService(), new Object[] {});
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        processor.process(invocation, newSpan, null);

        assertThat(tracer.finishedSpans()).hasSize(1);
        assertThat(tracer.finishedSpans().get(0).name).isEqualTo("process-order");
    }

    @Test
    void shouldUseValueAttribute_WhenNewSpanNameIsEmpty() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "withValueAttribute", new Class<?>[] {},
                new AnnotatedService(), new Object[] {});
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        processor.process(invocation, newSpan, null);

        assertThat(tracer.finishedSpans()).hasSize(1);
        assertThat(tracer.finishedSpans().get(0).name).isEqualTo("custom-value-name");
    }

    @Test
    void shouldCreateChildSpan_WhenParentSpanExists() throws Throwable {
        Span parent = tracer.nextSpan().name("parent").start();
        try (Tracer.SpanInScope parentScope = tracer.withSpan(parent)) {
            MethodInvocation invocation = createInvocation(
                    AnnotatedService.class, "processOrder", new Class<?>[] {},
                    new AnnotatedService(), new Object[] {});
            NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

            processor.process(invocation, newSpan, null);
        } finally {
            parent.end();
        }

        assertThat(tracer.finishedSpans()).hasSize(2);
        TestTracer.TestSpanData childSpan = tracer.finishedSpans().get(0);
        TestTracer.TestSpanData parentSpan = tracer.finishedSpans().get(1);
        assertThat(childSpan.context.parentId()).isEqualTo(parentSpan.context.spanId());
        assertThat(childSpan.context.traceId()).isEqualTo(parentSpan.context.traceId());
    }

    @Test
    void shouldRecordErrorAndEndSpan_WhenMethodThrows() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "failingMethod", new Class<?>[] {},
                new AnnotatedService(), new Object[] {});
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        assertThatThrownBy(() -> processor.process(invocation, newSpan, null))
                .isInstanceOf(RuntimeException.class)
                .hasMessage("something went wrong");

        assertThat(tracer.finishedSpans()).hasSize(1);
        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.name).isEqualTo("failing-method");
        assertThat(span.error).isNotNull();
        assertThat(span.error.getMessage()).isEqualTo("something went wrong");
    }

    // ── @ContinueSpan tests ────────────────────────────────────────────────

    @Test
    void shouldEnrichCurrentSpan_WhenContinueSpanAnnotationPresent() throws Throwable {
        Span existing = tracer.nextSpan().name("existing-operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(existing)) {
            MethodInvocation invocation = createInvocation(
                    AnnotatedService.class, "validateUser", new Class<?>[] { String.class },
                    new AnnotatedService(), new Object[] { "Alice" });
            ContinueSpan continueSpan = invocation.getMethod().getAnnotation(ContinueSpan.class);

            processor.process(invocation, null, continueSpan);
        } finally {
            existing.end();
        }

        assertThat(tracer.finishedSpans()).hasSize(1);
        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.name).isEqualTo("existing-operation");
        assertThat(span.tags).containsEntry("user.name", "Alice");
    }

    @Test
    void shouldLogLifecycleEvents_WhenContinueSpanHasLogAttribute() throws Throwable {
        Span existing = tracer.nextSpan().name("operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(existing)) {
            MethodInvocation invocation = createInvocation(
                    AnnotatedService.class, "validateUser", new Class<?>[] { String.class },
                    new AnnotatedService(), new Object[] { "Bob" });
            ContinueSpan continueSpan = invocation.getMethod().getAnnotation(ContinueSpan.class);

            processor.process(invocation, null, continueSpan);
        } finally {
            existing.end();
        }

        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.events).contains("validate-user.before", "validate-user.after");
    }

    @Test
    void shouldCreateFallbackSpan_WhenContinueSpanHasNoCurrentSpan() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "validateUser", new Class<?>[] { String.class },
                new AnnotatedService(), new Object[] { "Charlie" });
        ContinueSpan continueSpan = invocation.getMethod().getAnnotation(ContinueSpan.class);

        processor.process(invocation, null, continueSpan);

        assertThat(tracer.finishedSpans()).hasSize(1);
        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.tags).containsEntry("user.name", "Charlie");
    }

    @Test
    void shouldRecordErrorOnContinuedSpan_WhenMethodThrows() throws Throwable {
        Span existing = tracer.nextSpan().name("operation").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(existing)) {
            MethodInvocation invocation = createInvocation(
                    AnnotatedService.class, "failingContinueMethod", new Class<?>[] {},
                    new AnnotatedService(), new Object[] {});
            ContinueSpan continueSpan = invocation.getMethod().getAnnotation(ContinueSpan.class);

            assertThatThrownBy(() -> processor.process(invocation, null, continueSpan))
                    .isInstanceOf(RuntimeException.class);
        } finally {
            existing.end();
        }

        assertThat(tracer.finishedSpans()).hasSize(1);
        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.error).isNotNull();
        assertThat(span.events).contains("failing.afterFailure");
    }

    // ── @SpanTag tests ─────────────────────────────────────────────────────

    @Test
    void shouldTagSpanFromMultipleParameters() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "multiTag",
                new Class<?>[] { String.class, int.class },
                new AnnotatedService(), new Object[] { "prod", 42 });
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        processor.process(invocation, newSpan, null);

        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.tags).containsEntry("env", "prod");
        assertThat(span.tags).containsEntry("count", "42");
    }

    @Test
    void shouldUseKeyAttribute_WhenValueIsEmpty() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "withKeyAttribute", new Class<?>[] { String.class },
                new AnnotatedService(), new Object[] { "alice@example.com" });
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        processor.process(invocation, newSpan, null);

        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.tags).containsEntry("user.email", "alice@example.com");
    }

    @Test
    void shouldTagEmptyString_WhenParameterIsNull() throws Throwable {
        MethodInvocation invocation = createInvocation(
                AnnotatedService.class, "getUser", new Class<?>[] { String.class },
                new AnnotatedService(), new Object[] { null });
        NewSpan newSpan = invocation.getMethod().getAnnotation(NewSpan.class);

        processor.process(invocation, newSpan, null);

        TestTracer.TestSpanData span = tracer.finishedSpans().get(0);
        assertThat(span.tags).containsEntry("user.id", "");
    }

    // ── Helper: annotated service ──────────────────────────────────────────

    public static class AnnotatedService {

        @NewSpan("get-user")
        public String getUser(@SpanTag("user.id") String userId) {
            return "User:" + userId;
        }

        @NewSpan
        public String processOrder() {
            return "processed";
        }

        @NewSpan(value = "custom-value-name")
        public void withValueAttribute() {
        }

        @NewSpan("failing-method")
        public void failingMethod() {
            throw new RuntimeException("something went wrong");
        }

        @ContinueSpan(log = "validate-user")
        public void validateUser(@SpanTag("user.name") String name) {
        }

        @ContinueSpan(log = "failing")
        public void failingContinueMethod() {
            throw new RuntimeException("validation failed");
        }

        @NewSpan("multi-tag")
        public void multiTag(@SpanTag("env") String environment, @SpanTag("count") int count) {
        }

        @NewSpan("with-key")
        public void withKeyAttribute(@SpanTag(key = "user.email") String email) {
        }
    }

    // ── Helper: create MethodInvocation from annotated method ──────────────

    private static MethodInvocation createInvocation(Class<?> targetClass, String methodName,
            Class<?>[] paramTypes, Object target, Object[] args) throws NoSuchMethodException {
        Method method = targetClass.getMethod(methodName, paramTypes);
        return new MethodInvocation() {
            @Override
            public Method getMethod() {
                return method;
            }

            @Override
            public Object[] getArguments() {
                return args;
            }

            @Override
            public Object proceed() throws Throwable {
                try {
                    return method.invoke(target, args);
                } catch (InvocationTargetException e) {
                    throw e.getCause();
                }
            }

            @Override
            public Object getThis() {
                return target;
            }

            @Override
            public AccessibleObject getStaticPart() {
                return method;
            }
        };
    }

}
```
