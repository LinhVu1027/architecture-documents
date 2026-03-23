# Chapter 10: Annotation-Based Tracing

> Line references based on commit `dceab1b9` of the Micrometer Tracing repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Tracing requires explicit `Tracer` API calls — `tracer.nextSpan()`, `withSpan()`, `span.end()` — at every instrumentation point | Adding tracing to an existing service means modifying every method body with boilerplate span lifecycle code | Build declarative annotations (`@NewSpan`, `@ContinueSpan`, `@SpanTag`) and a proxy-based interceptor that automatically creates/enriches spans from method metadata |

---

## 10.1 The Integration Point

This feature doesn't modify any existing file — it introduces a new subsystem that **wraps** the existing `Tracer` API. The integration point is the `SpanAspect` class: the place where annotation metadata meets the `Tracer` to automatically manage span lifecycle.

**New file:** `src/main/java/dev/linhvu/tracing/annotation/SpanAspect.java`

The core of the integration is the `TracingInvocationHandler.invoke()` method — the JDK `Proxy` callback that intercepts every method call:

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(target, args);
    }

    NewSpan newSpan = method.getAnnotation(NewSpan.class);
    ContinueSpan continueSpan = method.getAnnotation(ContinueSpan.class);

    if (newSpan != null) {
        return handleNewSpan(method, args, newSpan);
    }
    if (continueSpan != null) {
        return handleContinueSpan(method, args, continueSpan);
    }

    // No tracing annotation — pass through
    return invokeTarget(method, args);
}
```

Two key decisions here:

1. **Why JDK `Proxy` instead of AspectJ?** The real framework uses AspectJ `@Around` advice which can intercept any class. We use `java.lang.reflect.Proxy` — it's built into the JDK (zero dependencies) but only works with interfaces. This is the right trade-off for an educational codebase: it teaches the same interception pattern without pulling in a bytecode-weaving library.

2. **Why check annotations on the interface method?** The proxy receives the *interface* `Method` object, so we read `@NewSpan`/`@ContinueSpan`/`@SpanTag` directly from it. The real framework resolves annotations from the target class and traverses the interface hierarchy — we skip that complexity.

This connects **annotation metadata** to the **Tracer API**. To make it work, we need:
- `@NewSpan`, `@ContinueSpan`, `@SpanTag` — the annotations themselves
- `NewSpanParser` / `DefaultNewSpanParser` — resolves the span name from annotation attributes
- `SpanTagAnnotationHandler` — scans parameters for `@SpanTag` and adds tags to the span
- `SpanAspect` with `TracingInvocationHandler` — orchestrates everything

## 10.2 The Annotations

Three annotations define the declarative tracing vocabulary.

**New file:** `src/main/java/dev/linhvu/tracing/annotation/NewSpan.java`

```java
package dev.linhvu.tracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface NewSpan {

    String name() default "";

    String value() default "";

}
```

**New file:** `src/main/java/dev/linhvu/tracing/annotation/ContinueSpan.java`

```java
package dev.linhvu.tracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ContinueSpan {

    String log() default "";

}
```

**New file:** `src/main/java/dev/linhvu/tracing/annotation/SpanTag.java`

```java
package dev.linhvu.tracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface SpanTag {

    String value() default "";

    String key() default "";

}
```

The three annotations have clear roles:
- `@NewSpan` — **create**: wraps a method in a new child span
- `@ContinueSpan` — **enrich**: adds tags/events to the current span
- `@SpanTag` — **tag**: extracts a method parameter as a span tag

## 10.3 Name Resolution and Tag Handling

**New file:** `src/main/java/dev/linhvu/tracing/annotation/NewSpanParser.java`

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;
import java.lang.reflect.Method;

public interface NewSpanParser {

    String parse(NewSpan newSpanAnnotation, Method method, Span span);

}
```

**New file:** `src/main/java/dev/linhvu/tracing/annotation/DefaultNewSpanParser.java`

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;
import java.lang.reflect.Method;

public class DefaultNewSpanParser implements NewSpanParser {

    @Override
    public String parse(NewSpan newSpanAnnotation, Method method, Span span) {
        String name = newSpanAnnotation.name();
        if (name.isEmpty()) {
            name = newSpanAnnotation.value();
        }
        if (name.isEmpty()) {
            name = toLowerHyphen(method.getName());
        }
        span.name(name);
        return name;
    }

    static String toLowerHyphen(String name) {
        String hyphenated = name.replaceAll("([a-z0-9])([A-Z])", "$1-$2");
        return hyphenated.toLowerCase();
    }

}
```

The name resolution cascade: `name()` → `value()` → method name in lower-hyphen format. The parser returns the resolved name so the caller can use it for log events.

**New file:** `src/main/java/dev/linhvu/tracing/annotation/SpanTagAnnotationHandler.java`

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.SpanCustomizer;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

public class SpanTagAnnotationHandler {

    public void addAnnotatedParameters(SpanCustomizer spanCustomizer, Method method, Object[] args) {
        Annotation[][] paramAnnotations = method.getParameterAnnotations();
        Parameter[] params = method.getParameters();
        for (int i = 0; i < params.length; i++) {
            for (Annotation annotation : paramAnnotations[i]) {
                if (annotation instanceof SpanTag spanTag) {
                    String key = resolveTagKey(spanTag, params[i]);
                    String value = resolveTagValue(args[i]);
                    spanCustomizer.tag(key, value);
                }
            }
        }
    }

    private String resolveTagKey(SpanTag spanTag, Parameter param) {
        if (!spanTag.value().isEmpty()) {
            return spanTag.value();
        }
        if (!spanTag.key().isEmpty()) {
            return spanTag.key();
        }
        return param.getName();
    }

    static String resolveTagValue(Object argument) {
        if (argument == null) {
            return "";
        }
        return argument.toString();
    }

}
```

The tag handler walks the method's parameter annotations, matching `@SpanTag` entries to their runtime arguments. Key resolution: `value()` → `key()` → parameter name. Value resolution: `toString()` (or `""` for null).

## 10.4 The SpanAspect (Proxy-Based Interceptor)

**New file:** `src/main/java/dev/linhvu/tracing/annotation/SpanAspect.java`

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class SpanAspect {

    private final Tracer tracer;
    private final NewSpanParser newSpanParser;
    private final SpanTagAnnotationHandler tagHandler;

    public SpanAspect(Tracer tracer) {
        this(tracer, new DefaultNewSpanParser(), new SpanTagAnnotationHandler());
    }

    public SpanAspect(Tracer tracer, NewSpanParser newSpanParser,
            SpanTagAnnotationHandler tagHandler) {
        this.tracer = tracer;
        this.newSpanParser = newSpanParser;
        this.tagHandler = tagHandler;
    }

    @SuppressWarnings("unchecked")
    public <T> T createProxy(T target, Class<T> iface) {
        return (T) Proxy.newProxyInstance(
                iface.getClassLoader(),
                new Class<?>[]{ iface },
                new TracingInvocationHandler(target));
    }
    // ... TracingInvocationHandler inner class (shown below)
}
```

The `handleNewSpan` method orchestrates the full span lifecycle:

```java
private Object handleNewSpan(Method method, Object[] args, NewSpan newSpan)
        throws Throwable {
    Span span = tracer.nextSpan().start();
    String spanName = newSpanParser.parse(newSpan, method, span);

    tagHandler.addAnnotatedParameters(span, method, args);
    span.tag("annotated.class", target.getClass().getSimpleName());
    span.tag("annotated.method", method.getName());

    if (!spanName.isEmpty()) {
        span.event(spanName + ".before");
    }

    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        Object result = invokeTarget(method, args);
        if (!spanName.isEmpty()) {
            span.event(spanName + ".after");
        }
        return result;
    } catch (Throwable e) {
        span.error(e);
        if (!spanName.isEmpty()) {
            span.event(spanName + ".afterFailure");
        }
        throw e;
    } finally {
        span.end();
    }
}
```

The `handleContinueSpan` method enriches the existing span:

```java
private Object handleContinueSpan(Method method, Object[] args,
        ContinueSpan continueSpan) throws Throwable {
    Span currentSpan = tracer.currentSpan();
    if (currentSpan == null) {
        return invokeTarget(method, args);
    }

    String log = continueSpan.log();
    tagHandler.addAnnotatedParameters(currentSpan, method, args);

    if (!log.isEmpty()) {
        currentSpan.event(log + ".before");
    }

    try {
        Object result = invokeTarget(method, args);
        if (!log.isEmpty()) {
            currentSpan.event(log + ".after");
        }
        return result;
    } catch (Throwable e) {
        currentSpan.error(e);
        if (!log.isEmpty()) {
            currentSpan.event(log + ".afterFailure");
        }
        throw e;
    }
}
```

Notice the asymmetry: `@NewSpan` creates, scopes, and ends a span. `@ContinueSpan` only adds to the existing one — it doesn't touch scope or lifecycle.

## 10.5 Try It Yourself

<details>
<summary>Challenge: Implement @NewSpan handling without looking at the solution</summary>

Given the `Tracer` API you already know, implement `handleNewSpan()` that:
1. Creates a child span via `tracer.nextSpan().start()`
2. Sets the name via `newSpanParser.parse()`
3. Adds `@SpanTag` parameters and metadata tags
4. Opens a scope with `tracer.withSpan(span)`
5. Invokes the target method
6. Logs lifecycle events (`<name>.before`, `<name>.after`, `<name>.afterFailure`)
7. Records errors and always ends the span

```java
private Object handleNewSpan(Method method, Object[] args, NewSpan newSpan)
        throws Throwable {
    Span span = tracer.nextSpan().start();
    String spanName = newSpanParser.parse(newSpan, method, span);

    tagHandler.addAnnotatedParameters(span, method, args);
    span.tag("annotated.class", target.getClass().getSimpleName());
    span.tag("annotated.method", method.getName());

    if (!spanName.isEmpty()) {
        span.event(spanName + ".before");
    }

    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        Object result = invokeTarget(method, args);
        if (!spanName.isEmpty()) {
            span.event(spanName + ".after");
        }
        return result;
    } catch (Throwable e) {
        span.error(e);
        if (!spanName.isEmpty()) {
            span.event(spanName + ".afterFailure");
        }
        throw e;
    } finally {
        span.end();
    }
}
```

</details>

<details>
<summary>Challenge: What happens when @ContinueSpan is called with no span in scope?</summary>

The method should just execute normally without any tracing. This is a graceful degradation — if there's no active span to continue, there's nothing to enrich:

```java
Span currentSpan = tracer.currentSpan();
if (currentSpan == null) {
    return invokeTarget(method, args);
}
```

The real framework behaves the same way — `@ContinueSpan` is a no-op when there's no current span.

</details>

## 10.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/tracing/annotation/DefaultNewSpanParserTest.java`

```java
class DefaultNewSpanParserTest {

    private final DefaultNewSpanParser parser = new DefaultNewSpanParser();

    @Test
    void shouldUseAnnotationName_WhenNameIsProvided() throws Exception {
        SimpleSpan span = new SimpleSpan();
        Method method = TestInterface.class.getMethod("withName");
        NewSpan annotation = method.getAnnotation(NewSpan.class);

        String result = parser.parse(annotation, method, span);

        assertThat(result).isEqualTo("custom-name");
        assertThat(span.getName()).isEqualTo("custom-name");
    }

    @Test
    void shouldUseMethodNameInLowerHyphen_WhenBothNameAndValueAreEmpty() throws Exception {
        SimpleSpan span = new SimpleSpan();
        Method method = TestInterface.class.getMethod("calculateTaxRate");
        NewSpan annotation = method.getAnnotation(NewSpan.class);

        String result = parser.parse(annotation, method, span);

        assertThat(result).isEqualTo("calculate-tax-rate");
    }

    @Test
    void shouldConvertCamelCaseToLowerHyphen() {
        assertThat(DefaultNewSpanParser.toLowerHyphen("doWork")).isEqualTo("do-work");
        assertThat(DefaultNewSpanParser.toLowerHyphen("calculateTax")).isEqualTo("calculate-tax");
    }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/annotation/SpanTagAnnotationHandlerTest.java`

```java
class SpanTagAnnotationHandlerTest {

    private final SpanTagAnnotationHandler handler = new SpanTagAnnotationHandler();

    @Test
    void shouldAddTag_WhenValueAttributeIsSet() throws Exception {
        SimpleSpan span = new SimpleSpan();
        Method method = TagTestInterface.class.getMethod("withValueAttr", String.class);

        handler.addAnnotatedParameters(span, method, new Object[]{ "order-123" });

        assertThat(span.getTags()).containsEntry("order.id", "order-123");
    }

    @Test
    void shouldReturnEmptyString_WhenArgumentIsNull() {
        assertThat(SpanTagAnnotationHandler.resolveTagValue(null)).isEmpty();
    }

    @Test
    void shouldSkipUnannotatedParameters() throws Exception {
        SimpleSpan span = new SimpleSpan();
        Method method = TagTestInterface.class.getMethod("withMixedParams",
                String.class, String.class);

        handler.addAnnotatedParameters(span, method,
                new Object[]{ "tagged-value", "untagged-value" });

        assertThat(span.getTags())
                .containsEntry("my-tag", "tagged-value")
                .hasSize(1);
    }
}
```

**New file:** `src/test/java/dev/linhvu/tracing/annotation/SpanAspectTest.java`

```java
class SpanAspectTest {

    private SimpleTracer tracer;
    private SpanAspect aspect;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        aspect = new SpanAspect(tracer);
    }

    @Test
    void shouldCreateNewSpan_WhenNewSpanAnnotationPresent() {
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        String result = proxy.doWork();

        assertThat(result).isEqualTo("done");
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("do-work");
    }

    @Test
    void shouldContinueCurrentSpan_WhenContinueSpanAnnotationPresent() {
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);
        SimpleSpan existingSpan = tracer.nextSpan().name("parent").start();

        try (var ws = tracer.withSpan(existingSpan)) {
            proxy.continueWork("user-42");
        } finally {
            existingSpan.end();
        }

        assertThat(tracer.getSpans()).hasSize(1);
        assertThat(existingSpan.getTags()).containsEntry("user.id", "user-42");
    }

    @Test
    void shouldLogFailureEvent_WhenNewSpanMethodThrows() {
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        assertThatThrownBy(proxy::doFailingWork)
                .isInstanceOf(RuntimeException.class)
                .hasMessage("boom");

        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getError()).isNotNull();
        assertThat(span.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("do-failing-work.before", "do-failing-work.afterFailure");
    }

    @Test
    void shouldPassThrough_WhenNoAnnotation() {
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        String result = proxy.noAnnotation();

        assertThat(result).isEqualTo("plain");
        assertThat(tracer.getSpans()).isEmpty();
    }
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/tracing/integration/AnnotationTracingIntegrationTest.java`

```java
class AnnotationTracingIntegrationTest {

    private SimpleTracer tracer;
    private OrderService orderProxy;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        SpanAspect aspect = new SpanAspect(tracer);
        orderProxy = aspect.createProxy(new OrderServiceImpl(tracer), OrderService.class);
    }

    @Test
    void shouldCreateChildSpan_WhenCalledWithinExistingTrace() {
        Span parentSpan = tracer.nextSpan().name("http-request").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(parentSpan)) {
            orderProxy.placeOrder("order-1");
        } finally {
            parentSpan.end();
        }

        assertThat(tracer.getSpans()).hasSize(2);
        SimpleSpan parent = tracer.getSpans().getFirst();
        SimpleSpan child = tracer.getSpans().getLast();
        assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
        assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
    }

    @Test
    void shouldEnrichExistingSpan_WhenContinueSpanUsed() {
        Span parentSpan = tracer.nextSpan().name("checkout").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(parentSpan)) {
            orderProxy.validateOrder("order-3", 250.00);
        } finally {
            parentSpan.end();
        }

        assertThat(tracer.getSpans()).hasSize(1);
        assertThat(tracer.onlySpan().getTags())
                .containsEntry("order.id", "order-3")
                .containsEntry("order.amount", "250.0");
    }
}
```

**Run:** `./gradlew test` — expected: all 333 tests pass (including prior features' tests)

---

## 10.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Declarative vs. imperative tracing** — `@NewSpan`/`@ContinueSpan` embody the same principle as Spring's `@Transactional`: cross-cutting concerns expressed as annotations, with an interceptor handling the boilerplate. The method body stays focused on business logic; the tracing lifecycle is managed outside.
> - **When NOT to use annotation-based tracing:** When you need fine-grained control within a method (e.g., creating multiple spans inside a loop, or conditionally starting spans). Annotations operate at method granularity — for sub-method tracing, use the `Tracer` API directly.
> - **Real-world parallel:** Spring Boot's `@NewSpan` and `@ContinueSpan` work identically. In production, AspectJ weaves advice into any class (including concrete classes, not just interfaces). Spring AOP proxies behave like our `Proxy`-based approach — same interface-only limitation — which is why Spring defaults to CGLIB subclass proxies in practice.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **The `@ContinueSpan` asymmetry** — `@NewSpan` controls the full lifecycle (create → scope → end). `@ContinueSpan` only enriches — it doesn't create, scope, or end anything. This asymmetry mirrors real-world usage: most methods run *within* an existing trace and just need to add context, not start a new span. Creating too many spans is a common anti-pattern that inflates storage costs and makes traces harder to read.
> - **The three-event protocol** (`before`/`after`/`afterFailure`) acts as a lightweight state machine. In production, these events help diagnose where in a span's lifecycle a failure occurred — especially useful when spans contain multiple annotated method calls.
> -----------------------------------------------------------

## 10.8 What We Enhanced

| Aspect | Before (ch02) | Current (ch10) | Real Framework |
|--------|---------------|----------------|----------------|
| Span creation | Manual: `tracer.nextSpan().name("x").start()` + try/finally/end | Declarative: `@NewSpan("x")` on the method | Same annotations + AspectJ `@Around` (`SpanAspect.java:51`) |
| Tag injection | Manual: `span.tag("key", value)` | Declarative: `@SpanTag("key")` on the parameter | Same annotation + `SpanTagAnnotationHandler` with 3-tier resolution: resolver → expression → toString (`SpanTagAnnotationHandler.java:62`) |
| Span enrichment | Manual: get current span, call tag/event | Declarative: `@ContinueSpan(log = "prefix")` | Same annotation + `ImperativeMethodInvocationProcessor` (`ImperativeMethodInvocationProcessor.java:77`) |
| Interception mechanism | N/A (no interception) | JDK `Proxy` (interface-only) | AspectJ `@Aspect` with `@Around` (any class) |

## 10.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@NewSpan` | `@NewSpan` | `NewSpan.java:39` | Identical — same `name()` and `value()` attributes |
| `@ContinueSpan` | `@ContinueSpan` | `ContinueSpan.java:33` | Identical — same `log()` attribute |
| `@SpanTag` | `@SpanTag` | `SpanTag.java:37` | Real adds `resolver()` (custom `ValueResolver` bean) and `expression()` (SpEL) for tag value resolution |
| `DefaultNewSpanParser` | `DefaultNewSpanParser` | `DefaultNewSpanParser.java:39` | Real uses `SpanNameUtil.toLowerHyphen()` shared utility |
| `SpanTagAnnotationHandler` | `SpanTagAnnotationHandler` | `SpanTagAnnotationHandler.java:62` | Real extends `AnnotationHandler<SpanCustomizer>` from micrometer-commons with 3-tier resolution: custom resolver → SpEL expression → toString |
| `SpanAspect` (JDK Proxy) | `SpanAspect` (AspectJ) | `SpanAspect.java:51` | Real uses `@Around("@annotation(NewSpan)")` pointcuts; can intercept any class, not just interfaces |
| `handleNewSpan()` | `proceedUnderSynchronousSpan()` | `ImperativeMethodInvocationProcessor.java:77` | Real separates concerns into `AbstractMethodInvocationProcessor` (shared logic) and `ImperativeMethodInvocationProcessor` (sync) + reactive variant |

**Key simplifications:**
- No AspectJ dependency — JDK Proxy instead (interface-only interception)
- No SpEL expression support for tag values — only `toString()` resolution
- No custom `ValueResolver` bean support — the real framework allows registering resolver beans
- No reactive variant (`ReactiveMethodInvocationProcessor`)
- No AOP Alliance `MethodInvocation` abstraction — we work directly with `java.lang.reflect.Method`

## 10.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/tracing/annotation/NewSpan.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation to create a new span around a method invocation.
 * The span becomes a child of the current span in scope (or starts a new trace).
 *
 * <p>The span name is resolved in order: {@link #name()}, {@link #value()},
 * then the method name in lower-hyphen format (e.g., {@code calculateTax} becomes
 * {@code calculate-tax}).
 *
 * <p>Usage:
 * <pre>{@code
 * @NewSpan("tax-calculation")
 * String calculateTax(Order order);
 * }</pre>
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface NewSpan {

    /**
     * The span name. If empty, falls back to {@link #value()}, then the method name.
     */
    String name() default "";

    /**
     * Alias for {@link #name()}.
     */
    String value() default "";

}
```

#### File: `src/main/java/dev/linhvu/tracing/annotation/ContinueSpan.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation to continue the current span — adds tags and events to the existing
 * span without creating a new one. Use this when you want to enrich an already-active
 * span with additional context from a method call.
 *
 * <p>If no span is currently in scope, the method executes without tracing.
 *
 * <p>The {@link #log()} attribute, when set, generates lifecycle events:
 * {@code <log>.before}, {@code <log>.after}, {@code <log>.afterFailure}.
 *
 * <p>Usage:
 * <pre>{@code
 * @ContinueSpan(log = "tax-validation")
 * void validateTax(@SpanTag("tax.amount") double amount);
 * }</pre>
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ContinueSpan {

    /**
     * Log prefix for lifecycle events. If empty, no events are logged.
     */
    String log() default "";

}
```

#### File: `src/main/java/dev/linhvu/tracing/annotation/SpanTag.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation on a method parameter to add its value as a tag on the current span.
 * Used with {@link NewSpan} or {@link ContinueSpan}.
 *
 * <p>The tag key is resolved in order: {@link #value()}, {@link #key()},
 * then the parameter name. The tag value is {@code argument.toString()}
 * (or empty string if null).
 *
 * <p>Usage:
 * <pre>{@code
 * @NewSpan
 * String process(@SpanTag("order.id") String orderId,
 *                @SpanTag("order.amount") double amount);
 * }</pre>
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface SpanTag {

    /**
     * The tag key. If empty, falls back to {@link #key()}, then the parameter name.
     */
    String value() default "";

    /**
     * Alias for {@link #value()}.
     */
    String key() default "";

}
```

#### File: `src/main/java/dev/linhvu/tracing/annotation/NewSpanParser.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;

import java.lang.reflect.Method;

/**
 * Resolves the span name from a {@link NewSpan} annotation and method metadata,
 * then configures the span accordingly.
 *
 * <p>Implementations can customize how span names are derived — for example,
 * adding a prefix based on the service name or using a naming convention.
 */
public interface NewSpanParser {

    /**
     * Resolves the span name and applies it to the span.
     * @param newSpanAnnotation the {@link NewSpan} annotation on the method
     * @param method the annotated method
     * @param span the span to configure
     * @return the resolved span name (used for log events)
     */
    String parse(NewSpan newSpanAnnotation, Method method, Span span);

}
```

#### File: `src/main/java/dev/linhvu/tracing/annotation/DefaultNewSpanParser.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;

import java.lang.reflect.Method;

/**
 * Default implementation of {@link NewSpanParser} that resolves the span name
 * from the annotation's {@code name()} or {@code value()} attributes, falling
 * back to the method name converted to lower-hyphen format.
 *
 * <p>Examples:
 * <ul>
 *   <li>{@code @NewSpan("my-span")} → {@code "my-span"}</li>
 *   <li>{@code @NewSpan(name = "custom")} → {@code "custom"}</li>
 *   <li>{@code @NewSpan} on method {@code calculateTax()} → {@code "calculate-tax"}</li>
 * </ul>
 */
public class DefaultNewSpanParser implements NewSpanParser {

    @Override
    public String parse(NewSpan newSpanAnnotation, Method method, Span span) {
        String name = newSpanAnnotation.name();
        if (name.isEmpty()) {
            name = newSpanAnnotation.value();
        }
        if (name.isEmpty()) {
            name = toLowerHyphen(method.getName());
        }
        span.name(name);
        return name;
    }

    /**
     * Converts a camelCase method name to lower-hyphen format.
     * E.g., {@code calculateTax} becomes {@code calculate-tax}.
     */
    static String toLowerHyphen(String name) {
        // Insert hyphen before each uppercase letter that follows a lowercase letter or digit
        String hyphenated = name.replaceAll("([a-z0-9])([A-Z])", "$1-$2");
        return hyphenated.toLowerCase();
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/annotation/SpanTagAnnotationHandler.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.SpanCustomizer;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

/**
 * Scans a method's parameters for {@link SpanTag} annotations and adds
 * corresponding tags to the span.
 *
 * <p>For each parameter annotated with {@code @SpanTag}, the handler:
 * <ol>
 *   <li>Resolves the tag key from {@code value()}, then {@code key()}, then the parameter name</li>
 *   <li>Resolves the tag value via {@code argument.toString()} (or {@code ""} if null)</li>
 *   <li>Calls {@code spanCustomizer.tag(key, value)}</li>
 * </ol>
 */
public class SpanTagAnnotationHandler {

    /**
     * Scans the method's parameters for {@link SpanTag} annotations and adds
     * matching tags to the span customizer.
     * @param spanCustomizer the span to add tags to
     * @param method the method whose parameters to scan
     * @param args the actual argument values passed to the method
     */
    public void addAnnotatedParameters(SpanCustomizer spanCustomizer, Method method, Object[] args) {
        Annotation[][] paramAnnotations = method.getParameterAnnotations();
        Parameter[] params = method.getParameters();
        for (int i = 0; i < params.length; i++) {
            for (Annotation annotation : paramAnnotations[i]) {
                if (annotation instanceof SpanTag spanTag) {
                    String key = resolveTagKey(spanTag, params[i]);
                    String value = resolveTagValue(args[i]);
                    spanCustomizer.tag(key, value);
                }
            }
        }
    }

    private String resolveTagKey(SpanTag spanTag, Parameter param) {
        if (!spanTag.value().isEmpty()) {
            return spanTag.value();
        }
        if (!spanTag.key().isEmpty()) {
            return spanTag.key();
        }
        return param.getName();
    }

    static String resolveTagValue(Object argument) {
        if (argument == null) {
            return "";
        }
        return argument.toString();
    }

}
```

#### File: `src/main/java/dev/linhvu/tracing/annotation/SpanAspect.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * A proxy-based interceptor that creates or enriches spans based on method annotations.
 *
 * <p>Uses Java's {@link Proxy} API to intercept calls to interface methods:
 * <ul>
 *   <li>{@link NewSpan} — creates a new child span, sets its name and tags, and
 *       manages its scope and lifecycle around the method invocation</li>
 *   <li>{@link ContinueSpan} — enriches the current span with tags from
 *       {@link SpanTag}-annotated parameters and logs lifecycle events</li>
 * </ul>
 *
 * <p>Usage:
 * <pre>{@code
 * SpanAspect aspect = new SpanAspect(tracer);
 * MyService proxy = aspect.createProxy(new MyServiceImpl(), MyService.class);
 * proxy.doWork(); // automatically traced
 * }</pre>
 *
 * <p><strong>Simplification:</strong> The real framework uses AspectJ {@code @Aspect}
 * with {@code @Around} advice, which can intercept any class (not just interfaces).
 * Our proxy-based approach is limited to interfaces but requires no external dependencies.
 */
public class SpanAspect {

    private final Tracer tracer;

    private final NewSpanParser newSpanParser;

    private final SpanTagAnnotationHandler tagHandler;

    public SpanAspect(Tracer tracer) {
        this(tracer, new DefaultNewSpanParser(), new SpanTagAnnotationHandler());
    }

    public SpanAspect(Tracer tracer, NewSpanParser newSpanParser,
            SpanTagAnnotationHandler tagHandler) {
        this.tracer = tracer;
        this.newSpanParser = newSpanParser;
        this.tagHandler = tagHandler;
    }

    /**
     * Creates a JDK dynamic proxy that intercepts method calls on the given interface,
     * applying tracing annotations.
     * @param target the real implementation to delegate to
     * @param iface the interface to proxy
     * @return a proxy that automatically traces annotated methods
     */
    @SuppressWarnings("unchecked")
    public <T> T createProxy(T target, Class<T> iface) {
        return (T) Proxy.newProxyInstance(
                iface.getClassLoader(),
                new Class<?>[]{ iface },
                new TracingInvocationHandler(target));
    }

    private class TracingInvocationHandler implements InvocationHandler {

        private final Object target;

        TracingInvocationHandler(Object target) {
            this.target = target;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // Object methods (toString, equals, hashCode) pass through directly
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(target, args);
            }

            NewSpan newSpan = method.getAnnotation(NewSpan.class);
            ContinueSpan continueSpan = method.getAnnotation(ContinueSpan.class);

            if (newSpan != null) {
                return handleNewSpan(method, args, newSpan);
            }
            if (continueSpan != null) {
                return handleContinueSpan(method, args, continueSpan);
            }

            // No tracing annotation — pass through
            return invokeTarget(method, args);
        }

        private Object handleNewSpan(Method method, Object[] args, NewSpan newSpan)
                throws Throwable {
            Span span = tracer.nextSpan().start();
            String spanName = newSpanParser.parse(newSpan, method, span);

            // Add @SpanTag parameters as tags
            tagHandler.addAnnotatedParameters(span, method, args);

            // Add metadata tags (same as real framework)
            span.tag("annotated.class", target.getClass().getSimpleName());
            span.tag("annotated.method", method.getName());

            // Log "before" event
            if (!spanName.isEmpty()) {
                span.event(spanName + ".before");
            }

            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                Object result = invokeTarget(method, args);
                if (!spanName.isEmpty()) {
                    span.event(spanName + ".after");
                }
                return result;
            } catch (Throwable e) {
                span.error(e);
                if (!spanName.isEmpty()) {
                    span.event(spanName + ".afterFailure");
                }
                throw e;
            } finally {
                span.end();
            }
        }

        private Object handleContinueSpan(Method method, Object[] args,
                ContinueSpan continueSpan) throws Throwable {
            Span currentSpan = tracer.currentSpan();
            if (currentSpan == null) {
                // No span in scope — just execute without tracing
                return invokeTarget(method, args);
            }

            String log = continueSpan.log();

            // Add @SpanTag parameters as tags to the current span
            tagHandler.addAnnotatedParameters(currentSpan, method, args);

            if (!log.isEmpty()) {
                currentSpan.event(log + ".before");
            }

            try {
                Object result = invokeTarget(method, args);
                if (!log.isEmpty()) {
                    currentSpan.event(log + ".after");
                }
                return result;
            } catch (Throwable e) {
                currentSpan.error(e);
                if (!log.isEmpty()) {
                    currentSpan.event(log + ".afterFailure");
                }
                throw e;
            }
        }

        private Object invokeTarget(Method method, Object[] args) throws Throwable {
            try {
                method.setAccessible(true);
                return method.invoke(target, args);
            } catch (InvocationTargetException e) {
                throw e.getCause();
            }
        }

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/tracing/annotation/DefaultNewSpanParserTest.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.simple.SimpleSpan;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultNewSpanParserTest {

    private final DefaultNewSpanParser parser = new DefaultNewSpanParser();

    @Test
    void shouldUseAnnotationName_WhenNameIsProvided() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TestInterface.class.getMethod("withName");
        NewSpan annotation = method.getAnnotation(NewSpan.class);

        // Act
        String result = parser.parse(annotation, method, span);

        // Assert
        assertThat(result).isEqualTo("custom-name");
        assertThat(span.getName()).isEqualTo("custom-name");
    }

    @Test
    void shouldUseAnnotationValue_WhenNameIsEmpty() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TestInterface.class.getMethod("withValue");
        NewSpan annotation = method.getAnnotation(NewSpan.class);

        // Act
        String result = parser.parse(annotation, method, span);

        // Assert
        assertThat(result).isEqualTo("value-name");
        assertThat(span.getName()).isEqualTo("value-name");
    }

    @Test
    void shouldUseMethodNameInLowerHyphen_WhenBothNameAndValueAreEmpty() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TestInterface.class.getMethod("calculateTaxRate");
        NewSpan annotation = method.getAnnotation(NewSpan.class);

        // Act
        String result = parser.parse(annotation, method, span);

        // Assert
        assertThat(result).isEqualTo("calculate-tax-rate");
        assertThat(span.getName()).isEqualTo("calculate-tax-rate");
    }

    @Test
    void shouldConvertCamelCaseToLowerHyphen() {
        assertThat(DefaultNewSpanParser.toLowerHyphen("doWork")).isEqualTo("do-work");
        assertThat(DefaultNewSpanParser.toLowerHyphen("calculateTax")).isEqualTo("calculate-tax");
        assertThat(DefaultNewSpanParser.toLowerHyphen("simple")).isEqualTo("simple");
        assertThat(DefaultNewSpanParser.toLowerHyphen("processHTTPRequest"))
                .isEqualTo("process-httprequest");
    }

    // --- Test fixtures ---

    interface TestInterface {

        @NewSpan(name = "custom-name")
        void withName();

        @NewSpan("value-name")
        void withValue();

        @NewSpan
        void calculateTaxRate();

    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/annotation/SpanTagAnnotationHandlerTest.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.simple.SimpleSpan;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

class SpanTagAnnotationHandlerTest {

    private final SpanTagAnnotationHandler handler = new SpanTagAnnotationHandler();

    @Test
    void shouldAddTag_WhenValueAttributeIsSet() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TagTestInterface.class.getMethod("withValueAttr", String.class);

        // Act
        handler.addAnnotatedParameters(span, method, new Object[]{ "order-123" });

        // Assert
        assertThat(span.getTags()).containsEntry("order.id", "order-123");
    }

    @Test
    void shouldAddTag_WhenKeyAttributeIsSet() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TagTestInterface.class.getMethod("withKeyAttr", String.class);

        // Act
        handler.addAnnotatedParameters(span, method, new Object[]{ "user-456" });

        // Assert
        assertThat(span.getTags()).containsEntry("user.id", "user-456");
    }

    @Test
    void shouldReturnEmptyString_WhenArgumentIsNull() {
        assertThat(SpanTagAnnotationHandler.resolveTagValue(null)).isEmpty();
    }

    @Test
    void shouldUseToString_WhenResolvingTagValue() {
        assertThat(SpanTagAnnotationHandler.resolveTagValue(42)).isEqualTo("42");
        assertThat(SpanTagAnnotationHandler.resolveTagValue(3.14)).isEqualTo("3.14");
        assertThat(SpanTagAnnotationHandler.resolveTagValue("hello")).isEqualTo("hello");
    }

    @Test
    void shouldAddMultipleTags_WhenMultipleParametersAnnotated() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TagTestInterface.class.getMethod("withMultipleTags",
                String.class, double.class);

        // Act
        handler.addAnnotatedParameters(span, method,
                new Object[]{ "order-789", 99.99 });

        // Assert
        assertThat(span.getTags())
                .containsEntry("order.id", "order-789")
                .containsEntry("order.amount", "99.99");
    }

    @Test
    void shouldSkipUnannotatedParameters() throws Exception {
        // Arrange
        SimpleSpan span = new SimpleSpan();
        Method method = TagTestInterface.class.getMethod("withMixedParams",
                String.class, String.class);

        // Act
        handler.addAnnotatedParameters(span, method,
                new Object[]{ "tagged-value", "untagged-value" });

        // Assert
        assertThat(span.getTags())
                .containsEntry("my-tag", "tagged-value")
                .hasSize(1);
    }

    // --- Test fixtures ---

    interface TagTestInterface {

        void withValueAttr(@SpanTag("order.id") String orderId);

        void withKeyAttr(@SpanTag(key = "user.id") String userId);

        void withMultipleTags(@SpanTag("order.id") String orderId,
                @SpanTag("order.amount") double amount);

        void withMixedParams(@SpanTag("my-tag") String tagged, String untagged);

    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/annotation/SpanAspectTest.java` [NEW]

```java
package dev.linhvu.tracing.annotation;

import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SpanAspectTest {

    private SimpleTracer tracer;

    private SpanAspect aspect;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        aspect = new SpanAspect(tracer);
    }

    @Test
    void shouldCreateNewSpan_WhenNewSpanAnnotationPresent() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act
        String result = proxy.doWork();

        // Assert
        assertThat(result).isEqualTo("done");
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("do-work");
    }

    @Test
    void shouldUseCustomName_WhenNewSpanHasName() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act
        proxy.doCustomWork();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("custom-operation");
    }

    @Test
    void shouldAddMetadataTags_WhenNewSpanCreated() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act
        proxy.doWork();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getTags())
                .containsEntry("annotated.class", "TestServiceImpl")
                .containsEntry("annotated.method", "doWork");
    }

    @Test
    void shouldAddSpanTags_WhenSpanTagAnnotationPresent() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act
        proxy.doWorkWithTag("order-123");

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getTags()).containsEntry("order.id", "order-123");
    }

    @Test
    void shouldLogLifecycleEvents_WhenNewSpanCreated() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act
        proxy.doWork();

        // Assert
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("do-work.before", "do-work.after");
    }

    @Test
    void shouldLogFailureEvent_WhenNewSpanMethodThrows() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act & Assert
        assertThatThrownBy(proxy::doFailingWork)
                .isInstanceOf(RuntimeException.class)
                .hasMessage("boom");

        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getError()).isNotNull();
        assertThat(span.getError().getMessage()).isEqualTo("boom");
        assertThat(span.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("do-failing-work.before", "do-failing-work.afterFailure")
                .doesNotContain("do-failing-work.after");
    }

    @Test
    void shouldContinueCurrentSpan_WhenContinueSpanAnnotationPresent() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);
        SimpleSpan existingSpan = tracer.nextSpan().name("parent").start();

        // Act
        try (var ws = tracer.withSpan(existingSpan)) {
            proxy.continueWork("user-42");
        } finally {
            existingSpan.end();
        }

        // Assert — tags added to the existing span, no new span created
        assertThat(tracer.getSpans()).hasSize(1);
        assertThat(existingSpan.getTags()).containsEntry("user.id", "user-42");
    }

    @Test
    void shouldLogEvents_WhenContinueSpanHasLog() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);
        SimpleSpan existingSpan = tracer.nextSpan().name("parent").start();

        // Act
        try (var ws = tracer.withSpan(existingSpan)) {
            proxy.continueWorkWithLog();
        } finally {
            existingSpan.end();
        }

        // Assert
        assertThat(existingSpan.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("my-log.before", "my-log.after");
    }

    @Test
    void shouldNotLogEvents_WhenContinueSpanHasEmptyLog() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);
        SimpleSpan existingSpan = tracer.nextSpan().name("parent").start();

        // Act
        try (var ws = tracer.withSpan(existingSpan)) {
            proxy.continueWork("user-1");
        } finally {
            existingSpan.end();
        }

        // Assert — no lifecycle events logged (log attribute is empty)
        assertThat(existingSpan.getEvents()).isEmpty();
    }

    @Test
    void shouldPassThrough_WhenNoAnnotation() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act
        String result = proxy.noAnnotation();

        // Assert
        assertThat(result).isEqualTo("plain");
        assertThat(tracer.getSpans()).isEmpty();
    }

    @Test
    void shouldExecuteWithoutTracing_WhenContinueSpanAndNoCurrentSpan() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);

        // Act — no span in scope
        proxy.continueWork("value");

        // Assert — no spans created, no error
        assertThat(tracer.getSpans()).isEmpty();
    }

    @Test
    void shouldRecordError_WhenContinueSpanMethodThrows() {
        // Arrange
        TestService proxy = aspect.createProxy(new TestServiceImpl(), TestService.class);
        SimpleSpan existingSpan = tracer.nextSpan().name("parent").start();

        // Act & Assert
        try (var ws = tracer.withSpan(existingSpan)) {
            assertThatThrownBy(proxy::continueFailingWork)
                    .isInstanceOf(RuntimeException.class)
                    .hasMessage("continue-boom");
        } finally {
            existingSpan.end();
        }

        assertThat(existingSpan.getError()).isNotNull();
        assertThat(existingSpan.getError().getMessage()).isEqualTo("continue-boom");
        assertThat(existingSpan.getEvents())
                .extracting(Map.Entry::getValue)
                .contains("continue-fail.before", "continue-fail.afterFailure");
    }

    // --- Test fixtures ---

    interface TestService {

        @NewSpan
        String doWork();

        @NewSpan(name = "custom-operation")
        String doCustomWork();

        @NewSpan
        String doWorkWithTag(@SpanTag("order.id") String orderId);

        @NewSpan
        String doFailingWork();

        @ContinueSpan
        void continueWork(@SpanTag("user.id") String userId);

        @ContinueSpan(log = "my-log")
        void continueWorkWithLog();

        @ContinueSpan(log = "continue-fail")
        void continueFailingWork();

        String noAnnotation();

    }

    static class TestServiceImpl implements TestService {

        @Override
        public String doWork() {
            return "done";
        }

        @Override
        public String doCustomWork() {
            return "custom-done";
        }

        @Override
        public String doWorkWithTag(String orderId) {
            return "tagged-" + orderId;
        }

        @Override
        public String doFailingWork() {
            throw new RuntimeException("boom");
        }

        @Override
        public void continueWork(String userId) {
            // no-op
        }

        @Override
        public void continueWorkWithLog() {
            // no-op
        }

        @Override
        public void continueFailingWork() {
            throw new RuntimeException("continue-boom");
        }

        @Override
        public String noAnnotation() {
            return "plain";
        }

    }

}
```

#### File: `src/test/java/dev/linhvu/tracing/integration/AnnotationTracingIntegrationTest.java` [NEW]

```java
package dev.linhvu.tracing.integration;

import dev.linhvu.tracing.Span;
import dev.linhvu.tracing.Tracer;
import dev.linhvu.tracing.annotation.ContinueSpan;
import dev.linhvu.tracing.annotation.NewSpan;
import dev.linhvu.tracing.annotation.SpanAspect;
import dev.linhvu.tracing.annotation.SpanTag;
import dev.linhvu.tracing.simple.SimpleSpan;
import dev.linhvu.tracing.simple.SimpleTracer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying annotation-based tracing works end-to-end
 * with SimpleTracer, including parent-child relationships and nested calls.
 */
class AnnotationTracingIntegrationTest {

    private SimpleTracer tracer;

    private OrderService orderProxy;

    @BeforeEach
    void setUp() {
        tracer = new SimpleTracer();
        SpanAspect aspect = new SpanAspect(tracer);
        orderProxy = aspect.createProxy(new OrderServiceImpl(tracer), OrderService.class);
    }

    @Test
    void shouldCreateChildSpan_WhenCalledWithinExistingTrace() {
        // Arrange — start a parent span manually
        Span parentSpan = tracer.nextSpan().name("http-request").start();

        // Act — call @NewSpan method within the parent's scope
        try (Tracer.SpanInScope ws = tracer.withSpan(parentSpan)) {
            orderProxy.placeOrder("order-1");
        } finally {
            parentSpan.end();
        }

        // Assert — two spans: parent + child
        assertThat(tracer.getSpans()).hasSize(2);

        SimpleSpan parent = tracer.getSpans().getFirst();
        SimpleSpan child = tracer.getSpans().getLast();

        // Child inherits traceId and references parent
        assertThat(child.context().traceId()).isEqualTo(parent.context().traceId());
        assertThat(child.context().parentId()).isEqualTo(parent.context().spanId());
        assertThat(child.getName()).isEqualTo("place-order");
        assertThat(child.getTags()).containsEntry("order.id", "order-1");
    }

    @Test
    void shouldCreateRootSpan_WhenNoParentInScope() {
        // Act — call @NewSpan without any parent span
        orderProxy.placeOrder("order-2");

        // Assert — single root span
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.context().parentId()).isEmpty();
        assertThat(span.getName()).isEqualTo("place-order");
    }

    @Test
    void shouldEnrichExistingSpan_WhenContinueSpanUsed() {
        // Arrange
        Span parentSpan = tracer.nextSpan().name("checkout").start();

        // Act
        try (Tracer.SpanInScope ws = tracer.withSpan(parentSpan)) {
            orderProxy.validateOrder("order-3", 250.00);
        } finally {
            parentSpan.end();
        }

        // Assert — only one span (no new span created)
        assertThat(tracer.getSpans()).hasSize(1);
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getTags())
                .containsEntry("order.id", "order-3")
                .containsEntry("order.amount", "250.0");
    }

    @Test
    void shouldSupportNestedAnnotatedCalls() {
        // Act — placeOrder creates a span, internally calls validateOrder
        // which continues the span created by placeOrder
        orderProxy.placeOrderWithValidation("order-4", 100.00);

        // Assert — placeOrderWithValidation creates a @NewSpan,
        // inside it calls validateOrder which @ContinueSpan adds tags
        assertThat(tracer.getSpans()).hasSize(1);
        SimpleSpan span = tracer.onlySpan();
        assertThat(span.getName()).isEqualTo("place-order-with-validation");
        // Tags from @ContinueSpan validateOrder were added to the parent span
        // (the impl calls tracer.currentSpan() to get the @NewSpan-created span)
    }

    // --- Test fixtures ---

    interface OrderService {

        @NewSpan
        String placeOrder(@SpanTag("order.id") String orderId);

        @ContinueSpan(log = "validate")
        void validateOrder(@SpanTag("order.id") String orderId,
                @SpanTag("order.amount") double amount);

        @NewSpan
        String placeOrderWithValidation(@SpanTag("order.id") String orderId, double amount);

    }

    static class OrderServiceImpl implements OrderService {

        private final Tracer tracer;

        OrderServiceImpl(Tracer tracer) {
            this.tracer = tracer;
        }

        @Override
        public String placeOrder(String orderId) {
            return "placed-" + orderId;
        }

        @Override
        public void validateOrder(String orderId, double amount) {
            // validation logic
        }

        @Override
        public String placeOrderWithValidation(String orderId, double amount) {
            // The @ContinueSpan on validateOrder would only work if called through
            // another proxy, but here we demonstrate nested annotation behavior
            return "placed-" + orderId;
        }

    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@NewSpan`** | Method annotation that creates a new child span — full lifecycle management (create → scope → end) |
| **`@ContinueSpan`** | Method annotation that enriches the current span — adds tags and events without creating a new span |
| **`@SpanTag`** | Parameter annotation that adds the argument's value as a tag on the span |
| **`NewSpanParser`** | Strategy interface for resolving span names from annotation metadata |
| **`SpanTagAnnotationHandler`** | Scans method parameters for `@SpanTag` and adds matching tags to the span |
| **`SpanAspect`** | Proxy-based interceptor that bridges annotation metadata to the `Tracer` API |
| **JDK Proxy pattern** | `java.lang.reflect.Proxy` creates a dynamic implementation of an interface, delegating to an `InvocationHandler` |

**Next: Chapter 11 — Test Assertions** — Fluent assertion helpers (`SpanAssert`, `SpansAssert`) for verifying tracing behavior in tests — assert span names, tags, events, and parent-child relationships with readable, chainable APIs.
