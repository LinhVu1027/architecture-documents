# Chapter 11: JSON Response Body

> **Build Challenge**
>
> | Current State | Limitation | Objective |
> |---|---|---|
> | DispatcherServlet routes requests to `@Controller` methods and writes `text/plain` responses (Feature 10) | Every return value is serialized via `toString()` with `text/plain` — no structured data format, no content-type negotiation, no way to build JSON APIs | Detect `@ResponseBody`/`@RestController` on handler methods, serialize return values as JSON via `HttpMessageConverter` + Jackson, and fall back to plain text for non-annotated methods |

---

## 11.1 The Integration Point: RequestMappingHandlerAdapter.handle() Branches on @ResponseBody

The integration point is `RequestMappingHandlerAdapter.handle()` — specifically the return value handling block. This is where the adapter checks `hasResponseBody(handler)` and branches between JSON serialization via converters and the existing plain-text `toString()` path.

Two subsystems connect here: **the annotation detection system** (`@ResponseBody`, `@RestController` via `AnnotationUtils`) and **the serialization pipeline** (`HttpMessageConverter`, `JacksonHttpMessageConverter`).

**Modifying:** `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingHandlerAdapter.java`
**Change:** Added `List<HttpMessageConverter> messageConverters` field, constructor taking converters, `hasResponseBody()` method, `writeWithMessageConverters()` method, and branching logic in `handle()`

The `handle()` method now reads:

```java
@Override
public void handle(HttpServletRequest request, HttpServletResponse response,
                   HandlerMethod handler) throws Exception {
    Method method = handler.getMethod();
    method.setAccessible(true);

    // Resolve all method parameters
    Object[] args = resolveArguments(method, request, response);

    // Invoke the handler method
    Object result = method.invoke(handler.getBean(), args);

    // Handle return value — write to response if not void and response not committed
    if (result != null && !response.isCommitted()) {
        if (hasResponseBody(handler)) {
            writeWithMessageConverters(result, response);
        } else {
            response.setContentType("text/plain;charset=UTF-8");
            response.getWriter().write(result.toString());
            response.getWriter().flush();
        }
    }
}
```

**Before this feature**, the return value block was unconditional — always `toString()` with `text/plain`. **After this feature**, the adapter first asks "should this go through converters?" and, if so, delegates to the first converter that can handle the return type.

```diff
 // Handle return value — write to response if not void and response not committed
 if (result != null && !response.isCommitted()) {
-    response.setContentType("text/plain;charset=UTF-8");
-    response.getWriter().write(result.toString());
-    response.getWriter().flush();
+    if (hasResponseBody(handler)) {
+        writeWithMessageConverters(result, response);
+    } else {
+        response.setContentType("text/plain;charset=UTF-8");
+        response.getWriter().write(result.toString());
+        response.getWriter().flush();
+    }
 }
```

The two new methods that support this branching:

```java
private boolean hasResponseBody(HandlerMethod handler) {
    // Check method-level @ResponseBody
    if (handler.getMethod().isAnnotationPresent(ResponseBody.class)) {
        return true;
    }
    // Check class-level @ResponseBody (covers @RestController)
    return AnnotationUtils.hasAnnotation(handler.getBean().getClass(), ResponseBody.class);
}

private void writeWithMessageConverters(Object result, HttpServletResponse response)
        throws IOException {
    for (HttpMessageConverter converter : messageConverters) {
        if (converter.canWrite(result.getClass())) {
            converter.write(result, response);
            return;
        }
    }
    // Fallback: no converter matched — write as plain text
    response.setContentType("text/plain;charset=UTF-8");
    response.getWriter().write(result.toString());
    response.getWriter().flush();
}
```

The second integration point is in the `DispatcherServlet`. Its `initHandlerAdapters()` method now discovers `HttpMessageConverter` beans from the `ApplicationContext` and passes them to the adapter.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/web/servlet/DispatcherServlet.java`
**Change:** `initHandlerAdapters()` now finds converters and passes them to the adapter constructor

```java
private void initHandlerAdapters() {
    List<HttpMessageConverter> converters = findMessageConverters();
    this.handlerAdapter = new RequestMappingHandlerAdapter(converters);
}

private List<HttpMessageConverter> findMessageConverters() {
    try {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) applicationContext).getBeanFactory();
        Map<String, HttpMessageConverter> converterBeans =
                factory.getBeansOfType(HttpMessageConverter.class);
        return new ArrayList<>(converterBeans.values());
    } catch (Exception e) {
        return List.of();
    }
}
```

```diff
 private void initHandlerAdapters() {
-    this.handlerAdapter = new RequestMappingHandlerAdapter();
+    List<HttpMessageConverter> converters = findMessageConverters();
+    this.handlerAdapter = new RequestMappingHandlerAdapter(converters);
 }
```

**Direction:** From this integration point, we need four things: (1) `@ResponseBody` and `@RestController` annotations, (2) `HttpMessageConverter` interface, (3) `JacksonHttpMessageConverter` implementation, and (4) `AnnotationUtils` for recursive meta-annotation detection so that `@RestController` (which carries `@ResponseBody` as a meta-annotation) is recognized.

---

## 11.2 The Annotations: @ResponseBody and @RestController

These annotations go in `iris-framework` (package `com.iris.framework.web.bind.annotation`) because they are part of the web framework layer.

### @ResponseBody -- marks a method or class for converter-based serialization

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ResponseBody {
}
```

When placed on a method, only that method's return value is serialized via converters. When placed on a class (or via `@RestController`), all handler methods in the class get response body semantics.

### @RestController -- composed annotation combining @Controller + @ResponseBody

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.stereotype.Controller;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    String value() default "";
}
```

`@RestController` is a *composed annotation*. Because it carries `@Controller` (which carries `@Component`), classes annotated with `@RestController` are automatically discovered by both component scanning and the handler mapping. Because it also carries `@ResponseBody`, every handler method automatically serializes its return value via message converters.

This is the key design question: how do we detect that a `@RestController`-annotated class has `@ResponseBody`? Java's `isAnnotationPresent()` only checks *direct* annotations. We need recursive meta-annotation detection -- which brings us to `AnnotationUtils` in section 11.4.

---

## 11.3 The Converter Abstraction: HttpMessageConverter and JacksonHttpMessageConverter

### HttpMessageConverter -- the strategy interface

This interface goes in `iris-framework` (package `com.iris.framework.web.http.converter`) because it defines the contract. Concrete implementations live in `iris-boot-core`.

```java
package com.iris.framework.web.http.converter;

import java.io.IOException;
import jakarta.servlet.http.HttpServletResponse;

public interface HttpMessageConverter {
    boolean canWrite(Class<?> clazz);
    void write(Object value, HttpServletResponse response) throws IOException;
}
```

Two methods, one concern: the converter decides whether it *can* serialize a given type (`canWrite`), and if so, *does* it (`write`). The adapter iterates registered converters and delegates to the first compatible one.

In the real Spring Framework, `HttpMessageConverter<T>` is generic and handles both reading (deserialization from `@RequestBody`) and writing (serialization for `@ResponseBody`), using `HttpInputMessage` and `HttpOutputMessage` abstractions instead of raw servlet request/response objects. It also uses `MediaType` for content negotiation. We keep only the write side and use `HttpServletResponse` directly.

### JacksonHttpMessageConverter -- the Jackson implementation

This class goes in `iris-boot-core` (package `com.iris.boot.web.http.converter`) because it depends on Jackson, which is a Boot-level dependency.

```java
package com.iris.boot.web.http.converter;

import java.io.IOException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.iris.framework.web.http.converter.HttpMessageConverter;
import jakarta.servlet.http.HttpServletResponse;

public class JacksonHttpMessageConverter implements HttpMessageConverter {

    private final ObjectMapper objectMapper;

    public JacksonHttpMessageConverter() {
        this(createDefaultObjectMapper());
    }

    public JacksonHttpMessageConverter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean canWrite(Class<?> clazz) {
        return true;
    }

    @Override
    public void write(Object value, HttpServletResponse response) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        String json = objectMapper.writeValueAsString(value);
        response.getWriter().write(json);
        response.getWriter().flush();
    }

    public ObjectMapper getObjectMapper() {
        return objectMapper;
    }

    private static ObjectMapper createDefaultObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        return mapper;
    }
}
```

`canWrite()` returns `true` for all classes because Jackson can serialize virtually any Java object. The `write()` method sets `Content-Type: application/json;charset=UTF-8`, serializes via `ObjectMapper.writeValueAsString()`, and flushes the writer.

The default `ObjectMapper` disables `FAIL_ON_EMPTY_BEANS` so that empty POJOs serialize to `{}` instead of throwing an exception. The constructor also accepts a custom `ObjectMapper` for cases where the application needs specific serialization settings.

This is the simplified equivalent of Spring's `MappingJackson2HttpMessageConverter`, which additionally handles content negotiation, JSON views (`@JsonView`), and anti-hijacking prefixes.

---

## 11.4 The Infrastructure: AnnotationUtils and Scanner/Mapping Updates

### AnnotationUtils -- recursive meta-annotation detection

This utility goes in `iris-framework` (package `com.iris.framework.core.annotation`) because it's a core framework utility used by multiple subsystems.

```java
package com.iris.framework.core.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.util.HashSet;
import java.util.Set;

public final class AnnotationUtils {

    private AnnotationUtils() {}

    public static boolean hasAnnotation(AnnotatedElement element,
                                         Class<? extends Annotation> targetAnnotation) {
        return hasAnnotation(element, targetAnnotation, new HashSet<>());
    }

    private static boolean hasAnnotation(AnnotatedElement element,
                                          Class<? extends Annotation> target,
                                          Set<Class<? extends Annotation>> visited) {
        if (element.isAnnotationPresent(target)) {
            return true;
        }
        for (Annotation ann : element.getDeclaredAnnotations()) {
            Class<? extends Annotation> annType = ann.annotationType();
            if (annType.getName().startsWith("java.lang.annotation")) {
                continue;
            }
            if (visited.add(annType)) {
                if (hasAnnotation(annType, target, visited)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

The algorithm is a depth-first traversal of the annotation hierarchy:

1. **Direct check** -- if the element has the target annotation directly, return `true`.
2. **Recursive check** -- for each annotation on the element, recursively check whether *that annotation type* has the target annotation.
3. **Skip JDK meta-annotations** -- `@Target`, `@Retention`, `@Documented` are in `java.lang.annotation` and are never what we're looking for.
4. **Cycle detection** -- the `visited` set prevents infinite loops from circular meta-annotation references.

This is the simplified equivalent of Spring's `AnnotatedElementUtils.hasAnnotation()`, which uses the full `MergedAnnotations` infrastructure with attribute aliasing (`@AliasFor`) and class hierarchy inheritance.

### ClassPathScanner -- updated to use AnnotationUtils

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ClassPathScanner.java`
**Change:** `hasComponentAnnotation()` and `resolveBeanName()` now use `AnnotationUtils.hasAnnotation()` for recursive meta-annotation detection at any depth.

```diff
+import com.iris.framework.core.annotation.AnnotationUtils;

 static boolean hasComponentAnnotation(Class<?> clazz) {
-    if (clazz.isAnnotationPresent(Component.class)) {
-        return true;
-    }
-    for (Annotation annotation : clazz.getAnnotations()) {
-        if (annotation.annotationType().isAnnotationPresent(Component.class)) {
-            return true;
-        }
-    }
-    return false;
+    return AnnotationUtils.hasAnnotation(clazz, Component.class);
 }
```

Before this change, `hasComponentAnnotation()` only looked one level deep: it checked direct `@Component` and then checked whether any direct annotation carried `@Component`. This worked for `@Controller`, `@Service`, `@Repository` (which are directly meta-annotated with `@Component`) but would fail for `@RestController` (which is `@RestController` -> `@Controller` -> `@Component`, two levels deep). With `AnnotationUtils`, the detection works at any depth.

Similarly, `resolveBeanName()` was updated to use `AnnotationUtils` for discovering which annotations in the stereotype hierarchy carry `@Component`:

```diff
 static String resolveBeanName(Class<?> clazz) {
     Component component = clazz.getAnnotation(Component.class);
     if (component != null && !component.value().isEmpty()) {
         return component.value();
     }
     for (Annotation annotation : clazz.getAnnotations()) {
-        if (annotation.annotationType().isAnnotationPresent(Component.class)) {
+        if (AnnotationUtils.hasAnnotation(annotation.annotationType(), Component.class)) {
             try {
                 Method valueMethod = annotation.annotationType().getMethod("value");
```

### RequestMappingHandlerMapping -- updated isController()

**Modifying:** `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingHandlerMapping.java`
**Change:** `isController()` now uses `AnnotationUtils.hasAnnotation()` instead of `isAnnotationPresent()`

```diff
+import com.iris.framework.core.annotation.AnnotationUtils;

 private boolean isController(Class<?> clazz) {
-    return clazz.isAnnotationPresent(Controller.class);
+    return AnnotationUtils.hasAnnotation(clazz, Controller.class);
 }
```

Without this change, `@RestController`-annotated classes would not be detected as controllers because `@RestController` is not `@Controller` -- it *carries* `@Controller` as a meta-annotation. The recursive detection makes `@RestController` work seamlessly.

---

## 11.5 Try It Yourself

<details><summary>Challenge 1: Implement hasResponseBody() for @ResponseBody detection</summary>

Given a `HandlerMethod`, determine whether the return value should be serialized via message converters. You need to check both the method level and the class level (for `@RestController` support):

```java
private boolean hasResponseBody(HandlerMethod handler) {
    // Check method-level @ResponseBody
    if (handler.getMethod().isAnnotationPresent(ResponseBody.class)) {
        return true;
    }
    // Check class-level @ResponseBody (covers @RestController)
    return AnnotationUtils.hasAnnotation(handler.getBean().getClass(), ResponseBody.class);
}
```

The method-level check uses `isAnnotationPresent()` because `@ResponseBody` is applied directly to methods -- no meta-annotation depth needed. The class-level check uses `AnnotationUtils.hasAnnotation()` because `@RestController` carries `@ResponseBody` as a meta-annotation (one level deep).

</details>

<details><summary>Challenge 2: Implement writeWithMessageConverters() for converter delegation</summary>

Given a return value and a list of registered converters, find the first converter that can serialize the value and delegate to it. Fall back to plain text if no converter matches:

```java
private void writeWithMessageConverters(Object result, HttpServletResponse response)
        throws IOException {
    for (HttpMessageConverter converter : messageConverters) {
        if (converter.canWrite(result.getClass())) {
            converter.write(result, response);
            return;
        }
    }
    // Fallback: no converter matched — write as plain text
    response.setContentType("text/plain;charset=UTF-8");
    response.getWriter().write(result.toString());
    response.getWriter().flush();
}
```

The iteration order matters -- the first compatible converter wins. In the real Spring Framework, this logic lives in `AbstractMessageConverterMethodProcessor.writeWithMessageConverters()` and includes content negotiation (checking the `Accept` header against each converter's supported media types), `ResponseBodyAdvice` callbacks, and media type selection. Our version simply asks "can you write this type?" and delegates.

</details>

---

## 11.6 Tests

### AnnotationUtilsTest -- recursive meta-annotation detection

```java
package com.iris.framework.core.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.junit.jupiter.api.Test;

import com.iris.framework.stereotype.Component;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.bind.annotation.RestController;

import static org.assertj.core.api.Assertions.assertThat;

class AnnotationUtilsTest {

    @Test
    void shouldDetectDirectAnnotation() {
        assertThat(AnnotationUtils.hasAnnotation(DirectComponent.class, Component.class))
                .isTrue();
    }

    @Test
    void shouldDetectOneLevel_WhenMetaAnnotatedWithComponent() {
        // @Controller carries @Component
        assertThat(AnnotationUtils.hasAnnotation(ControllerClass.class, Component.class))
                .isTrue();
    }

    @Test
    void shouldDetectTwoLevels_WhenRestControllerUsed() {
        // @RestController → @Controller → @Component
        assertThat(AnnotationUtils.hasAnnotation(RestControllerClass.class, Component.class))
                .isTrue();
    }

    @Test
    void shouldDetectControllerViaRestController() {
        // @RestController carries @Controller
        assertThat(AnnotationUtils.hasAnnotation(RestControllerClass.class, Controller.class))
                .isTrue();
    }

    @Test
    void shouldDetectResponseBodyViaRestController() {
        // @RestController carries @ResponseBody
        assertThat(AnnotationUtils.hasAnnotation(RestControllerClass.class, ResponseBody.class))
                .isTrue();
    }

    @Test
    void shouldReturnFalse_WhenAnnotationNotPresent() {
        assertThat(AnnotationUtils.hasAnnotation(PlainClass.class, Component.class))
                .isFalse();
    }

    @Test
    void shouldReturnFalse_WhenUnrelatedAnnotation() {
        assertThat(AnnotationUtils.hasAnnotation(ControllerClass.class, ResponseBody.class))
                .isFalse();
    }

    @Test
    void shouldDetectAnnotationOnMethod() throws NoSuchMethodException {
        assertThat(AnnotationUtils.hasAnnotation(
                MethodAnnotatedClass.class.getMethod("annotatedMethod"),
                ResponseBody.class))
                .isTrue();
    }

    @Test
    void shouldHandleCustomMetaAnnotationDepth() {
        // @Level3 → @Level2 → @Level1 → @Component (3 levels deep)
        assertThat(AnnotationUtils.hasAnnotation(DeeplyAnnotatedClass.class, Component.class))
                .isTrue();
    }

    // --- Test fixtures ---

    @Component
    static class DirectComponent {}

    @Controller
    static class ControllerClass {}

    @RestController
    static class RestControllerClass {}

    static class PlainClass {}

    static class MethodAnnotatedClass {
        @ResponseBody
        public String annotatedMethod() { return "test"; }
    }

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Component
    @interface Level1 {}

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Level1
    @interface Level2 {}

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Level2
    @interface Level3 {}

    @Level3
    static class DeeplyAnnotatedClass {}
}
```

The test covers the full spectrum: direct annotation, one-level meta-annotation (`@Controller` -> `@Component`), two-level (`@RestController` -> `@Controller` -> `@Component`), three-level (custom chain), method-level detection, and negative cases.

### HttpMessageConverterTest -- @ResponseBody/@RestController detection + converter selection

```java
package com.iris.framework.web.http.converter;

import java.io.IOException;
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import com.iris.framework.web.bind.annotation.GetMapping;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.bind.annotation.RestController;
import com.iris.framework.web.servlet.HandlerMethod;
import com.iris.framework.web.servlet.RequestMappingHandlerAdapter;
import com.iris.framework.web.servlet.mock.MockHttpServletRequest;
import com.iris.framework.web.servlet.mock.MockHttpServletResponse;
import com.iris.framework.stereotype.Controller;

import jakarta.servlet.http.HttpServletResponse;

import static org.assertj.core.api.Assertions.assertThat;

class HttpMessageConverterTest {

    @Test
    void shouldUseConverter_WhenMethodHasResponseBody() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(ResponseBodyController.class, "getMessage");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/message");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[Hello]");
        assertThat(response.getContentType()).isEqualTo("text/brackets");
    }

    @Test
    void shouldUseConverter_WhenClassHasRestController() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(RestApiController.class, "getData");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/data");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[some data]");
        assertThat(response.getContentType()).isEqualTo("text/brackets");
    }

    @Test
    void shouldUsePlainText_WhenNoResponseBody() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(PlainController.class, "getText");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/text");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("plain text");
        assertThat(response.getContentType()).isEqualTo("text/plain;charset=UTF-8");
    }

    @Test
    void shouldFallbackToPlainText_WhenNoConverterAvailable() throws Exception {
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of());

        HandlerMethod handler = createHandler(ResponseBodyController.class, "getMessage");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/message");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("Hello");
        assertThat(response.getContentType()).isEqualTo("text/plain;charset=UTF-8");
    }

    @Test
    void shouldSkipIncompatibleConverter_WhenCanWriteReturnsFalse() throws Exception {
        HttpMessageConverter mapOnlyConverter = new MapOnlyConverter();
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(mapOnlyConverter, bracketsConverter));

        HandlerMethod handler = createHandler(ResponseBodyController.class, "getMessage");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/message");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        // MapOnlyConverter.canWrite(String.class) returns false, so BracketsConverter is used
        assertThat(response.getContentAsString()).isEqualTo("[Hello]");
    }

    @Test
    void shouldUseClassLevelResponseBody_WhenMethodLacksAnnotation() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(ClassLevelResponseBodyController.class, "getValue");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/value");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[test value]");
    }

    // --- Test converters ---

    static class BracketsConverter implements HttpMessageConverter {
        @Override
        public boolean canWrite(Class<?> clazz) { return true; }

        @Override
        public void write(Object value, HttpServletResponse response) throws IOException {
            response.setContentType("text/brackets");
            response.getWriter().write("[" + value + "]");
            response.getWriter().flush();
        }
    }

    static class MapOnlyConverter implements HttpMessageConverter {
        @Override
        public boolean canWrite(Class<?> clazz) {
            return Map.class.isAssignableFrom(clazz);
        }

        @Override
        public void write(Object value, HttpServletResponse response) throws IOException {
            response.setContentType("text/map");
            response.getWriter().write("MAP:" + value);
            response.getWriter().flush();
        }
    }

    // --- Test controllers ---

    @Controller
    static class ResponseBodyController {
        @ResponseBody
        public String getMessage() { return "Hello"; }
    }

    @RestController
    static class RestApiController {
        public String getData() { return "some data"; }
    }

    @Controller
    static class PlainController {
        public String getText() { return "plain text"; }
    }

    @Controller
    @ResponseBody
    static class ClassLevelResponseBodyController {
        public String getValue() { return "test value"; }
    }

    // --- Helpers ---

    private HandlerMethod createHandler(Class<?> controllerClass, String methodName) {
        try {
            Object controller = controllerClass.getDeclaredConstructor().newInstance();
            for (Method m : controllerClass.getDeclaredMethods()) {
                if (m.getName().equals(methodName)) {
                    return new HandlerMethod(controller, m);
                }
            }
            throw new IllegalArgumentException("No method named: " + methodName);
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException(e);
        }
    }
}
```

This test uses a custom `BracketsConverter` (wraps values in `[]`) to verify that the adapter delegates to converters without depending on Jackson. It tests six scenarios: method-level `@ResponseBody`, class-level `@RestController`, plain controller (no converter), empty converter list fallback, `canWrite()` filtering, and class-level `@ResponseBody` without `@RestController`.

### JacksonHttpMessageConverterTest -- JSON serialization

```java
package com.iris.boot.web.http.converter;

import java.io.IOException;
import java.io.PrintWriter;
import java.io.StringWriter;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import static org.assertj.core.api.Assertions.assertThat;

class JacksonHttpMessageConverterTest {

    private final JacksonHttpMessageConverter converter = new JacksonHttpMessageConverter();

    @Test
    void shouldCanWriteAnyClass() {
        assertThat(converter.canWrite(String.class)).isTrue();
        assertThat(converter.canWrite(UserDto.class)).isTrue();
        assertThat(converter.canWrite(Object.class)).isTrue();
    }

    @Test
    void shouldSerializePojo_WhenWriteCalled() throws IOException {
        UserDto user = new UserDto(1, "Alice", "alice@example.com");
        SimpleResponse response = new SimpleResponse();

        converter.write(user, response);

        String json = response.getBody();
        assertThat(json).contains("\"id\":1");
        assertThat(json).contains("\"name\":\"Alice\"");
        assertThat(json).contains("\"email\":\"alice@example.com\"");
        assertThat(response.contentType).isEqualTo("application/json;charset=UTF-8");
    }

    @Test
    void shouldSerializeRecord_WhenWriteCalled() throws IOException {
        ProductRecord product = new ProductRecord("Widget", 9.99);
        SimpleResponse response = new SimpleResponse();

        converter.write(product, response);

        String json = response.getBody();
        assertThat(json).contains("\"name\":\"Widget\"");
        assertThat(json).contains("\"price\":9.99");
    }

    @Test
    void shouldSerializeString_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write("hello", response);

        assertThat(response.getBody()).isEqualTo("\"hello\"");
        assertThat(response.contentType).isEqualTo("application/json;charset=UTF-8");
    }

    @Test
    void shouldSerializeMap_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write(java.util.Map.of("key", "value"), response);

        assertThat(response.getBody()).contains("\"key\":\"value\"");
    }

    @Test
    void shouldSerializeList_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write(java.util.List.of(1, 2, 3), response);

        assertThat(response.getBody()).isEqualTo("[1,2,3]");
    }

    @Test
    void shouldUseCustomObjectMapper_WhenProvided() throws IOException {
        ObjectMapper customMapper = new ObjectMapper();
        JacksonHttpMessageConverter customConverter = new JacksonHttpMessageConverter(customMapper);
        SimpleResponse response = new SimpleResponse();

        customConverter.write(new UserDto(1, "Bob", null), response);

        String json = response.getBody();
        assertThat(json).contains("\"name\":\"Bob\"");
        assertThat(customConverter.getObjectMapper()).isSameAs(customMapper);
    }

    @Test
    void shouldHandleNull_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write(null, response);

        assertThat(response.getBody()).isEqualTo("null");
    }

    // --- Minimal response mock ---

    static class SimpleResponse extends NoOpHttpServletResponse {
        private final StringWriter writer = new StringWriter();
        private final PrintWriter printWriter = new PrintWriter(writer);
        String contentType;

        @Override
        public void setContentType(String type) { this.contentType = type; }

        @Override
        public String getContentType() { return contentType; }

        @Override
        public PrintWriter getWriter() { return printWriter; }

        String getBody() {
            printWriter.flush();
            return writer.toString();
        }
    }

    // --- Test fixtures ---

    static class UserDto {
        private int id;
        private String name;
        private String email;

        public UserDto() {}
        public UserDto(int id, String name, String email) {
            this.id = id; this.name = name; this.email = email;
        }

        public int getId() { return id; }
        public void setId(int id) { this.id = id; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    record ProductRecord(String name, double price) {}
}
```

This test covers the full range of types Jackson can serialize: POJOs, records, strings, maps, lists, and null. It uses a minimal `SimpleResponse` mock that captures content type and body without depending on a full servlet container.

### JsonResponseBodyIntegrationTest -- full end-to-end

```java
package com.iris.boot.integration;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.http.converter.JacksonHttpMessageConverter;
import com.iris.boot.web.context.ServletWebServerApplicationContext;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.web.bind.annotation.GetMapping;
import com.iris.framework.web.bind.annotation.PathVariable;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.bind.annotation.RestController;
import com.iris.framework.web.http.converter.HttpMessageConverter;

import static org.assertj.core.api.Assertions.assertThat;

class JsonResponseBodyIntegrationTest {

    private ServletWebServerApplicationContext context;
    private HttpClient httpClient;
    private ObjectMapper objectMapper;
    private int port;

    @BeforeEach
    void setUp() {
        context = new ServletWebServerApplicationContext();
        context.register(JsonTestConfig.class);
        context.refresh();

        port = context.getWebServer().getPort();
        httpClient = HttpClient.newHttpClient();
        objectMapper = new ObjectMapper();
    }

    @AfterEach
    void cleanup() {
        if (context != null) {
            context.close();
        }
    }

    @Test
    void shouldReturnJson_WhenRestControllerReturnsObject() throws Exception {
        HttpResponse<String> response = get("/api/users/1");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("application/json"));

        Map<?, ?> user = objectMapper.readValue(response.body(), Map.class);
        assertThat(user.get("id")).isEqualTo(1);
        assertThat(user.get("name")).isEqualTo("Alice");
    }

    @Test
    void shouldReturnJsonArray_WhenRestControllerReturnsList() throws Exception {
        HttpResponse<String> response = get("/api/users");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("application/json"));

        List<?> users = objectMapper.readValue(response.body(), List.class);
        assertThat(users).hasSize(2);
    }

    @Test
    void shouldReturnJsonRecord_WhenRestControllerReturnsRecord() throws Exception {
        HttpResponse<String> response = get("/api/products/42");

        assertThat(response.statusCode()).isEqualTo(200);

        Map<?, ?> product = objectMapper.readValue(response.body(), Map.class);
        assertThat(product.get("name")).isEqualTo("Widget");
        assertThat(product.get("price")).isEqualTo(19.99);
    }

    @Test
    void shouldReturnJson_WhenMethodHasResponseBody() throws Exception {
        HttpResponse<String> response = get("/mixed/json");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("application/json"));

        Map<?, ?> result = objectMapper.readValue(response.body(), Map.class);
        assertThat(result.get("status")).isEqualTo("ok");
    }

    @Test
    void shouldReturnPlainText_WhenMethodLacksResponseBody() throws Exception {
        HttpResponse<String> response = get("/mixed/text");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("text/plain"));
        assertThat(response.body()).isEqualTo("just text");
    }

    @Test
    void shouldReturnJsonMap_WhenRestControllerReturnsMap() throws Exception {
        HttpResponse<String> response = get("/api/config");

        assertThat(response.statusCode()).isEqualTo(200);

        Map<?, ?> config = objectMapper.readValue(response.body(), Map.class);
        assertThat(config.get("version")).isEqualTo("1.0");
        assertThat(config.get("debug")).isEqualTo(false);
    }

    // --- Helper ---

    private HttpResponse<String> get(String path) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("http://localhost:" + port + path))
                .GET()
                .build();
        return httpClient.send(request, HttpResponse.BodyHandlers.ofString());
    }

    // --- Test configuration ---

    @Configuration
    static class JsonTestConfig {
        @Bean
        public ServletWebServerFactory webServerFactory() {
            return new TomcatServletWebServerFactory(0);
        }

        @Bean
        public HttpMessageConverter jacksonConverter() {
            return new JacksonHttpMessageConverter();
        }

        @Bean
        public UserApiController userApiController() {
            return new UserApiController();
        }

        @Bean
        public MixedController mixedController() {
            return new MixedController();
        }
    }

    // --- Test controllers ---

    @RestController
    static class UserApiController {
        @GetMapping("/api/users/{id}")
        public UserDto getUser(@PathVariable("id") int id) {
            return new UserDto(id, "Alice", "alice@example.com");
        }

        @GetMapping("/api/users")
        public List<UserDto> listUsers() {
            return List.of(
                    new UserDto(1, "Alice", "alice@example.com"),
                    new UserDto(2, "Bob", "bob@example.com"));
        }

        @GetMapping("/api/products/{id}")
        public ProductRecord getProduct(@PathVariable("id") int id) {
            return new ProductRecord("Widget", 19.99);
        }

        @GetMapping("/api/config")
        public Map<String, Object> getConfig() {
            return Map.of("version", "1.0", "debug", false);
        }
    }

    @Controller
    static class MixedController {
        @GetMapping("/mixed/json")
        @ResponseBody
        public Map<String, String> jsonEndpoint() {
            return Map.of("status", "ok");
        }

        @GetMapping("/mixed/text")
        public String textEndpoint() {
            return "just text";
        }
    }

    // --- DTOs ---

    static class UserDto {
        private int id;
        private String name;
        private String email;

        public UserDto() {}
        public UserDto(int id, String name, String email) {
            this.id = id; this.name = name; this.email = email;
        }

        public int getId() { return id; }
        public void setId(int id) { this.id = id; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    record ProductRecord(String name, double price) {}
}
```

The integration test boots a real `ServletWebServerApplicationContext` with embedded Tomcat on a random port, registers a `JacksonHttpMessageConverter` as a bean, and sends real HTTP requests using Java's `HttpClient`. It verifies JSON content type, JSON structure, and the mixed controller behavior (JSON for `@ResponseBody` methods, plain text for others).

The key detail in the configuration: `JacksonHttpMessageConverter` is registered as a bean of type `HttpMessageConverter`. The `DispatcherServlet.findMessageConverters()` discovers it via `factory.getBeansOfType(HttpMessageConverter.class)` and passes it to the adapter.

Run: `./gradlew test`

---

## 11.7 Why This Works

`★ Insight ─────────────────────────────────────`
**Composed Annotations + Recursive Detection = Zero-Effort Convention.**
`@RestController` is not a magical keyword -- it is a plain annotation that happens to carry `@Controller` and `@ResponseBody` as meta-annotations. The entire "make this a REST controller" behavior emerges from three independently developed pieces: component scanning finds it (because `@Controller` -> `@Component`), handler mapping registers it (because `@Controller`), and the adapter serializes its responses (because `@ResponseBody`). No single piece of code has a `case RESTCONTROLLER:` branch. `AnnotationUtils.hasAnnotation()` is the glue that makes composed annotations work transparently at any depth.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Strategy Interface + Bean Discovery = Open/Closed Serialization.**
The adapter does not know about JSON, XML, or any specific format. It knows about `HttpMessageConverter` -- a strategy interface. The `DispatcherServlet` discovers converter beans from the `ApplicationContext` and hands them to the adapter. To add XML support, you register a new `HttpMessageConverter` bean; to remove JSON support, you remove the Jackson converter bean. No existing code changes. This is the Open/Closed Principle in action. In the real Spring Framework, `WebMvcAutoConfiguration` auto-registers converters based on classpath detection (Jackson on classpath -> JSON converter registered). Our version requires explicit bean registration, but the extension mechanism is identical.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Two Levels of Fallback = Graceful Degradation.**
The system has two fallback layers. First, `hasResponseBody()` checks: if the handler method does not have `@ResponseBody` (directly or via `@RestController`), the adapter bypasses the converter pipeline entirely and writes plain text. Second, `writeWithMessageConverters()` iterates converters: if no converter's `canWrite()` returns `true`, the method falls back to `toString()`. This means: (1) non-`@ResponseBody` methods always get plain text (backward compatible with chapter 10), (2) `@ResponseBody` methods with a compatible converter get JSON, and (3) `@ResponseBody` methods with no compatible converter still work -- they just get plain text. Nothing throws an exception. This graceful degradation matches real Spring's behavior.
`─────────────────────────────────────────────────`

---

## 11.8 What We Enhanced

| Aspect | Before (ch10) | Current (ch11) | Real Framework |
|--------|---------------|----------------|----------------|
| Return value handling | Always `toString()` with `text/plain` | `@ResponseBody` -> JSON via converters; plain text otherwise | `HandlerMethodReturnValueHandler` SPI with 15+ handlers (`RequestResponseBodyMethodProcessor.java:195`) |
| Controller detection | `clazz.isAnnotationPresent(Controller.class)` -- direct only | Recursive meta-annotation via `AnnotationUtils.hasAnnotation()` | `AnnotatedElementUtils.hasAnnotation()` with `MergedAnnotations` (`RequestMappingHandlerMapping.java:294`) |
| Component scanning | One-level meta-annotation check | Recursive meta-annotation detection at any depth | ASM-based `MetadataReader` with `MergedAnnotations` (`ClassPathScanningCandidateComponentProvider.java:372`) |
| Adapter initialization | `new RequestMappingHandlerAdapter()` -- no converters | Discovers `HttpMessageConverter` beans from context | `WebMvcConfigurationSupport.getMessageConverters()` with customizer chain (`WebMvcConfigurationSupport.java:704`) |

---

## 11.9 Connection to Real Spring Framework

All file references are from Spring Framework commit `11ab0b4351` and Spring Boot commit `5922311a95a`.

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@ResponseBody` | `@ResponseBody` | `ResponseBody.java:41` (spring-web) | Identical -- simple marker annotation |
| `@RestController` | `@RestController` | `RestController.java:43` (spring-web) | Real uses `@AliasFor` for value aliasing |
| `HttpMessageConverter` | `HttpMessageConverter<T>` | `HttpMessageConverter.java:50` (spring-web) | Real is generic, has `canRead()`/`read()`, uses `MediaType`, `HttpOutputMessage` |
| `JacksonHttpMessageConverter` | `MappingJackson2HttpMessageConverter` | `MappingJackson2HttpMessageConverter.java:58` (spring-web) | Real handles content negotiation, JSON views, anti-hijacking prefix |
| `hasResponseBody()` | `RequestResponseBodyMethodProcessor.supportsReturnType()` | `RequestResponseBodyMethodProcessor.java:139` (spring-webmvc) | Real uses `HandlerMethodReturnValueHandler` SPI with cached lookups |
| `writeWithMessageConverters()` | `AbstractMessageConverterMethodProcessor.writeWithMessageConverters()` | `AbstractMessageConverterMethodProcessor.java:205` (spring-webmvc) | Real does content negotiation, ResponseBodyAdvice, media type selection |
| `AnnotationUtils.hasAnnotation()` | `AnnotatedElementUtils.hasAnnotation()` | `AnnotatedElementUtils.java:153` (spring-core) | Real uses `MergedAnnotations` with attribute aliasing and inheritance |
| `findMessageConverters()` | `WebMvcAutoConfiguration` | `WebMvcAutoConfiguration.java` (spring-boot-autoconfigure) | Real uses `WebMvcConfigurer.configureMessageConverters()` customization pipeline |

---

## 11.10 Complete Code

### Production Code

#### [NEW] `iris-framework/src/main/java/com/iris/framework/core/annotation/AnnotationUtils.java`

```java
package com.iris.framework.core.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.util.HashSet;
import java.util.Set;

/**
 * Utility for recursive meta-annotation detection.
 *
 * <p>Java's built-in {@code AnnotatedElement.isAnnotationPresent()} only
 * checks <em>direct</em> annotations. But Spring's stereotype model is
 * built on meta-annotations: {@code @RestController} carries
 * {@code @Controller} which carries {@code @Component}. To detect
 * {@code @Component} on a {@code @RestController}-annotated class, we
 * need to walk the annotation hierarchy.
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code AnnotatedElementUtils.hasAnnotation()} which uses Spring's
 * full merged annotation infrastructure ({@code MergedAnnotations}).
 * We use a simple recursive traversal with cycle detection.
 *
 * @see org.springframework.core.annotation.AnnotatedElementUtils#hasAnnotation
 */
public final class AnnotationUtils {

    private AnnotationUtils() {
        // utility class
    }

    /**
     * Check whether the given element has the target annotation, either
     * directly or as a meta-annotation at any depth.
     *
     * <p>For example, given:
     * <pre>
     * {@code @Controller}  // meta-annotated with @Component
     * {@code @ResponseBody}
     * public @interface RestController {}
     * </pre>
     *
     * Calling {@code hasAnnotation(RestController.class, Component.class)}
     * returns {@code true} because {@code @RestController} → {@code @Controller}
     * → {@code @Component}.
     *
     * @param element          the element to inspect (class, method, or annotation type)
     * @param targetAnnotation the annotation to look for
     * @return true if the target annotation is present (directly or as meta-annotation)
     */
    public static boolean hasAnnotation(AnnotatedElement element,
                                         Class<? extends Annotation> targetAnnotation) {
        return hasAnnotation(element, targetAnnotation, new HashSet<>());
    }

    /**
     * Recursive implementation with cycle detection.
     *
     * <p>The {@code visited} set prevents infinite loops from circular
     * meta-annotation references (which shouldn't happen in practice,
     * but defensive programming is cheap here).
     */
    private static boolean hasAnnotation(AnnotatedElement element,
                                          Class<? extends Annotation> target,
                                          Set<Class<? extends Annotation>> visited) {
        // Direct check
        if (element.isAnnotationPresent(target)) {
            return true;
        }

        // Recursive: check meta-annotations
        for (Annotation ann : element.getDeclaredAnnotations()) {
            Class<? extends Annotation> annType = ann.annotationType();

            // Skip JDK meta-annotations (@Target, @Retention, etc.)
            if (annType.getName().startsWith("java.lang.annotation")) {
                continue;
            }

            // Visit each annotation type at most once (cycle prevention)
            if (visited.add(annType)) {
                if (hasAnnotation(annType, target, visited)) {
                    return true;
                }
            }
        }

        return false;
    }
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/ResponseBody.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that a method's return value should be serialized directly
 * to the HTTP response body (e.g., as JSON), rather than interpreted
 * as a view name.
 *
 * <p>When placed on a method, only that method's return value is serialized.
 * When placed on a class (or via {@link RestController @RestController}),
 * all handler methods in the class get response body semantics.
 *
 * <p>In the real Spring Framework, {@code @ResponseBody} triggers the
 * {@code RequestResponseBodyMethodProcessor} which iterates registered
 * {@code HttpMessageConverter}s to find one that can serialize the
 * return value for the negotiated media type. Our simplified version
 * checks for the annotation in the {@code RequestMappingHandlerAdapter}
 * and delegates to the first compatible converter.
 *
 * <h3>Simplifications from Real Spring</h3>
 * <ul>
 *   <li>No content negotiation (always writes JSON if a converter is available)</li>
 *   <li>No {@code ResponseBodyAdvice} / {@code @ControllerAdvice}</li>
 *   <li>No status code customization via {@code @ResponseStatus}</li>
 * </ul>
 *
 * @see RestController
 * @see org.springframework.web.bind.annotation.ResponseBody
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ResponseBody {
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/bind/annotation/RestController.java`

```java
package com.iris.framework.web.bind.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.stereotype.Controller;

/**
 * A convenience annotation that is itself annotated with
 * {@link Controller @Controller} and {@link ResponseBody @ResponseBody}.
 *
 * <p>Classes annotated with {@code @RestController} are treated as
 * controllers (detected by {@code RequestMappingHandlerMapping}) and
 * every handler method automatically gets {@code @ResponseBody}
 * semantics — its return value is serialized to the response body
 * (typically as JSON) instead of being interpreted as a view name.
 *
 * <p>This is the most common annotation for building REST APIs:
 * <pre>{@code
 * @RestController
 * public class UserController {
 *     @GetMapping("/users/{id}")
 *     public User getUser(@PathVariable int id) {
 *         return new User(id, "Alice");  // serialized as JSON
 *     }
 * }
 * }</pre>
 *
 * <h3>Meta-Annotation Design</h3>
 * <p>{@code @RestController} is a <em>composed annotation</em>. Because it
 * carries {@code @Controller} (which carries {@code @Component}), classes
 * annotated with {@code @RestController} are automatically discovered by
 * both component scanning and the handler mapping. The
 * {@link com.iris.framework.core.annotation.AnnotationUtils AnnotationUtils}
 * provides recursive meta-annotation detection to support this.
 *
 * @see Controller
 * @see ResponseBody
 * @see org.springframework.web.bind.annotation.RestController
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {

    /**
     * The logical bean name for this controller component.
     * If empty, the scanner derives a name from the class name.
     */
    String value() default "";
}
```

#### [NEW] `iris-framework/src/main/java/com/iris/framework/web/http/converter/HttpMessageConverter.java`

```java
package com.iris.framework.web.http.converter;

import java.io.IOException;

import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for converting an object to an HTTP response body.
 *
 * <p>Implementations handle the serialization of Java objects into a
 * specific content type. For example, a Jackson-based converter serializes
 * objects to JSON ({@code application/json}).
 *
 * <p>In the real Spring Framework, {@code HttpMessageConverter} is a
 * generic interface that handles both reading (deserialization) and
 * writing (serialization), using {@code HttpInputMessage} and
 * {@code HttpOutputMessage} abstractions instead of raw servlet
 * request/response objects.
 *
 * <h3>Simplifications from Real Spring</h3>
 * <ul>
 *   <li>Write-only — no {@code canRead()} / {@code read()} methods
 *       (that would come with {@code @RequestBody} support)</li>
 *   <li>Uses {@link HttpServletResponse} directly instead of
 *       {@code HttpOutputMessage}</li>
 *   <li>No media type negotiation — the converter decides its
 *       own content type</li>
 * </ul>
 *
 * @see org.springframework.http.converter.HttpMessageConverter
 */
public interface HttpMessageConverter {

    /**
     * Check if this converter can serialize the given class.
     *
     * @param clazz the class to check
     * @return true if this converter can write it
     */
    boolean canWrite(Class<?> clazz);

    /**
     * Serialize the given object and write it to the HTTP response.
     *
     * <p>The converter is responsible for setting the {@code Content-Type}
     * header on the response.
     *
     * @param value    the object to serialize
     * @param response the HTTP response to write to
     * @throws IOException if writing fails
     */
    void write(Object value, HttpServletResponse response) throws IOException;
}
```

#### [NEW] `iris-boot-core/src/main/java/com/iris/boot/web/http/converter/JacksonHttpMessageConverter.java`

```java
package com.iris.boot.web.http.converter;

import java.io.IOException;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;

import com.iris.framework.web.http.converter.HttpMessageConverter;

import jakarta.servlet.http.HttpServletResponse;

/**
 * {@link HttpMessageConverter} implementation that uses Jackson's
 * {@link ObjectMapper} to serialize Java objects as JSON.
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code MappingJackson2HttpMessageConverter} (for Jackson 2.x) or
 * {@code JacksonJsonHttpMessageConverter} (for Jackson 3.x in Spring
 * Framework 7.0). The real implementation:
 * <ul>
 *   <li>Extends {@code AbstractGenericHttpMessageConverter} for generic type support</li>
 *   <li>Supports content negotiation ({@code application/json}, {@code application/*+json})</li>
 *   <li>Handles JSON views ({@code @JsonView})</li>
 *   <li>Supports JSON prefix for anti-hijacking</li>
 *   <li>Uses {@code HttpOutputMessage} instead of raw {@code HttpServletResponse}</li>
 * </ul>
 *
 * <p>Our version simply serializes any object to JSON and sets the
 * {@code Content-Type} to {@code application/json}.
 *
 * @see com.fasterxml.jackson.databind.ObjectMapper
 * @see org.springframework.http.converter.json.MappingJackson2HttpMessageConverter
 */
public class JacksonHttpMessageConverter implements HttpMessageConverter {

    private final ObjectMapper objectMapper;

    /**
     * Create a converter with a default {@link ObjectMapper}.
     */
    public JacksonHttpMessageConverter() {
        this(createDefaultObjectMapper());
    }

    /**
     * Create a converter with the given {@link ObjectMapper}.
     *
     * @param objectMapper the ObjectMapper to use for serialization
     */
    public JacksonHttpMessageConverter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    /**
     * Returns {@code true} for any class — Jackson can serialize most
     * Java objects, including POJOs, records, collections, and maps.
     *
     * <p>In the real Spring Framework, the check is more nuanced:
     * {@code MappingJackson2HttpMessageConverter.canWrite()} verifies
     * that the {@code ObjectMapper} can actually serialize the type
     * by calling {@code objectMapper.canSerialize(clazz)}.
     */
    @Override
    public boolean canWrite(Class<?> clazz) {
        return true;
    }

    /**
     * Serialize the given value as JSON and write it to the response.
     *
     * <p>Sets the {@code Content-Type} header to
     * {@code application/json;charset=UTF-8} before writing.
     */
    @Override
    public void write(Object value, HttpServletResponse response) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        String json = objectMapper.writeValueAsString(value);
        response.getWriter().write(json);
        response.getWriter().flush();
    }

    /**
     * Return the {@link ObjectMapper} used by this converter.
     */
    public ObjectMapper getObjectMapper() {
        return objectMapper;
    }

    private static ObjectMapper createDefaultObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        return mapper;
    }
}
```

#### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingHandlerAdapter.java`

```java
package com.iris.framework.web.servlet;

import java.io.IOException;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.Collections;
import java.util.List;
import java.util.Map;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import com.iris.framework.core.annotation.AnnotationUtils;
import com.iris.framework.web.bind.annotation.PathVariable;
import com.iris.framework.web.bind.annotation.RequestParam;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.http.converter.HttpMessageConverter;

/**
 * Implementation of {@link HandlerAdapter} that invokes handler methods
 * annotated with {@code @RequestMapping} (and its variants), resolving
 * method parameters from the HTTP request.
 *
 * <p>This adapter is the bridge between the HTTP request and the Java
 * method call. It handles:
 * <ol>
 *   <li><strong>Parameter resolution</strong> — match method parameters to
 *       their sources ({@code @PathVariable}, {@code @RequestParam},
 *       {@code HttpServletRequest}, {@code HttpServletResponse})</li>
 *   <li><strong>Type conversion</strong> — convert string values to the
 *       parameter's declared type</li>
 *   <li><strong>Method invocation</strong> — reflectively call the handler</li>
 *   <li><strong>Return value handling</strong> — write the result to the
 *       response (plain text for now; Feature 11 adds JSON)</li>
 * </ol>
 *
 * <h3>Parameter Resolution Order</h3>
 * <p>For each method parameter, the adapter checks (in order):
 * <ol>
 *   <li>{@code HttpServletRequest} type → inject the request</li>
 *   <li>{@code HttpServletResponse} type → inject the response</li>
 *   <li>{@code @PathVariable} annotation → extract from URI template variables</li>
 *   <li>{@code @RequestParam} annotation → extract from query parameters</li>
 * </ol>
 *
 * <h3>Simplifications from Real Spring</h3>
 * <ul>
 *   <li>No {@code HandlerMethodArgumentResolver} SPI — resolution is inline</li>
 *   <li>No {@code HandlerMethodReturnValueHandler} SPI — returns are written directly</li>
 *   <li>No {@code ModelAndView} — responses are written immediately</li>
 *   <li>No {@code @RequestBody} deserialization — only {@code @ResponseBody} serialization</li>
 *   <li>No content negotiation — the first compatible converter wins</li>
 *   <li>Type conversion limited to: String, int/Integer, long/Long, boolean/Boolean, double/Double</li>
 * </ul>
 *
 * @see org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
 */
public class RequestMappingHandlerAdapter implements HandlerAdapter {

    /** Sentinel value used by {@link RequestParam#defaultValue()} to indicate "no default". */
    private static final String NO_DEFAULT = "\n\t\t\n\t\t\n\uE000\uE001\uE002\n\t\t\t\t\n";

    /** Registered message converters for @ResponseBody serialization. */
    private final List<HttpMessageConverter> messageConverters;

    /**
     * Create an adapter with the given message converters.
     *
     * <p>When a handler method has {@code @ResponseBody} (or belongs to a
     * {@code @RestController}), the adapter iterates these converters to
     * find one that can serialize the return value. If no converter matches,
     * it falls back to plain-text {@code toString()} behavior.
     *
     * @param messageConverters the converters to use for @ResponseBody methods
     */
    public RequestMappingHandlerAdapter(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters != null ? messageConverters : List.of();
    }

    /**
     * Create an adapter with no message converters.
     * All return values will be written as plain text via {@code toString()}.
     */
    public RequestMappingHandlerAdapter() {
        this(List.of());
    }

    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       HandlerMethod handler) throws Exception {
        Method method = handler.getMethod();
        method.setAccessible(true);

        // Resolve all method parameters
        Object[] args = resolveArguments(method, request, response);

        // Invoke the handler method
        Object result = method.invoke(handler.getBean(), args);

        // Handle return value — write to response if not void and response not committed
        if (result != null && !response.isCommitted()) {
            if (hasResponseBody(handler)) {
                writeWithMessageConverters(result, response);
            } else {
                response.setContentType("text/plain;charset=UTF-8");
                response.getWriter().write(result.toString());
                response.getWriter().flush();
            }
        }
    }

    // -----------------------------------------------------------------------
    // @ResponseBody detection and converter delegation
    // -----------------------------------------------------------------------

    /**
     * Check if the handler method should serialize its return value via
     * message converters.
     *
     * <p>Returns {@code true} if:
     * <ul>
     *   <li>The method itself is annotated with {@code @ResponseBody}</li>
     *   <li>The controller class has {@code @ResponseBody} (directly or
     *       via {@code @RestController})</li>
     * </ul>
     *
     * <p>Uses recursive meta-annotation detection via
     * {@link AnnotationUtils#hasAnnotation} so that {@code @RestController}
     * (which carries {@code @ResponseBody} as a meta-annotation) is
     * detected correctly.
     */
    private boolean hasResponseBody(HandlerMethod handler) {
        // Check method-level @ResponseBody
        if (handler.getMethod().isAnnotationPresent(ResponseBody.class)) {
            return true;
        }
        // Check class-level @ResponseBody (covers @RestController)
        return AnnotationUtils.hasAnnotation(handler.getBean().getClass(), ResponseBody.class);
    }

    /**
     * Serialize the return value using the first compatible message converter.
     *
     * <p>Iterates registered converters and delegates to the first one
     * whose {@code canWrite()} returns {@code true}. If no converter
     * matches, falls back to plain-text output.
     *
     * <p>In the real Spring Framework, this logic lives in
     * {@code AbstractMessageConverterMethodProcessor.writeWithMessageConverters()}
     * and includes content negotiation, {@code ResponseBodyAdvice}, and
     * media type selection.
     */
    private void writeWithMessageConverters(Object result, HttpServletResponse response)
            throws IOException {
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(result.getClass())) {
                converter.write(result, response);
                return;
            }
        }
        // Fallback: no converter matched — write as plain text
        response.setContentType("text/plain;charset=UTF-8");
        response.getWriter().write(result.toString());
        response.getWriter().flush();
    }

    // -----------------------------------------------------------------------
    // Parameter resolution
    // -----------------------------------------------------------------------

    /**
     * Resolve all parameters for the given handler method.
     */
    private Object[] resolveArguments(Method method, HttpServletRequest request,
                                       HttpServletResponse response) {
        Parameter[] params = method.getParameters();
        Object[] args = new Object[params.length];

        @SuppressWarnings("unchecked")
        Map<String, String> pathVariables = (Map<String, String>) request.getAttribute(
                HandlerMapping.PATH_VARIABLES_ATTRIBUTE);
        if (pathVariables == null) {
            pathVariables = Collections.emptyMap();
        }

        for (int i = 0; i < params.length; i++) {
            args[i] = resolveArgument(params[i], request, response, pathVariables);
        }
        return args;
    }

    /**
     * Resolve a single method parameter.
     */
    private Object resolveArgument(Parameter param, HttpServletRequest request,
                                    HttpServletResponse response,
                                    Map<String, String> pathVariables) {
        Class<?> type = param.getType();

        // 1. HttpServletRequest injection
        if (HttpServletRequest.class.isAssignableFrom(type)) {
            return request;
        }

        // 2. HttpServletResponse injection
        if (HttpServletResponse.class.isAssignableFrom(type)) {
            return response;
        }

        // 3. @PathVariable
        PathVariable pathVar = param.getAnnotation(PathVariable.class);
        if (pathVar != null) {
            String name = pathVar.value().isEmpty() ? param.getName() : pathVar.value();
            String value = pathVariables.get(name);
            return convertValue(value, type);
        }

        // 4. @RequestParam
        RequestParam reqParam = param.getAnnotation(RequestParam.class);
        if (reqParam != null) {
            String name = reqParam.value().isEmpty() ? param.getName() : reqParam.value();
            String value = request.getParameter(name);
            if (value == null) {
                String defaultValue = reqParam.defaultValue();
                if (!NO_DEFAULT.equals(defaultValue)) {
                    value = defaultValue;
                } else if (reqParam.required()) {
                    throw new IllegalArgumentException(
                            "Required request parameter '" + name + "' is not present");
                }
            }
            return convertValue(value, type);
        }

        // 5. No annotation — return null for unrecognized parameters
        return null;
    }

    // -----------------------------------------------------------------------
    // Type conversion
    // -----------------------------------------------------------------------

    /**
     * Convert a string value to the target type.
     *
     * <p>Supports the most common types. In the real Spring Framework, type
     * conversion is handled by the full {@code ConversionService} and
     * {@code TypeConverter} infrastructure.
     */
    static Object convertValue(String value, Class<?> targetType) {
        if (value == null) {
            if (targetType.isPrimitive()) {
                return getDefaultPrimitiveValue(targetType);
            }
            return null;
        }

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

        return value;
    }

    private static Object getDefaultPrimitiveValue(Class<?> type) {
        if (type == int.class) return 0;
        if (type == long.class) return 0L;
        if (type == boolean.class) return false;
        if (type == double.class) return 0.0;
        return null;
    }
}
```

#### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/web/servlet/DispatcherServlet.java`

```java
package com.iris.framework.web.servlet;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.web.http.converter.HttpMessageConverter;

/**
 * Central dispatcher for HTTP request handling — the <strong>front controller</strong>
 * of the Iris MVC framework.
 *
 * <p>The DispatcherServlet is the single entry point for all HTTP requests.
 * It receives each request, finds the appropriate handler (via {@link HandlerMapping}),
 * invokes it (via {@link HandlerAdapter}), and the handler writes the response.
 *
 * <h3>Initialization</h3>
 * <p>The DispatcherServlet is created by {@code ServletWebServerApplicationContext}
 * during {@code onRefresh()} and registered with Tomcat. When Tomcat starts
 * (after all beans are instantiated), it calls {@link #init()} which triggers
 * strategy initialization:
 * <ul>
 *   <li>{@link #initHandlerMappings()} — creates a {@link RequestMappingHandlerMapping}
 *       that scans all {@code @Controller} beans</li>
 *   <li>{@link #initHandlerAdapters()} — creates a {@link RequestMappingHandlerAdapter}
 *       that knows how to invoke handler methods</li>
 * </ul>
 *
 * <h3>Request Processing</h3>
 * <p>For each request, {@link #service(HttpServletRequest, HttpServletResponse)}
 * delegates to {@link #doDispatch(HttpServletRequest, HttpServletResponse)}:
 * <ol>
 *   <li>Find handler via HandlerMapping</li>
 *   <li>Find adapter via HandlerAdapter.supports()</li>
 *   <li>Invoke handler via HandlerAdapter.handle()</li>
 *   <li>If no handler found, return 404</li>
 * </ol>
 *
 * <h3>Simplifications from Real Spring</h3>
 * <ul>
 *   <li>No {@code HandlerExecutionChain} / interceptors</li>
 *   <li>No {@code ModelAndView} / view resolution</li>
 *   <li>No {@code HandlerExceptionResolver} — exceptions become 500</li>
 *   <li>No multipart resolution, locale resolution, theme resolution</li>
 *   <li>No {@code FlashMap} support</li>
 *   <li>Single HandlerMapping and HandlerAdapter (not lists)</li>
 * </ul>
 *
 * @see org.springframework.web.servlet.DispatcherServlet
 */
public class DispatcherServlet extends HttpServlet {

    private final ApplicationContext applicationContext;

    private HandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    /**
     * Create a new DispatcherServlet backed by the given ApplicationContext.
     *
     * <p>The context is stored but not used until {@link #init()} is called
     * by the servlet container (Tomcat). By that time, all beans have been
     * instantiated by the context's {@code refresh()} method.
     */
    public DispatcherServlet(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    // -----------------------------------------------------------------------
    // Initialization — called by Tomcat on startup
    // -----------------------------------------------------------------------

    /**
     * Initialize the servlet's strategies (HandlerMapping, HandlerAdapter).
     *
     * <p>In the real Spring Framework, {@code DispatcherServlet.init()} calls
     * {@code initWebApplicationContext()} → {@code onRefresh()} →
     * {@code initStrategies()}, which discovers strategy beans from the
     * context. We create them directly since we have one of each.
     */
    @Override
    public void init() throws ServletException {
        initHandlerMappings();
        initHandlerAdapters();
    }

    /**
     * Initialize the HandlerMapping strategy.
     *
     * <p>Creates a {@link RequestMappingHandlerMapping} which scans the
     * ApplicationContext for {@code @Controller} beans and registers their
     * handler methods.
     */
    private void initHandlerMappings() {
        this.handlerMapping = new RequestMappingHandlerMapping(applicationContext);
    }

    /**
     * Initialize the HandlerAdapter strategy.
     *
     * <p>Creates a {@link RequestMappingHandlerAdapter} with any
     * {@link HttpMessageConverter} beans found in the ApplicationContext.
     * These converters enable {@code @ResponseBody} serialization
     * (e.g., Jackson for JSON output).
     */
    private void initHandlerAdapters() {
        List<HttpMessageConverter> converters = findMessageConverters();
        this.handlerAdapter = new RequestMappingHandlerAdapter(converters);
    }

    /**
     * Discover {@link HttpMessageConverter} beans registered in the
     * ApplicationContext.
     *
     * <p>In the real Spring Framework, the message converters are configured
     * by {@code WebMvcAutoConfiguration} and can be customized via
     * {@code WebMvcConfigurer.configureMessageConverters()}. Our simplified
     * version simply looks up all converter beans by type.
     */
    private List<HttpMessageConverter> findMessageConverters() {
        try {
            DefaultBeanFactory factory =
                    ((AnnotationConfigApplicationContext) applicationContext).getBeanFactory();
            Map<String, HttpMessageConverter> converterBeans =
                    factory.getBeansOfType(HttpMessageConverter.class);
            return new ArrayList<>(converterBeans.values());
        } catch (Exception e) {
            return List.of();
        }
    }

    // -----------------------------------------------------------------------
    // Request processing
    // -----------------------------------------------------------------------

    /**
     * Process an incoming HTTP request.
     *
     * <p>Overrides {@code HttpServlet.service()} to handle ALL HTTP methods
     * through our dispatch pipeline. In the real Spring Framework,
     * {@code FrameworkServlet.service()} overrides this and dispatches to
     * {@code processRequest()} → {@code doService()} → {@code doDispatch()}.
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            if (ex instanceof ServletException se) {
                throw se;
            }
            if (ex instanceof IOException ioe) {
                throw ioe;
            }
            throw new ServletException("Request processing failed", ex);
        }
    }

    /**
     * The central dispatch method — the heart of Spring MVC.
     *
     * <p>This is the simplified version of {@code DispatcherServlet.doDispatch()}
     * (line 935 in the real Spring Framework). The real method has 9 steps
     * including multipart handling, interceptors, async support, and exception
     * resolution. We implement the core 3-step flow:
     * <ol>
     *   <li>Find handler (HandlerMapping)</li>
     *   <li>Find adapter (HandlerAdapter)</li>
     *   <li>Invoke handler (HandlerAdapter.handle())</li>
     * </ol>
     */
    private void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        // Step 1: Find handler
        HandlerMethod handler = handlerMapping.getHandler(request);
        if (handler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    "No handler found for " + request.getMethod() + " " + request.getRequestURI());
            return;
        }

        // Step 2: Find adapter (we have only one, but follow the pattern)
        if (!handlerAdapter.supports(handler)) {
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
                    "No adapter for handler: " + handler);
            return;
        }

        // Step 3: Invoke handler
        handlerAdapter.handle(request, response, handler);
    }

    /**
     * Return the ApplicationContext this servlet is associated with.
     */
    public ApplicationContext getApplicationContext() {
        return applicationContext;
    }
}
```

#### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/web/servlet/RequestMappingHandlerMapping.java`

```java
package com.iris.framework.web.servlet;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import jakarta.servlet.http.HttpServletRequest;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.core.annotation.AnnotationUtils;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.web.bind.annotation.GetMapping;
import com.iris.framework.web.bind.annotation.PostMapping;
import com.iris.framework.web.bind.annotation.RequestMapping;
import com.iris.framework.web.bind.annotation.RequestMethod;

/**
 * Implementation of {@link HandlerMapping} that scans {@code @Controller}
 * beans for {@code @RequestMapping}, {@code @GetMapping}, and
 * {@code @PostMapping} annotations and builds a registry of URL patterns
 * to handler methods.
 *
 * <p>This is the heart of the MVC routing system. During initialization,
 * it performs three steps:
 * <ol>
 *   <li>Iterate all beans in the ApplicationContext</li>
 *   <li>For each bean annotated with {@code @Controller}, scan its methods</li>
 *   <li>For each annotated method, create a {@link RequestMappingInfo} and
 *       register it with the corresponding {@link HandlerMethod}</li>
 * </ol>
 *
 * <p>During request handling, it iterates the registry and returns the first
 * matching handler, setting path variables as a request attribute.
 *
 * <h3>Key Design: Scanning at Initialization Time</h3>
 * <p>The real Spring {@code RequestMappingHandlerMapping} extends
 * {@code AbstractHandlerMethodMapping} which implements {@code InitializingBean}.
 * The scanning happens in {@code afterPropertiesSet()} → {@code initHandlerMethods()}.
 * We trigger the scan from the constructor for simplicity.
 *
 * <h3>Simplifications from Real Spring</h3>
 * <ul>
 *   <li>No {@code HandlerExecutionChain} (no interceptors)</li>
 *   <li>No content negotiation or condition matching beyond path + method</li>
 *   <li>No {@code @AliasFor} — composed annotations are checked independently</li>
 *   <li>Simple segment-based path matching instead of {@code PathPattern}</li>
 * </ul>
 *
 * @see org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
 */
public class RequestMappingHandlerMapping implements HandlerMapping {

    /** Registry of request mapping info → handler method. */
    private final List<MappingRegistration> registry = new ArrayList<>();

    /**
     * Create a new handler mapping by scanning the given ApplicationContext
     * for {@code @Controller} beans.
     *
     * @param context the application context to scan
     */
    public RequestMappingHandlerMapping(ApplicationContext context) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        for (String name : factory.getBeanDefinitionNames()) {
            Object bean = factory.getBean(name);
            if (isController(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    /**
     * Find the handler method for the given request.
     *
     * <p>Iterates the registered mappings and returns the first match.
     * If matched, sets the {@link #PATH_VARIABLES_ATTRIBUTE} request
     * attribute with the extracted path variables.
     */
    @Override
    public HandlerMethod getHandler(HttpServletRequest request) {
        String path = getRequestPath(request);
        String method = request.getMethod();

        for (MappingRegistration registration : registry) {
            Map<String, String> pathVariables = registration.info.match(path, method);
            if (pathVariables != null) {
                request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);
                return registration.handler;
            }
        }
        return null;
    }

    /**
     * Return the number of registered handler methods (useful for testing).
     */
    public int getHandlerMethodCount() {
        return registry.size();
    }

    // -----------------------------------------------------------------------
    // Scanning logic
    // -----------------------------------------------------------------------

    /**
     * Check if the given class is a controller — annotated with
     * {@code @Controller} (directly or via meta-annotation like
     * {@code @RestController}).
     *
     * <p>Uses recursive meta-annotation detection so that composed
     * annotations like {@code @RestController} (which carries
     * {@code @Controller}) are recognized.
     */
    private boolean isController(Class<?> clazz) {
        return AnnotationUtils.hasAnnotation(clazz, Controller.class);
    }

    /**
     * Scan the controller bean's methods for request mapping annotations.
     *
     * <p>For each annotated method, combines the class-level prefix (from
     * {@code @RequestMapping} on the class) with the method-level path.
     */
    private void detectHandlerMethods(Object bean) {
        Class<?> clazz = bean.getClass();
        String classPrefix = getClassLevelPath(clazz);

        for (Method method : clazz.getDeclaredMethods()) {
            List<RequestMappingInfo> infos = getMappingInfos(method, classPrefix);
            for (RequestMappingInfo info : infos) {
                registry.add(new MappingRegistration(info, new HandlerMethod(bean, method)));
            }
        }
    }

    /**
     * Get the class-level path prefix from {@code @RequestMapping} on the class.
     * Returns empty string if not present.
     */
    private String getClassLevelPath(Class<?> clazz) {
        RequestMapping rm = clazz.getAnnotation(RequestMapping.class);
        if (rm != null) {
            String path = getFirstPath(rm.value(), rm.path());
            if (path != null) {
                return path;
            }
        }
        return "";
    }

    /**
     * Extract {@link RequestMappingInfo}s from a handler method's annotations.
     *
     * <p>Checks for {@code @RequestMapping}, {@code @GetMapping}, and
     * {@code @PostMapping} independently (no meta-annotation resolution).
     */
    private List<RequestMappingInfo> getMappingInfos(Method method, String classPrefix) {
        List<RequestMappingInfo> infos = new ArrayList<>();

        // Check @RequestMapping
        RequestMapping rm = method.getAnnotation(RequestMapping.class);
        if (rm != null) {
            String path = classPrefix + getPathOrDefault(rm.value(), rm.path());
            RequestMethod[] methods = rm.method();
            if (methods.length == 0) {
                // No method restriction — matches all methods
                infos.add(new RequestMappingInfo(path, null));
            } else {
                for (RequestMethod m : methods) {
                    infos.add(new RequestMappingInfo(path, m));
                }
            }
        }

        // Check @GetMapping
        GetMapping gm = method.getAnnotation(GetMapping.class);
        if (gm != null) {
            String path = classPrefix + getPathOrDefault(gm.value(), new String[0]);
            infos.add(new RequestMappingInfo(path, RequestMethod.GET));
        }

        // Check @PostMapping
        PostMapping pm = method.getAnnotation(PostMapping.class);
        if (pm != null) {
            String path = classPrefix + getPathOrDefault(pm.value(), new String[0]);
            infos.add(new RequestMappingInfo(path, RequestMethod.POST));
        }

        return infos;
    }

    // -----------------------------------------------------------------------
    // Path helpers
    // -----------------------------------------------------------------------

    /**
     * Extract the request path from the request URI, stripping the query string.
     */
    private String getRequestPath(HttpServletRequest request) {
        String uri = request.getRequestURI();
        // Remove context path if present
        String contextPath = request.getContextPath();
        if (contextPath != null && !contextPath.isEmpty() && uri.startsWith(contextPath)) {
            uri = uri.substring(contextPath.length());
        }
        return uri;
    }

    /**
     * Return the first non-empty path from the value or path arrays.
     * Returns "/" if both are empty.
     */
    private String getPathOrDefault(String[] value, String[] path) {
        String result = getFirstPath(value, path);
        return result != null ? result : "";
    }

    /**
     * Return the first non-empty path from two arrays, or null.
     */
    private String getFirstPath(String[] value, String[] path) {
        if (value.length > 0 && !value[0].isEmpty()) {
            return value[0];
        }
        if (path.length > 0 && !path[0].isEmpty()) {
            return path[0];
        }
        return null;
    }

    // -----------------------------------------------------------------------
    // Internal registry entry
    // -----------------------------------------------------------------------

    /**
     * A registration entry pairing mapping info with a handler method.
     */
    private record MappingRegistration(RequestMappingInfo info, HandlerMethod handler) {
    }
}
```

#### [MODIFIED] `iris-framework/src/main/java/com/iris/framework/context/annotation/ClassPathScanner.java`

```java
package com.iris.framework.context.annotation;

import java.io.File;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.net.URL;
import java.util.Enumeration;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.annotation.AnnotationUtils;
import com.iris.framework.stereotype.Component;

/**
 * Scans the classpath for classes annotated with {@link Component @Component}
 * (or any stereotype meta-annotated with {@code @Component}) and registers
 * them as {@link BeanDefinition}s in the given registry.
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code ClassPathBeanDefinitionScanner} (which extends
 * {@code ClassPathScanningCandidateComponentProvider}). The real scanner:
 * <ol>
 *   <li>Uses ASM-based metadata reading ({@code MetadataReader}) to inspect
 *       {@code .class} files <em>without loading them</em> via the ClassLoader,
 *       avoiding side effects from static initializers</li>
 *   <li>Supports include/exclude filters ({@code AnnotationTypeFilter},
 *       {@code AssignableTypeFilter}, regex, AspectJ, custom)</li>
 *   <li>Supports a compile-time component index
 *       ({@code META-INF/spring.components}) as an alternative to runtime
 *       classpath scanning</li>
 *   <li>Handles scoped proxy generation for non-singleton beans</li>
 *   <li>Detects conflicts between scanned beans and explicitly registered beans</li>
 * </ol>
 *
 * <p>Our simplification:
 * <ul>
 *   <li>Uses {@code Class.forName(name, false, classLoader)} — loads the class
 *       without initialization, but still heavier than ASM bytecode reading</li>
 *   <li>Single include filter: {@code @Component} (including meta-annotations)</li>
 *   <li>No compile-time index — runtime scanning only</li>
 *   <li>No scoped proxy support — singleton scope only</li>
 *   <li>Simple conflict resolution: skip if name already registered</li>
 *   <li>Skips inner classes (files containing {@code $})</li>
 * </ul>
 *
 * @see Component
 * @see ComponentScan
 * @see ConfigurationClassProcessor
 * @see org.springframework.context.annotation.ClassPathBeanDefinitionScanner
 * @see org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider
 */
public class ClassPathScanner {

    /**
     * Scan the given base packages for {@code @Component}-annotated classes
     * and register them as {@link BeanDefinition}s in the given registry.
     *
     * <p>This maps to Spring's {@code ClassPathBeanDefinitionScanner.scan()}
     * which delegates to {@code doScan()} for each base package. The return
     * value is the total count of newly registered bean definitions.
     *
     * @param registry     the bean definition registry to register into
     * @param basePackages the packages to scan (e.g., {@code "com.example.app"})
     * @return the number of newly registered bean definitions
     */
    public int scan(BeanDefinitionRegistry registry, String... basePackages) {
        int count = 0;
        for (String basePackage : basePackages) {
            count += scanPackage(registry, basePackage);
        }
        return count;
    }

    /**
     * Scan a single base package by converting the package name to a resource
     * path, using the ClassLoader to locate the directory, and walking it
     * recursively for {@code .class} files.
     *
     * <p>This maps to Spring's pipeline:
     * {@code ClassPathScanningCandidateComponentProvider.scanCandidateComponents()}
     * which resolves the package to a classpath pattern like
     * {@code classpath*:com/example/**&#47;*.class} and uses
     * {@code PathMatchingResourcePatternResolver} to find resources.
     *
     * <p>We simplify by using {@code ClassLoader.getResources()} to find the
     * directory and walking the file system directly. This works for file-system
     * classpath entries but not for classes inside JAR files.
     */
    private int scanPackage(BeanDefinitionRegistry registry, String basePackage) {
        String resourcePath = basePackage.replace('.', '/');
        ClassLoader classLoader = getClassLoader();

        try {
            Enumeration<URL> resources = classLoader.getResources(resourcePath);
            int count = 0;

            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                if ("file".equals(resource.getProtocol())) {
                    File directory = new File(resource.toURI());
                    count += scanDirectory(registry, directory, basePackage);
                }
            }

            return count;
        } catch (Exception e) {
            throw new BeanCreationException("component-scan",
                    "Failed to scan package '" + basePackage + "'", e);
        }
    }

    /**
     * Recursively walk a directory, loading each {@code .class} file and
     * checking if it's a component candidate.
     *
     * <p>Filters applied:
     * <ul>
     *   <li>Skip inner classes ({@code $} in filename) — they depend on an
     *       enclosing instance and aren't independent components</li>
     *   <li>Skip {@code module-info.class} and {@code package-info.class}</li>
     *   <li>Recurse into subdirectories (sub-packages)</li>
     * </ul>
     */
    private int scanDirectory(BeanDefinitionRegistry registry, File dir, String packageName) {
        int count = 0;
        File[] files = dir.listFiles();
        if (files == null) {
            return 0;
        }

        for (File file : files) {
            if (file.isDirectory()) {
                // Recurse into sub-packages
                count += scanDirectory(registry, file, packageName + "." + file.getName());
            } else if (isClassFile(file.getName())) {
                String className = packageName + "."
                        + file.getName().substring(0, file.getName().length() - ".class".length());
                if (processCandidate(registry, className)) {
                    count++;
                }
            }
        }

        return count;
    }

    /**
     * Check if a filename represents a scannable {@code .class} file.
     * Excludes inner classes, module-info, and package-info.
     */
    private boolean isClassFile(String fileName) {
        return fileName.endsWith(".class")
                && !fileName.contains("$")
                && !fileName.equals("module-info.class")
                && !fileName.equals("package-info.class");
    }

    /**
     * Load a class by name, check if it's a component candidate, and if so,
     * register a {@link BeanDefinition} for it.
     *
     * <p>Uses {@code Class.forName(name, false, classLoader)} which loads the
     * class without running its static initializers. This is safer than
     * {@code Class.forName(name)} but still heavier than Spring's ASM-based
     * approach which never loads the class at all.
     *
     * <p>In the real Spring Framework, this is a two-phase check:
     * <ol>
     *   <li>Phase 1: {@code isCandidateComponent(MetadataReader)} — annotation check
     *       (exclude filters first, then include filters)</li>
     *   <li>Phase 2: {@code isCandidateComponent(AnnotatedBeanDefinition)} — structural
     *       check (concrete, independent, not a CGLIB proxy)</li>
     * </ol>
     *
     * @return true if the class was registered, false if skipped
     */
    private boolean processCandidate(BeanDefinitionRegistry registry, String className) {
        try {
            Class<?> clazz = Class.forName(className, false, getClassLoader());

            if (!isCandidate(clazz)) {
                return false;
            }

            String beanName = resolveBeanName(clazz);
            if (registry.containsBeanDefinition(beanName)) {
                return false;
            }

            BeanDefinition bd = new BeanDefinition(clazz);
            registry.registerBeanDefinition(beanName, bd);
            return true;
        } catch (ClassNotFoundException | NoClassDefFoundError e) {
            // Class not loadable — skip silently
            return false;
        }
    }

    /**
     * Determine if a class is a component candidate:
     * <ol>
     *   <li>Must have {@code @Component} — directly or as a meta-annotation</li>
     *   <li>Must be concrete (not an interface, not abstract)</li>
     * </ol>
     *
     * <p>In the real Spring Framework, the structural check also allows
     * abstract classes with {@code @Lookup} methods (which Spring implements
     * via CGLIB). We require concrete classes only.
     */
    private boolean isCandidate(Class<?> clazz) {
        // Must be concrete
        if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
            return false;
        }

        // Must have @Component (directly or as a meta-annotation)
        return hasComponentAnnotation(clazz);
    }

    /**
     * Check whether the given class has {@code @Component} either directly
     * or as a meta-annotation on one of its annotations.
     *
     * <p>This is the simplified equivalent of Spring's
     * {@code AnnotationTypeFilter.matchSelf()} which checks both
     * {@code hasAnnotation()} and {@code hasMetaAnnotation()} via the
     * ASM-based {@code AnnotationMetadata}.
     *
     * <p>The key insight: only ONE filter check is needed for the entire
     * stereotype hierarchy. {@code @Service}, {@code @Controller}, and
     * {@code @Repository} all carry {@code @Component} as a meta-annotation,
     * so this single check catches all of them.
     *
     * @see org.springframework.core.type.filter.AnnotationTypeFilter#matchSelf
     */
    static boolean hasComponentAnnotation(Class<?> clazz) {
        // Use recursive meta-annotation detection so that composed
        // annotations like @RestController (→ @Controller → @Component)
        // are detected at any depth.
        return AnnotationUtils.hasAnnotation(clazz, Component.class);
    }

    /**
     * Resolve the bean name for a scanned component class.
     *
     * <p>Resolution order:
     * <ol>
     *   <li>If {@code @Component("explicitName")} → use that name</li>
     *   <li>If a stereotype annotation has a non-empty {@code value()} →
     *       use that (e.g., {@code @Service("myService")})</li>
     *   <li>Otherwise → decapitalize the simple class name
     *       ({@code MyService} → {@code "myService"})</li>
     * </ol>
     *
     * <p>In the real Spring Framework, this is handled by
     * {@code AnnotationBeanNameGenerator.determineBeanNameFromAnnotation()}
     * which uses Spring's merged annotation infrastructure and
     * {@code @AliasFor} to resolve names across the stereotype hierarchy.
     * We use direct reflection on the {@code value()} method instead.
     *
     * @see org.springframework.context.annotation.AnnotationBeanNameGenerator
     */
    static String resolveBeanName(Class<?> clazz) {
        // 1. Direct @Component with explicit name
        Component component = clazz.getAnnotation(Component.class);
        if (component != null && !component.value().isEmpty()) {
            return component.value();
        }

        // 2. Check stereotype annotations (meta-annotated with @Component at any depth)
        for (Annotation annotation : clazz.getAnnotations()) {
            if (AnnotationUtils.hasAnnotation(annotation.annotationType(), Component.class)) {
                try {
                    Method valueMethod = annotation.annotationType().getMethod("value");
                    String value = (String) valueMethod.invoke(annotation);
                    if (value != null && !value.isEmpty()) {
                        return value;
                    }
                } catch (Exception e) {
                    // No value() method or invocation failed — fall through to default
                }
            }
        }

        // 3. Default: decapitalize the simple class name
        return decapitalize(clazz.getSimpleName());
    }

    /**
     * Decapitalize a name following the JavaBeans {@code Introspector.decapitalize()}
     * convention: lowercase the first character, UNLESS the first two characters
     * are both uppercase (e.g., {@code "URLService"} stays {@code "URLService"}).
     *
     * <p>This matches Spring's {@code AnnotationBeanNameGenerator.buildDefaultBeanName()}
     * which delegates to {@code java.beans.Introspector.decapitalize()}.
     *
     * @see java.beans.Introspector#decapitalize(String)
     */
    static String decapitalize(String name) {
        if (name.isEmpty()) {
            return name;
        }
        // JavaBeans spec: if first two chars are uppercase, keep as-is
        if (name.length() > 1
                && Character.isUpperCase(name.charAt(0))
                && Character.isUpperCase(name.charAt(1))) {
            return name;
        }
        return Character.toLowerCase(name.charAt(0)) + name.substring(1);
    }

    /**
     * Get the ClassLoader to use for loading classes and resources.
     * Prefers the thread's context ClassLoader, falling back to the
     * ClassLoader that loaded this scanner class.
     */
    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return (cl != null) ? cl : ClassPathScanner.class.getClassLoader();
    }
}
```

### Test Code

#### [NEW] `iris-framework/src/test/java/com/iris/framework/core/annotation/AnnotationUtilsTest.java`

```java
package com.iris.framework.core.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.junit.jupiter.api.Test;

import com.iris.framework.stereotype.Component;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.bind.annotation.RestController;

import static org.assertj.core.api.Assertions.assertThat;

class AnnotationUtilsTest {

    @Test
    void shouldDetectDirectAnnotation() {
        assertThat(AnnotationUtils.hasAnnotation(DirectComponent.class, Component.class))
                .isTrue();
    }

    @Test
    void shouldDetectOneLevel_WhenMetaAnnotatedWithComponent() {
        assertThat(AnnotationUtils.hasAnnotation(ControllerClass.class, Component.class))
                .isTrue();
    }

    @Test
    void shouldDetectTwoLevels_WhenRestControllerUsed() {
        assertThat(AnnotationUtils.hasAnnotation(RestControllerClass.class, Component.class))
                .isTrue();
    }

    @Test
    void shouldDetectControllerViaRestController() {
        assertThat(AnnotationUtils.hasAnnotation(RestControllerClass.class, Controller.class))
                .isTrue();
    }

    @Test
    void shouldDetectResponseBodyViaRestController() {
        assertThat(AnnotationUtils.hasAnnotation(RestControllerClass.class, ResponseBody.class))
                .isTrue();
    }

    @Test
    void shouldReturnFalse_WhenAnnotationNotPresent() {
        assertThat(AnnotationUtils.hasAnnotation(PlainClass.class, Component.class))
                .isFalse();
    }

    @Test
    void shouldReturnFalse_WhenUnrelatedAnnotation() {
        assertThat(AnnotationUtils.hasAnnotation(ControllerClass.class, ResponseBody.class))
                .isFalse();
    }

    @Test
    void shouldDetectAnnotationOnMethod() throws NoSuchMethodException {
        assertThat(AnnotationUtils.hasAnnotation(
                MethodAnnotatedClass.class.getMethod("annotatedMethod"),
                ResponseBody.class))
                .isTrue();
    }

    @Test
    void shouldHandleCustomMetaAnnotationDepth() {
        assertThat(AnnotationUtils.hasAnnotation(DeeplyAnnotatedClass.class, Component.class))
                .isTrue();
    }

    // --- Test fixtures ---

    @Component
    static class DirectComponent {}

    @Controller
    static class ControllerClass {}

    @RestController
    static class RestControllerClass {}

    static class PlainClass {}

    static class MethodAnnotatedClass {
        @ResponseBody
        public String annotatedMethod() { return "test"; }
    }

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Component
    @interface Level1 {}

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Level1
    @interface Level2 {}

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Level2
    @interface Level3 {}

    @Level3
    static class DeeplyAnnotatedClass {}
}
```

#### [NEW] `iris-framework/src/test/java/com/iris/framework/web/http/converter/HttpMessageConverterTest.java`

```java
package com.iris.framework.web.http.converter;

import java.io.IOException;
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import com.iris.framework.web.bind.annotation.GetMapping;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.bind.annotation.RestController;
import com.iris.framework.web.servlet.HandlerMethod;
import com.iris.framework.web.servlet.RequestMappingHandlerAdapter;
import com.iris.framework.web.servlet.mock.MockHttpServletRequest;
import com.iris.framework.web.servlet.mock.MockHttpServletResponse;
import com.iris.framework.stereotype.Controller;

import jakarta.servlet.http.HttpServletResponse;

import static org.assertj.core.api.Assertions.assertThat;

class HttpMessageConverterTest {

    @Test
    void shouldUseConverter_WhenMethodHasResponseBody() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(ResponseBodyController.class, "getMessage");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/message");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[Hello]");
        assertThat(response.getContentType()).isEqualTo("text/brackets");
    }

    @Test
    void shouldUseConverter_WhenClassHasRestController() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(RestApiController.class, "getData");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/data");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[some data]");
        assertThat(response.getContentType()).isEqualTo("text/brackets");
    }

    @Test
    void shouldUsePlainText_WhenNoResponseBody() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(PlainController.class, "getText");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/text");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("plain text");
        assertThat(response.getContentType()).isEqualTo("text/plain;charset=UTF-8");
    }

    @Test
    void shouldFallbackToPlainText_WhenNoConverterAvailable() throws Exception {
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of());

        HandlerMethod handler = createHandler(ResponseBodyController.class, "getMessage");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/message");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("Hello");
        assertThat(response.getContentType()).isEqualTo("text/plain;charset=UTF-8");
    }

    @Test
    void shouldSkipIncompatibleConverter_WhenCanWriteReturnsFalse() throws Exception {
        HttpMessageConverter mapOnlyConverter = new MapOnlyConverter();
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(mapOnlyConverter, bracketsConverter));

        HandlerMethod handler = createHandler(ResponseBodyController.class, "getMessage");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/message");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[Hello]");
    }

    @Test
    void shouldUseClassLevelResponseBody_WhenMethodLacksAnnotation() throws Exception {
        HttpMessageConverter bracketsConverter = new BracketsConverter();
        RequestMappingHandlerAdapter adapter =
                new RequestMappingHandlerAdapter(List.of(bracketsConverter));

        HandlerMethod handler = createHandler(ClassLevelResponseBodyController.class, "getValue");
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/value");
        MockHttpServletResponse response = new MockHttpServletResponse();

        adapter.handle(request, response, handler);

        assertThat(response.getContentAsString()).isEqualTo("[test value]");
    }

    // --- Test converters ---

    static class BracketsConverter implements HttpMessageConverter {
        @Override
        public boolean canWrite(Class<?> clazz) { return true; }

        @Override
        public void write(Object value, HttpServletResponse response) throws IOException {
            response.setContentType("text/brackets");
            response.getWriter().write("[" + value + "]");
            response.getWriter().flush();
        }
    }

    static class MapOnlyConverter implements HttpMessageConverter {
        @Override
        public boolean canWrite(Class<?> clazz) {
            return Map.class.isAssignableFrom(clazz);
        }

        @Override
        public void write(Object value, HttpServletResponse response) throws IOException {
            response.setContentType("text/map");
            response.getWriter().write("MAP:" + value);
            response.getWriter().flush();
        }
    }

    // --- Test controllers ---

    @Controller
    static class ResponseBodyController {
        @ResponseBody
        public String getMessage() { return "Hello"; }
    }

    @RestController
    static class RestApiController {
        public String getData() { return "some data"; }
    }

    @Controller
    static class PlainController {
        public String getText() { return "plain text"; }
    }

    @Controller
    @ResponseBody
    static class ClassLevelResponseBodyController {
        public String getValue() { return "test value"; }
    }

    // --- Helpers ---

    private HandlerMethod createHandler(Class<?> controllerClass, String methodName) {
        try {
            Object controller = controllerClass.getDeclaredConstructor().newInstance();
            for (Method m : controllerClass.getDeclaredMethods()) {
                if (m.getName().equals(methodName)) {
                    return new HandlerMethod(controller, m);
                }
            }
            throw new IllegalArgumentException("No method named: " + methodName);
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException(e);
        }
    }
}
```

#### [NEW] `iris-boot-core/src/test/java/com/iris/boot/web/http/converter/JacksonHttpMessageConverterTest.java`

```java
package com.iris.boot.web.http.converter;

import java.io.IOException;
import java.io.PrintWriter;
import java.io.StringWriter;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import static org.assertj.core.api.Assertions.assertThat;

class JacksonHttpMessageConverterTest {

    private final JacksonHttpMessageConverter converter = new JacksonHttpMessageConverter();

    @Test
    void shouldCanWriteAnyClass() {
        assertThat(converter.canWrite(String.class)).isTrue();
        assertThat(converter.canWrite(UserDto.class)).isTrue();
        assertThat(converter.canWrite(Object.class)).isTrue();
    }

    @Test
    void shouldSerializePojo_WhenWriteCalled() throws IOException {
        UserDto user = new UserDto(1, "Alice", "alice@example.com");
        SimpleResponse response = new SimpleResponse();

        converter.write(user, response);

        String json = response.getBody();
        assertThat(json).contains("\"id\":1");
        assertThat(json).contains("\"name\":\"Alice\"");
        assertThat(json).contains("\"email\":\"alice@example.com\"");
        assertThat(response.contentType).isEqualTo("application/json;charset=UTF-8");
    }

    @Test
    void shouldSerializeRecord_WhenWriteCalled() throws IOException {
        ProductRecord product = new ProductRecord("Widget", 9.99);
        SimpleResponse response = new SimpleResponse();

        converter.write(product, response);

        String json = response.getBody();
        assertThat(json).contains("\"name\":\"Widget\"");
        assertThat(json).contains("\"price\":9.99");
    }

    @Test
    void shouldSerializeString_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write("hello", response);

        assertThat(response.getBody()).isEqualTo("\"hello\"");
        assertThat(response.contentType).isEqualTo("application/json;charset=UTF-8");
    }

    @Test
    void shouldSerializeMap_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write(java.util.Map.of("key", "value"), response);

        assertThat(response.getBody()).contains("\"key\":\"value\"");
    }

    @Test
    void shouldSerializeList_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write(java.util.List.of(1, 2, 3), response);

        assertThat(response.getBody()).isEqualTo("[1,2,3]");
    }

    @Test
    void shouldUseCustomObjectMapper_WhenProvided() throws IOException {
        ObjectMapper customMapper = new ObjectMapper();
        JacksonHttpMessageConverter customConverter = new JacksonHttpMessageConverter(customMapper);
        SimpleResponse response = new SimpleResponse();

        customConverter.write(new UserDto(1, "Bob", null), response);

        String json = response.getBody();
        assertThat(json).contains("\"name\":\"Bob\"");
        assertThat(customConverter.getObjectMapper()).isSameAs(customMapper);
    }

    @Test
    void shouldHandleNull_WhenWriteCalled() throws IOException {
        SimpleResponse response = new SimpleResponse();

        converter.write(null, response);

        assertThat(response.getBody()).isEqualTo("null");
    }

    // --- Minimal response mock ---

    static class SimpleResponse extends NoOpHttpServletResponse {
        private final StringWriter writer = new StringWriter();
        private final PrintWriter printWriter = new PrintWriter(writer);
        String contentType;

        @Override
        public void setContentType(String type) { this.contentType = type; }

        @Override
        public String getContentType() { return contentType; }

        @Override
        public PrintWriter getWriter() { return printWriter; }

        String getBody() {
            printWriter.flush();
            return writer.toString();
        }
    }

    // --- Test fixtures ---

    static class UserDto {
        private int id;
        private String name;
        private String email;

        public UserDto() {}
        public UserDto(int id, String name, String email) {
            this.id = id; this.name = name; this.email = email;
        }

        public int getId() { return id; }
        public void setId(int id) { this.id = id; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    record ProductRecord(String name, double price) {}
}
```

#### [NEW] `iris-boot-core/src/test/java/com/iris/boot/integration/JsonResponseBodyIntegrationTest.java`

```java
package com.iris.boot.integration;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.http.converter.JacksonHttpMessageConverter;
import com.iris.boot.web.context.ServletWebServerApplicationContext;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.web.bind.annotation.GetMapping;
import com.iris.framework.web.bind.annotation.PathVariable;
import com.iris.framework.web.bind.annotation.ResponseBody;
import com.iris.framework.web.bind.annotation.RestController;
import com.iris.framework.web.http.converter.HttpMessageConverter;

import static org.assertj.core.api.Assertions.assertThat;

class JsonResponseBodyIntegrationTest {

    private ServletWebServerApplicationContext context;
    private HttpClient httpClient;
    private ObjectMapper objectMapper;
    private int port;

    @BeforeEach
    void setUp() {
        context = new ServletWebServerApplicationContext();
        context.register(JsonTestConfig.class);
        context.refresh();

        port = context.getWebServer().getPort();
        httpClient = HttpClient.newHttpClient();
        objectMapper = new ObjectMapper();
    }

    @AfterEach
    void cleanup() {
        if (context != null) {
            context.close();
        }
    }

    @Test
    void shouldReturnJson_WhenRestControllerReturnsObject() throws Exception {
        HttpResponse<String> response = get("/api/users/1");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("application/json"));

        Map<?, ?> user = objectMapper.readValue(response.body(), Map.class);
        assertThat(user.get("id")).isEqualTo(1);
        assertThat(user.get("name")).isEqualTo("Alice");
    }

    @Test
    void shouldReturnJsonArray_WhenRestControllerReturnsList() throws Exception {
        HttpResponse<String> response = get("/api/users");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("application/json"));

        List<?> users = objectMapper.readValue(response.body(), List.class);
        assertThat(users).hasSize(2);
    }

    @Test
    void shouldReturnJsonRecord_WhenRestControllerReturnsRecord() throws Exception {
        HttpResponse<String> response = get("/api/products/42");

        assertThat(response.statusCode()).isEqualTo(200);

        Map<?, ?> product = objectMapper.readValue(response.body(), Map.class);
        assertThat(product.get("name")).isEqualTo("Widget");
        assertThat(product.get("price")).isEqualTo(19.99);
    }

    @Test
    void shouldReturnJson_WhenMethodHasResponseBody() throws Exception {
        HttpResponse<String> response = get("/mixed/json");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("application/json"));

        Map<?, ?> result = objectMapper.readValue(response.body(), Map.class);
        assertThat(result.get("status")).isEqualTo("ok");
    }

    @Test
    void shouldReturnPlainText_WhenMethodLacksResponseBody() throws Exception {
        HttpResponse<String> response = get("/mixed/text");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.headers().firstValue("Content-Type"))
                .hasValueSatisfying(ct -> assertThat(ct).contains("text/plain"));
        assertThat(response.body()).isEqualTo("just text");
    }

    @Test
    void shouldReturnJsonMap_WhenRestControllerReturnsMap() throws Exception {
        HttpResponse<String> response = get("/api/config");

        assertThat(response.statusCode()).isEqualTo(200);

        Map<?, ?> config = objectMapper.readValue(response.body(), Map.class);
        assertThat(config.get("version")).isEqualTo("1.0");
        assertThat(config.get("debug")).isEqualTo(false);
    }

    // --- Helper ---

    private HttpResponse<String> get(String path) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("http://localhost:" + port + path))
                .GET()
                .build();
        return httpClient.send(request, HttpResponse.BodyHandlers.ofString());
    }

    // --- Test configuration ---

    @Configuration
    static class JsonTestConfig {
        @Bean
        public ServletWebServerFactory webServerFactory() {
            return new TomcatServletWebServerFactory(0);
        }

        @Bean
        public HttpMessageConverter jacksonConverter() {
            return new JacksonHttpMessageConverter();
        }

        @Bean
        public UserApiController userApiController() {
            return new UserApiController();
        }

        @Bean
        public MixedController mixedController() {
            return new MixedController();
        }
    }

    // --- Test controllers ---

    @RestController
    static class UserApiController {
        @GetMapping("/api/users/{id}")
        public UserDto getUser(@PathVariable("id") int id) {
            return new UserDto(id, "Alice", "alice@example.com");
        }

        @GetMapping("/api/users")
        public List<UserDto> listUsers() {
            return List.of(
                    new UserDto(1, "Alice", "alice@example.com"),
                    new UserDto(2, "Bob", "bob@example.com"));
        }

        @GetMapping("/api/products/{id}")
        public ProductRecord getProduct(@PathVariable("id") int id) {
            return new ProductRecord("Widget", 19.99);
        }

        @GetMapping("/api/config")
        public Map<String, Object> getConfig() {
            return Map.of("version", "1.0", "debug", false);
        }
    }

    @Controller
    static class MixedController {
        @GetMapping("/mixed/json")
        @ResponseBody
        public Map<String, String> jsonEndpoint() {
            return Map.of("status", "ok");
        }

        @GetMapping("/mixed/text")
        public String textEndpoint() {
            return "just text";
        }
    }

    // --- DTOs ---

    static class UserDto {
        private int id;
        private String name;
        private String email;

        public UserDto() {}
        public UserDto(int id, String name, String email) {
            this.id = id; this.name = name; this.email = email;
        }

        public int getId() { return id; }
        public void setId(int id) { this.id = id; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    record ProductRecord(String name, double price) {}
}
```

---

## Summary

| What | File(s) |
|---|---|
| **New annotations** | `@ResponseBody`, `@RestController` |
| **Converter abstraction** | `HttpMessageConverter` (iris-framework), `JacksonHttpMessageConverter` (iris-boot-core) |
| **Meta-annotation utility** | `AnnotationUtils` |
| **Modified for @ResponseBody** | `RequestMappingHandlerAdapter` (branching + converter delegation) |
| **Modified for converter discovery** | `DispatcherServlet` (`findMessageConverters()` + `initHandlerAdapters()`) |
| **Modified for meta-annotations** | `RequestMappingHandlerMapping` (`isController()`), `ClassPathScanner` (`hasComponentAnnotation()`, `resolveBeanName()`) |
| **Tests** | `AnnotationUtilsTest`, `HttpMessageConverterTest`, `JacksonHttpMessageConverterTest`, `JsonResponseBodyIntegrationTest` |
| **Pattern** | Strategy (HttpMessageConverter) + Composed Annotations (recursive meta-annotation detection) |

**Next chapter:** [Chapter 12: Conditional Beans](ch12_conditional_beans.md) -- Add `@ConditionalOnClass` and `@ConditionalOnMissingBean` to control which beans are registered based on classpath presence and existing bean definitions, enabling auto-configuration that adapts to the application's environment.
