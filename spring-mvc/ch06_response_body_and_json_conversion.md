# Chapter 6: @ResponseBody & JSON Conversion

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Handler methods return Strings, and `SimpleHandlerAdapter.writeResponse()` always writes them as `text/plain` | Cannot return structured data (Maps, POJOs, Lists) as JSON — every response is `toString()` | Introduce `HttpMessageConverter` and `HandlerMethodReturnValueHandler` patterns so `@ResponseBody` methods automatically serialize to JSON via Jackson |

---

## 6.1 The Integration Point

The integration point is `SimpleHandlerAdapter.handle()` — specifically, replacing the hardcoded `writeResponse()` call with a pluggable return value handler chain.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Replace `writeResponse(response, result)` with `returnValueHandlers.handleReturnValue(result, handlerMethod, request, response)`

```java
@Override
public void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    HandlerMethod handlerMethod = (HandlerMethod) handler;

    // Step 1: Resolve method arguments from the request (ch05).
    Object[] args = resolveArguments(handlerMethod, request);

    // Step 2: Invoke the controller method via reflection.
    Object result = invokeHandlerMethod(handlerMethod, args);

    // Step 3: Process the return value through the handler chain.    ← NEW
    // The composite iterates handlers: @ResponseBody → JSON, else → text/plain.
    returnValueHandlers.handleReturnValue(result, handlerMethod, request, response);
}
```

And in the constructor, wire up the chain with message converters:

```java
private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
        new HandlerMethodReturnValueHandlerComposite();
private final List<HttpMessageConverter> messageConverters = new ArrayList<>();

public SimpleHandlerAdapter() {
    argumentResolvers.addResolver(new RequestParamArgumentResolver());

    // Register the default message converters.
    messageConverters.add(new JacksonMessageConverter());

    // ORDER MATTERS: @ResponseBody handler must come BEFORE the String fallback.
    returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
    returnValueHandlers.addHandler(new StringReturnValueHandler());
}
```

Two key decisions here:

1. **Why replace `writeResponse()` with a composite?** The real framework's `RequestMappingHandlerAdapter` uses exactly this pattern — `getDefaultReturnValueHandlers()` builds a prioritized list, and `invokeAndHandle()` delegates to it. By adopting the same Strategy + Composite pattern, we can add new response formats (views in ch13, SSE in ch22) without modifying the adapter.

2. **Why does order matter?** The composite uses first-match. `ResponseBodyReturnValueHandler` checks for `@ResponseBody` — if present, it serializes to JSON. `StringReturnValueHandler` is a catch-all that always matches. If we registered them in reverse order, every method would get `text/plain`, even `@ResponseBody` ones.

This connects **the handler invocation pipeline** to **the serialization subsystem**. To make it work, we need to build:
- `@ResponseBody` annotation — the marker that triggers JSON serialization
- `HttpMessageConverter` interface + `JacksonMessageConverter` — the serialization strategy
- `HandlerMethodReturnValueHandler` interface + composite — the dispatch mechanism
- `ResponseBodyReturnValueHandler` — bridges annotation detection to converter invocation
- `StringReturnValueHandler` — extracts the old `writeResponse()` behavior into a handler

## 6.2 The @ResponseBody Annotation

**New file:** `src/main/java/com/simplespringmvc/annotation/ResponseBody.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ResponseBody {
}
```

Note `@Target({TYPE, METHOD})` — it can go on a class (all methods become JSON) or on individual methods. Class-level `@ResponseBody` is the foundation for `@RestController` in ch10.

## 6.3 The HttpMessageConverter Pattern

**New file:** `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java`

```java
package com.simplespringmvc.converter;

import jakarta.servlet.http.HttpServletResponse;

public interface HttpMessageConverter {

    boolean canWrite(Class<?> clazz);

    void write(Object value, HttpServletResponse response) throws Exception;
}
```

**New file:** `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java`

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletResponse;

public class JacksonMessageConverter implements HttpMessageConverter {

    private final ObjectMapper objectMapper;

    public JacksonMessageConverter() {
        this.objectMapper = new ObjectMapper();
    }

    public JacksonMessageConverter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean canWrite(Class<?> clazz) {
        return true;  // Jackson can serialize most objects
    }

    @Override
    public void write(Object value, HttpServletResponse response) throws Exception {
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        objectMapper.writeValue(response.getWriter(), value);
    }

    public ObjectMapper getObjectMapper() {
        return objectMapper;
    }
}
```

## 6.4 The ReturnValueHandler Chain

**New file:** `src/main/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(HandlerMethod handlerMethod);

    void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                           HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```

**New file:** `src/main/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandlerComposite.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class HandlerMethodReturnValueHandlerComposite {

    private final List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>();

    public void addHandler(HandlerMethodReturnValueHandler handler) {
        handlers.add(handler);
    }

    public void addHandlers(List<HandlerMethodReturnValueHandler> newHandlers) {
        handlers.addAll(newHandlers);
    }

    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        HandlerMethodReturnValueHandler handler = selectHandler(handlerMethod);
        if (handler == null) {
            throw new IllegalStateException(
                    "No suitable HandlerMethodReturnValueHandler for return type ["
                            + (returnValue != null ? returnValue.getClass().getName() : "null")
                            + "] on method " + handlerMethod);
        }
        handler.handleReturnValue(returnValue, handlerMethod, request, response);
    }

    private HandlerMethodReturnValueHandler selectHandler(HandlerMethod handlerMethod) {
        for (HandlerMethodReturnValueHandler handler : handlers) {
            if (handler.supportsReturnType(handlerMethod)) {
                return handler;
            }
        }
        return null;
    }

    public List<HandlerMethodReturnValueHandler> getHandlers() {
        return Collections.unmodifiableList(handlers);
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

public class ResponseBodyReturnValueHandler implements HandlerMethodReturnValueHandler {

    private final List<HttpMessageConverter> messageConverters;

    public ResponseBodyReturnValueHandler(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters;
    }

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        if (handlerMethod.getMethod().isAnnotationPresent(ResponseBody.class)) {
            return true;
        }
        return handlerMethod.getBeanType().isAnnotationPresent(ResponseBody.class);
    }

    @Override
    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue == null) {
            response.setStatus(HttpServletResponse.SC_OK);
            return;
        }

        Class<?> valueType = returnValue.getClass();
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(valueType)) {
                converter.write(returnValue, response);
                return;
            }
        }

        throw new IllegalStateException(
                "No suitable HttpMessageConverter found for return value of type ["
                        + valueType.getName() + "]");
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/adapter/StringReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public class StringReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return true;  // Catch-all — sits last in the chain
    }

    @Override
    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        if (returnValue != null) {
            response.getWriter().write(returnValue.toString());
        }
        response.getWriter().flush();
    }
}
```

## 6.5 Try It Yourself

<details>
<summary>Challenge: Given the following controller, predict which response format each method produces</summary>

```java
@Controller
public class QuizController {

    @RequestMapping(path = "/a", method = "GET")
    public String methodA() {
        return "hello";
    }

    @ResponseBody
    @RequestMapping(path = "/b", method = "GET")
    public String methodB() {
        return "hello";
    }

    @ResponseBody
    @RequestMapping(path = "/c", method = "GET")
    public Map<String, String> methodC() {
        return Map.of("msg", "hello");
    }
}
```

**Answers:**
- `/a` → `text/plain`, body: `hello` (no `@ResponseBody`, falls through to `StringReturnValueHandler`)
- `/b` → `application/json`, body: `"hello"` (has `@ResponseBody`, Jackson serializes the String as a JSON string with quotes)
- `/c` → `application/json`, body: `{"msg":"hello"}` (has `@ResponseBody`, Jackson serializes the Map)

Note the subtle difference: method B returns the Java String `hello`, but because `@ResponseBody` triggers Jackson, the output is `"hello"` (with JSON quotes), not bare `hello`.

</details>

<details>
<summary>Challenge: Add an XML message converter alongside Jackson</summary>

Think about: What would you need to change if you wanted `@ResponseBody` to sometimes produce XML instead of JSON?

```java
public class XmlMessageConverter implements HttpMessageConverter {

    @Override
    public boolean canWrite(Class<?> clazz) {
        // Check if the class has JAXB annotations
        return clazz.isAnnotationPresent(jakarta.xml.bind.annotation.XmlRootElement.class);
    }

    @Override
    public void write(Object value, HttpServletResponse response) throws Exception {
        response.setContentType("application/xml");
        response.setCharacterEncoding("UTF-8");
        jakarta.xml.bind.JAXBContext context = jakarta.xml.bind.JAXBContext.newInstance(value.getClass());
        context.createMarshaller().marshal(value, response.getWriter());
    }
}
```

With `canWrite()` filtering by annotation, the converter chain naturally selects the right format. This is exactly how content negotiation begins — ch14 adds `Accept` header matching to make the selection automatic.

</details>

## 6.6 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/converter/JacksonMessageConverterTest.java`

```java
@Test
void shouldSerializeObjectToJson() throws Exception {
    Map<String, Object> data = Map.of("name", "Alice", "age", 30);
    converter.write(data, response);
    String json = responseBody.toString();
    assertThat(json).contains("\"name\"").contains("\"Alice\"");
}

@Test
void shouldSetContentType() throws Exception {
    converter.write("test", response);
    verify(response).setContentType("application/json");
}

@Test
void shouldSerializePojoToJson() throws Exception {
    converter.write(new UserDto("Bob", 25), response);
    assertThat(responseBody.toString()).contains("\"name\":\"Bob\"").contains("\"age\":25");
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandlerTest.java`

```java
@Test
void shouldSupportMethodAnnotated() throws Exception {
    // Method with @ResponseBody → handler should match
    assertThat(handler.supportsReturnType(handlerMethod)).isTrue();
}

@Test
void shouldSupportClassAnnotated() throws Exception {
    // Class with @ResponseBody → all methods should match
    assertThat(handler.supportsReturnType(handlerMethod)).isTrue();
}

@Test
void shouldNotSupportWithoutAnnotation() throws Exception {
    // Plain method without @ResponseBody → handler should not match
    assertThat(handler.supportsReturnType(handlerMethod)).isFalse();
}

@Test
void shouldSerializeToJson() throws Exception {
    handler.handleReturnValue(Map.of("name", "Alice"), handlerMethod, request, response);
    verify(response).setContentType("application/json");
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandlerCompositeTest.java`

```java
@Test
void shouldDelegateToResponseBodyHandler() throws Exception {
    // @ResponseBody method → should get application/json
    composite.handleReturnValue(Map.of("k", "v"), handlerMethod, request, response);
    verify(response).setContentType("application/json");
}

@Test
void shouldDelegateToStringHandler() throws Exception {
    // Plain method → should get text/plain
    composite.handleReturnValue("hello", handlerMethod, request, response);
    verify(response).setContentType("text/plain");
}

@Test
void shouldUseFirstMatchOrdering() throws Exception {
    // @ResponseBody registered FIRST → JSON wins over text/plain
    verify(response).setContentType("application/json");
    verify(response, never()).setContentType("text/plain");
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/ResponseBodyIntegrationTest.java`

Tests the full stack through embedded Tomcat:

```java
@Test
void shouldReturnJsonForMap() throws Exception {
    // GET /json/map → @ResponseBody method returning Map
    // Expects: Content-Type: application/json, body: {"name":"Alice","age":30}
}

@Test
void shouldReturnTextPlain() throws Exception {
    // GET /text → plain method without @ResponseBody
    // Expects: Content-Type: text/plain, body: "hello world"
}

@Test
void shouldReturnJsonForClassAnnotation() throws Exception {
    // GET /api/status → @ResponseBody on class, not on method
    // Expects: Content-Type: application/json
}

@Test
void shouldSerializePojoToJson() throws Exception {
    // GET /api/user → returns UserDto POJO
    // Expects: {"name":"Bob","age":25}
}
```

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 6.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why Strategy + Composite instead of if/else?** The `writeResponse()` method we replaced had one code path: `text/plain`. Adding JSON would mean `if (hasResponseBody) { writeJson() } else { writeText() }`. But views (ch13), SSE (ch22), and `ResponseEntity` would each add another branch. The Strategy pattern makes each format self-contained, and the Composite makes them pluggable. The real framework registers **15+ return value handlers** in `getDefaultReturnValueHandlers()` — imagine that as an if/else chain.
> - **When NOT to use this pattern:** If you're building a small app with exactly one response format, the indirection of converters and handlers adds complexity without benefit. The pattern pays off when multiple formats coexist or when you need extensibility.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why separate HttpMessageConverter from ReturnValueHandler?** The converter handles serialization (`Object → bytes`). The handler handles the decision (`should I serialize?`). This separation means Jackson doesn't know about `@ResponseBody`, and `@ResponseBody` detection doesn't know about JSON. When ch08 adds `@RequestBody` deserialization, it reuses the same `JacksonMessageConverter` — only the handler is different. This is the Single Responsibility Principle in action.
> - **Real-world impact:** Spring ships 7+ message converters (JSON, XML, Protobuf, RSS, etc.) and 15+ return value handlers. They compose freely because neither knows about the other.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why does registration order determine priority?** The composite iterates handlers linearly and returns the first match. This is intentional — it lets the framework define a clear precedence: `ResponseEntity` > `@ResponseBody` > view name > model attribute. In the real framework, `getDefaultReturnValueHandlers()` carefully orders 15+ handlers. Custom handlers are inserted between annotation-based and catch-all handlers, giving users the ability to override specific behaviors without breaking the fallback.
> -----------------------------------------------------------

## 6.8 What We Enhanced

| Aspect | Before (ch04) | Current (ch06) | Real Framework |
|--------|---------------|----------------|----------------|
| Response writing | Hardcoded `writeResponse()` — always `text/plain` + `toString()` | Pluggable `HandlerMethodReturnValueHandler` chain — annotation-driven format selection | 15+ return value handlers in priority order (`RequestMappingHandlerAdapter.java:730`) |
| Serialization | None — raw `toString()` | `HttpMessageConverter` strategy with Jackson JSON implementation | `HttpMessageConverter<T>` with 7+ implementations including JSON, XML, Protobuf (`HttpMessageConverter.java:57`) |
| @ResponseBody | Not supported | Method-level and class-level detection → JSON via Jackson | Same, plus meta-annotation support via `AnnotatedElementUtils` and class-level caching (`RequestResponseBodyMethodProcessor.java:139`) |

## 6.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `HttpMessageConverter` | `HttpMessageConverter<T>` | `HttpMessageConverter.java:57` | Real is generic, has `canRead()`/`read()`, takes `MediaType` parameter |
| `JacksonMessageConverter` | `MappingJackson2HttpMessageConverter` | `MappingJackson2HttpMessageConverter.java:51` | Real extends `AbstractJackson2HttpMessageConverter`, uses `Jackson2ObjectMapperBuilder`, supports `application/*+json` |
| `HandlerMethodReturnValueHandler` | `HandlerMethodReturnValueHandler` | `HandlerMethodReturnValueHandler.java:32` | Real takes `MethodParameter` (index -1 for return type) + `ModelAndViewContainer` + `NativeWebRequest` |
| `HandlerMethodReturnValueHandlerComposite` | `HandlerMethodReturnValueHandlerComposite` | `HandlerMethodReturnValueHandlerComposite.java:71` | Real handles async return values via `AsyncHandlerMethodReturnValueHandler` |
| `ResponseBodyReturnValueHandler` | `RequestResponseBodyMethodProcessor` | `RequestResponseBodyMethodProcessor.java:139` | Real also handles `@RequestBody` (argument resolution), performs content negotiation, caches class-level checks |
| `StringReturnValueHandler` | `ViewNameMethodReturnValueHandler` | `ViewNameMethodReturnValueHandler.java` | Real resolves String returns as view names, not as raw text output |
| `SimpleHandlerAdapter` constructor (handler registration) | `getDefaultReturnValueHandlers()` | `RequestMappingHandlerAdapter.java:730` | Real registers 15+ handlers with careful priority ordering |
| `JacksonMessageConverter.write()` | `AbstractJackson2HttpMessageConverter.writeInternal()` | `AbstractJackson2HttpMessageConverter.java:442` | Real handles streaming, encoding selection, JsonGenerator config, view filtering |

## 6.10 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/ResponseBody.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that the return value of a handler method should be written directly
 * to the HTTP response body, rather than being interpreted as a view name.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.ResponseBody}
 *
 * When present on a method, the return value is serialized to the response body
 * using an {@code HttpMessageConverter} (e.g., JSON via Jackson). When present
 * on a class, it applies to all handler methods in that class — this is the
 * foundation for {@code @RestController} (ch10).
 *
 * The real annotation is equally simple — a pure marker with no attributes.
 * The behavior is entirely in {@code RequestResponseBodyMethodProcessor}, which
 * detects it and delegates to the converter chain.
 *
 * Simplifications: None — this matches the real annotation exactly.
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ResponseBody {
}
```

#### File: `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java` [NEW]

```java
package com.simplespringmvc.converter;

import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for converting between Java objects and HTTP response bodies.
 *
 * Maps to: {@code org.springframework.http.converter.HttpMessageConverter<T>}
 *
 * The real interface is generic ({@code <T>}) and supports both reading (deserialization)
 * and writing (serialization) with full media type negotiation:
 * <pre>
 *   boolean canRead(Class<?> clazz, MediaType mediaType)
 *   T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
 *   boolean canWrite(Class<?> clazz, MediaType mediaType)
 *   void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
 *   List&lt;MediaType&gt; getSupportedMediaTypes()
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Write-only — no {@code canRead()}/{@code read()} yet (ch08 adds @RequestBody deserialization)</li>
 *   <li>No generic type parameter — all converters handle {@code Object}</li>
 *   <li>No MediaType parameter — writes to its default content type (ch14 adds content negotiation)</li>
 *   <li>Takes {@code HttpServletResponse} directly instead of {@code HttpOutputMessage}</li>
 * </ul>
 */
public interface HttpMessageConverter {

    /**
     * Whether this converter can serialize the given class.
     *
     * Maps to: {@code HttpMessageConverter.canWrite(Class<?>, MediaType)} (line 92)
     * Real version also checks media type compatibility.
     *
     * @param clazz the class to check
     * @return true if this converter can serialize instances of the class
     */
    boolean canWrite(Class<?> clazz);

    /**
     * Serialize the given object to the HTTP response.
     *
     * Maps to: {@code HttpMessageConverter.write(T, MediaType, HttpOutputMessage)} (line 104)
     * Real version writes to an HttpOutputMessage abstraction with headers.
     * We write directly to HttpServletResponse for simplicity.
     *
     * @param value    the object to serialize
     * @param response the HTTP response to write to
     * @throws Exception if serialization fails
     */
    void write(Object value, HttpServletResponse response) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletResponse;

/**
 * An {@link HttpMessageConverter} that serializes Java objects to JSON
 * using Jackson's {@link ObjectMapper}.
 *
 * Maps to: {@code org.springframework.http.converter.json.MappingJackson2HttpMessageConverter}
 *
 * The real converter:
 * <ul>
 *   <li>Extends {@code AbstractJackson2HttpMessageConverter} which extends
 *       {@code AbstractGenericHttpMessageConverter}</li>
 *   <li>Uses {@code Jackson2ObjectMapperBuilder} for opinionated default configuration</li>
 *   <li>Supports both {@code application/json} and {@code application/*+json}</li>
 *   <li>Handles ProblemDetail responses with special media type selection</li>
 *   <li>Supports JSON prefix for XSS protection (anti-hijacking)</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No builder — uses a plain {@code new ObjectMapper()}</li>
 *   <li>Always writes {@code application/json} — no media type negotiation</li>
 *   <li>No read support (ch08)</li>
 *   <li>No JSON prefix / anti-hijacking</li>
 * </ul>
 */
public class JacksonMessageConverter implements HttpMessageConverter {

    private final ObjectMapper objectMapper;

    public JacksonMessageConverter() {
        this.objectMapper = new ObjectMapper();
    }

    public JacksonMessageConverter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    /**
     * Jackson can write most Java objects to JSON.
     *
     * Maps to: {@code AbstractJackson2HttpMessageConverter.canWrite(Class, MediaType)} (line 328)
     * Real version also checks media type compatibility and calls
     * {@code objectMapper.canSerialize(clazz)}.
     *
     * We return true for all types — Jackson handles most objects, and for those
     * it can't serialize, it will throw at write time with a clear error.
     */
    @Override
    public boolean canWrite(Class<?> clazz) {
        return true;
    }

    /**
     * Serialize the given object to JSON and write it to the response.
     *
     * Maps to: {@code AbstractJackson2HttpMessageConverter.writeInternal()} (line 432)
     * Real version handles streaming, media type-based encoding selection,
     * JsonGenerator configuration, and Jackson view filtering.
     */
    @Override
    public void write(Object value, HttpServletResponse response) throws Exception {
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        objectMapper.writeValue(response.getWriter(), value);
    }

    public ObjectMapper getObjectMapper() {
        return objectMapper;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandler.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for processing the return value of a handler method
 * and writing the appropriate HTTP response.
 *
 * Maps to: {@code org.springframework.web.method.support.HandlerMethodReturnValueHandler}
 *
 * The real interface takes a {@code MethodParameter returnType} (with index -1
 * representing the return type) and a {@code ModelAndViewContainer} for view/model
 * state. We simplify by passing the {@link HandlerMethod} directly — it gives us
 * access to both the method (for annotation checks) and the bean type (for
 * class-level annotation checks like {@code @ResponseBody} on the class).
 *
 * Implementations:
 * <ul>
 *   <li>{@code ResponseBodyReturnValueHandler} — serializes to JSON when @ResponseBody is present</li>
 *   <li>{@code StringReturnValueHandler} — writes plain text (the default fallback)</li>
 *   <li>Future: {@code ViewNameReturnValueHandler} — resolves view names (ch13)</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Takes {@code HandlerMethod} instead of {@code MethodParameter} — avoids the "index -1" return type concept</li>
 *   <li>Takes {@code HttpServletRequest}/{@code HttpServletResponse} instead of {@code NativeWebRequest}</li>
 *   <li>No {@code ModelAndViewContainer} — no view/model support yet (ch13)</li>
 * </ul>
 */
public interface HandlerMethodReturnValueHandler {

    /**
     * Whether this handler supports the given handler method's return type.
     *
     * Maps to: {@code HandlerMethodReturnValueHandler.supportsReturnType(MethodParameter)}
     *
     * @param handlerMethod the handler method whose return value needs processing
     * @return true if this handler can process the return value
     */
    boolean supportsReturnType(HandlerMethod handlerMethod);

    /**
     * Process the return value by writing to the HTTP response.
     *
     * Maps to: {@code HandlerMethodReturnValueHandler.handleReturnValue(Object, MethodParameter, ModelAndViewContainer, NativeWebRequest)}
     *
     * @param returnValue   the value returned by the handler method (may be null)
     * @param handlerMethod the handler method that produced the return value
     * @param request       the current HTTP request
     * @param response      the current HTTP response
     * @throws Exception if processing fails
     */
    void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                           HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandlerComposite.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Composite that iterates a list of {@link HandlerMethodReturnValueHandler}s
 * and delegates to the first one that supports the return type.
 *
 * Maps to: {@code org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite}
 *
 * The real version:
 * <ul>
 *   <li>Implements {@code HandlerMethodReturnValueHandler} itself (composite pattern)</li>
 *   <li>Has special handling for async return values ({@code AsyncHandlerMethodReturnValueHandler})</li>
 *   <li>Iterates handlers in registration order — first match wins</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No async return value handling</li>
 *   <li>Does not implement the handler interface itself — used directly by the adapter</li>
 * </ul>
 */
public class HandlerMethodReturnValueHandlerComposite {

    private final List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>();

    /**
     * Add a return value handler to the end of the list.
     */
    public void addHandler(HandlerMethodReturnValueHandler handler) {
        handlers.add(handler);
    }

    /**
     * Add multiple return value handlers.
     */
    public void addHandlers(List<HandlerMethodReturnValueHandler> newHandlers) {
        handlers.addAll(newHandlers);
    }

    /**
     * Find the first handler that supports the return type and delegate to it.
     *
     * Maps to: {@code HandlerMethodReturnValueHandlerComposite.handleReturnValue()} (line 71)
     * → {@code selectHandler()} (line 81)
     *
     * The real version throws {@code IllegalArgumentException} if no handler matches.
     * We do the same — registration order determines priority.
     */
    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        HandlerMethodReturnValueHandler handler = selectHandler(handlerMethod);
        if (handler == null) {
            throw new IllegalStateException(
                    "No suitable HandlerMethodReturnValueHandler for return type ["
                            + (returnValue != null ? returnValue.getClass().getName() : "null")
                            + "] on method " + handlerMethod);
        }
        handler.handleReturnValue(returnValue, handlerMethod, request, response);
    }

    /**
     * Find the first handler that supports this return type.
     *
     * Maps to: {@code HandlerMethodReturnValueHandlerComposite.selectHandler()} (line 81)
     * Real version also checks for async return values and skips non-async handlers.
     */
    private HandlerMethodReturnValueHandler selectHandler(HandlerMethod handlerMethod) {
        for (HandlerMethodReturnValueHandler handler : handlers) {
            if (handler.supportsReturnType(handlerMethod)) {
                return handler;
            }
        }
        return null;
    }

    /**
     * Returns an unmodifiable view of the registered handlers.
     */
    public List<HandlerMethodReturnValueHandler> getHandlers() {
        return Collections.unmodifiableList(handlers);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

/**
 * Handles return values from handler methods annotated with {@link ResponseBody}
 * by serializing the return value through an {@link HttpMessageConverter}.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor}
 *
 * The real version:
 * <ul>
 *   <li>Implements both {@code HandlerMethodArgumentResolver} (for @RequestBody) and
 *       {@code HandlerMethodReturnValueHandler} (for @ResponseBody)</li>
 *   <li>Extends {@code AbstractMessageConverterMethodProcessor} which handles
 *       content negotiation and converter selection</li>
 *   <li>Caches class-level @ResponseBody checks in a ConcurrentHashMap</li>
 *   <li>Uses AnnotatedElementUtils for meta-annotation detection (for @RestController)</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Return value handling only — no @RequestBody argument resolution (ch08)</li>
 *   <li>No content negotiation — uses the first compatible converter (ch14)</li>
 *   <li>Direct annotation check — no meta-annotation walking (ch10 adds @RestController)</li>
 *   <li>No class-level caching (not needed at this scale)</li>
 * </ul>
 */
public class ResponseBodyReturnValueHandler implements HandlerMethodReturnValueHandler {

    private final List<HttpMessageConverter> messageConverters;

    public ResponseBodyReturnValueHandler(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters;
    }

    /**
     * Supports methods or classes annotated with {@link ResponseBody}.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.supportsReturnType()} (line 139)
     * Real version:
     * <pre>
     *   return (AnnotatedElementUtils.hasAnnotation(containingClass, ResponseBody.class)
     *           || returnType.hasMethodAnnotation(ResponseBody.class));
     * </pre>
     *
     * We check both method-level and class-level @ResponseBody directly.
     * Class-level @ResponseBody is the foundation for @RestController (ch10).
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        // Check method-level @ResponseBody
        if (handlerMethod.getMethod().isAnnotationPresent(ResponseBody.class)) {
            return true;
        }
        // Check class-level @ResponseBody (foundation for @RestController in ch10)
        return handlerMethod.getBeanType().isAnnotationPresent(ResponseBody.class);
    }

    /**
     * Serialize the return value using the first compatible message converter.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.handleReturnValue()} (line 195)
     * → {@code AbstractMessageConverterMethodProcessor.writeWithMessageConverters()} (line 262)
     *
     * The real version:
     * 1. Sets mavContainer.setRequestHandled(true) — signals no view resolution needed
     * 2. Performs content negotiation to determine the media type
     * 3. Iterates converters and picks the first matching one
     * 4. Calls converter.write() with the negotiated media type
     *
     * We skip step 1 (no ModelAndViewContainer) and step 2 (no content negotiation),
     * but follow the same converter iteration pattern.
     */
    @Override
    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue == null) {
            // Nothing to serialize — the real framework also returns early for null
            response.setStatus(HttpServletResponse.SC_OK);
            return;
        }

        Class<?> valueType = returnValue.getClass();
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(valueType)) {
                converter.write(returnValue, response);
                return;
            }
        }

        throw new IllegalStateException(
                "No suitable HttpMessageConverter found for return value of type ["
                        + valueType.getName() + "]");
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/StringReturnValueHandler.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Handles return values by writing them as plain text — the default fallback
 * when no other return value handler matches.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler}
 *
 * In the real framework, a String return value without @ResponseBody is treated
 * as a logical view name and resolved via ViewResolver to render HTML. We don't
 * have view resolution yet (ch13), so this handler writes the String directly
 * as text/plain — preserving the behavior from ch04.
 *
 * This handler is registered AFTER {@code ResponseBodyReturnValueHandler} in the
 * composite, so @ResponseBody methods are handled by the JSON handler. Only
 * methods WITHOUT @ResponseBody fall through to this handler.
 *
 * Simplifications:
 * <ul>
 *   <li>Writes text/plain instead of resolving a view name (ch13 changes this)</li>
 *   <li>Handles all return types, not just String (acts as a catch-all)</li>
 * </ul>
 */
public class StringReturnValueHandler implements HandlerMethodReturnValueHandler {

    /**
     * This handler is the default fallback — it supports all return types.
     * It sits at the end of the handler chain and catches anything not handled
     * by more specific handlers (like ResponseBodyReturnValueHandler).
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return true;
    }

    /**
     * Write the return value as text/plain to the response.
     * This is the same behavior as the previous ch04 writeResponse() method,
     * now extracted into a proper return value handler.
     */
    @Override
    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        if (returnValue != null) {
            response.getWriter().write(returnValue.toString());
        }
        response.getWriter().flush();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;

/**
 * The default HandlerAdapter that knows how to invoke {@link HandlerMethod}
 * instances via reflection, resolving method arguments from the request first
 * and processing the return value through the return value handler chain.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 * which extends {@code AbstractHandlerMethodAdapter}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handleInternal() creates a ServletInvocableHandlerMethod
 *   3. Sets argument resolvers on the invocable method
 *   4. Sets return value handlers on the invocable method
 *   5. Calls invocableMethod.invokeAndHandle()
 *   6. invokeAndHandle() resolves arguments, invokes via reflection,
 *      handles the return value
 * </pre>
 *
 * We collapse that chain:
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handle() resolves arguments via the resolver chain
 *   3. Invokes the method via reflection with the resolved arguments
 *   4. Delegates return value to the return value handler chain
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No InvocableHandlerMethod wrapper — we call Method.invoke() directly</li>
 *   <li>No ModelAndView return — response is written directly</li>
 *   <li>No WebDataBinderFactory</li>
 *   <li>No ModelFactory</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();
    private final List<HttpMessageConverter> messageConverters = new ArrayList<>();

    public SimpleHandlerAdapter() {
        // Register the default argument resolvers.
        // Maps to: RequestMappingHandlerAdapter.getDefaultArgumentResolvers() (line 644)
        // Real version registers ~30 resolvers in a specific order. We start with one.
        argumentResolvers.addResolver(new RequestParamArgumentResolver());

        // Register the default message converters.
        // Maps to: RequestMappingHandlerAdapter.getMessageConverters()
        // Real version registers ~7 converters. We start with Jackson.
        messageConverters.add(new JacksonMessageConverter());

        // Register the default return value handlers.
        // Maps to: RequestMappingHandlerAdapter.getDefaultReturnValueHandlers() (line 730)
        // ORDER MATTERS: @ResponseBody handler must come BEFORE the String fallback,
        // because the composite uses first-match. This is the same priority scheme
        // the real framework uses — annotation-based handlers before catch-all handlers.
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    /**
     * Create an adapter with custom resolvers added BEFORE the defaults.
     * This mirrors how the real framework's setCustomArgumentResolvers() works.
     */
    public SimpleHandlerAdapter(List<HandlerMethodArgumentResolver> customResolvers) {
        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        argumentResolvers.addResolver(new RequestParamArgumentResolver());
        messageConverters.add(new JacksonMessageConverter());
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    /**
     * This adapter supports only HandlerMethod instances — the type produced
     * by our SimpleHandlerMapping when scanning @RequestMapping methods.
     *
     * Maps to: {@code AbstractHandlerMethodAdapter.supports()} (line 44)
     * which checks {@code handler instanceof HandlerMethod}
     */
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    /**
     * Resolve arguments, invoke the handler method, and write the result to the response.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.handleInternal()} (line 831)
     * → {@code invokeHandlerMethod()} (line 885)
     * → {@code ServletInvocableHandlerMethod.invokeAndHandle()} (line 114)
     * → {@code InvocableHandlerMethod.invokeForRequest()} (line 171)
     *
     * The real chain creates an InvocableHandlerMethod, configures argument
     * resolvers and return value handlers, then invokes. We do a simplified
     * version that still follows the same resolve → invoke → handle-return flow.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the HandlerMethod to invoke
     */
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // Step 1: Resolve method arguments from the request (ch05).
        // Maps to: InvocableHandlerMethod.getMethodArgumentValues() (line 186)
        Object[] args = resolveArguments(handlerMethod, request);

        // Step 2: Invoke the controller method via reflection.
        // Maps to: InvocableHandlerMethod.doInvoke() (line 243)
        Object result = invokeHandlerMethod(handlerMethod, args);

        // Step 3: Process the return value through the handler chain.
        // Maps to: ServletInvocableHandlerMethod.invokeAndHandle() (line 114)
        // → HandlerMethodReturnValueHandlerComposite.handleReturnValue()
        // The composite iterates handlers: @ResponseBody → JSON, else → text/plain.
        returnValueHandlers.handleReturnValue(result, handlerMethod, request, response);
    }

    /**
     * Resolve each method parameter into an argument value using the resolver chain.
     *
     * Maps to: {@code InvocableHandlerMethod.getMethodArgumentValues()} (line 186)
     *
     * The real version:
     * <pre>
     *   for each parameter:
     *     if resolvers.supportsParameter(param):
     *       args[i] = resolvers.resolveArgument(param, mavContainer, webRequest, binderFactory)
     *     else:
     *       throw IllegalStateException("no suitable resolver")
     * </pre>
     *
     * We follow the same pattern but with our simplified resolver interface.
     * Parameters without a matching resolver get null — this allows no-arg
     * methods and methods with a mix of resolved and unresolved params to work.
     * Future chapters will tighten this as more resolvers are added.
     */
    private Object[] resolveArguments(HandlerMethod handlerMethod, HttpServletRequest request) throws Exception {
        MethodParameter[] parameters = handlerMethod.getMethodParameters();
        if (parameters.length == 0) {
            return new Object[0];
        }

        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter param = parameters[i];
            if (argumentResolvers.supportsParameter(param)) {
                args[i] = argumentResolvers.resolveArgument(param, request);
            } else {
                throw new IllegalStateException(
                        "No suitable resolver for argument " + i + " of type '"
                                + param.getParameterType().getName()
                                + "' on method " + handlerMethod);
            }
        }
        return args;
    }

    /**
     * Invoke the handler method with the resolved arguments and return the result.
     *
     * Maps to: {@code InvocableHandlerMethod.doInvoke()} (line 243)
     * The real version uses getBridgedMethod() for generic type bridge methods,
     * handles Kotlin suspend functions, and formats detailed error messages.
     */
    private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
        } catch (InvocationTargetException ex) {
            // Unwrap the real exception from the reflection wrapper.
            // Maps to: InvocableHandlerMethod.doInvoke() (line 260-275)
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    /**
     * Provides access to the argument resolver composite for testing.
     */
    public HandlerMethodArgumentResolverComposite getArgumentResolvers() {
        return argumentResolvers;
    }

    /**
     * Provides access to the return value handler composite for testing.
     */
    public HandlerMethodReturnValueHandlerComposite getReturnValueHandlers() {
        return returnValueHandlers;
    }

    /**
     * Provides access to the message converters for testing.
     */
    public List<HttpMessageConverter> getMessageConverters() {
        return messageConverters;
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/converter/JacksonMessageConverterTest.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class JacksonMessageConverterTest {

    private JacksonMessageConverter converter;
    private HttpServletResponse response;
    private StringWriter responseBody;

    @BeforeEach
    void setUp() throws Exception {
        converter = new JacksonMessageConverter();
        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    @Nested
    @DisplayName("canWrite()")
    class CanWrite {
        @Test
        @DisplayName("should support any class")
        void shouldSupportAnyClass() {
            assertThat(converter.canWrite(String.class)).isTrue();
            assertThat(converter.canWrite(Integer.class)).isTrue();
            assertThat(converter.canWrite(Map.class)).isTrue();
            assertThat(converter.canWrite(Object.class)).isTrue();
        }
    }

    @Nested
    @DisplayName("write()")
    class Write {
        @Test
        @DisplayName("should serialize object to JSON")
        void shouldSerializeObjectToJson() throws Exception {
            Map<String, Object> data = Map.of("name", "Alice", "age", 30);
            converter.write(data, response);
            String json = responseBody.toString();
            assertThat(json).contains("\"name\"").contains("\"Alice\"").contains("\"age\"").contains("30");
        }

        @Test
        @DisplayName("should set content type to application/json")
        void shouldSetContentType() throws Exception {
            converter.write("test", response);
            verify(response).setContentType("application/json");
        }

        @Test
        @DisplayName("should set character encoding to UTF-8")
        void shouldSetCharacterEncoding() throws Exception {
            converter.write("test", response);
            verify(response).setCharacterEncoding("UTF-8");
        }

        @Test
        @DisplayName("should serialize a list to JSON array")
        void shouldSerializeListToJsonArray() throws Exception {
            List<String> items = List.of("a", "b", "c");
            converter.write(items, response);
            assertThat(responseBody.toString()).isEqualTo("[\"a\",\"b\",\"c\"]");
        }

        @Test
        @DisplayName("should serialize a string as JSON string")
        void shouldSerializeStringAsJsonString() throws Exception {
            converter.write("hello", response);
            assertThat(responseBody.toString()).isEqualTo("\"hello\"");
        }

        @Test
        @DisplayName("should serialize a POJO to JSON")
        void shouldSerializePojoToJson() throws Exception {
            converter.write(new UserDto("Bob", 25), response);
            String json = responseBody.toString();
            assertThat(json).contains("\"name\":\"Bob\"").contains("\"age\":25");
        }
    }

    @Nested
    @DisplayName("custom ObjectMapper")
    class CustomObjectMapper {
        @Test
        @DisplayName("should use provided ObjectMapper")
        void shouldUseProvidedObjectMapper() {
            ObjectMapper custom = new ObjectMapper();
            JacksonMessageConverter customConverter = new JacksonMessageConverter(custom);
            assertThat(customConverter.getObjectMapper()).isSameAs(custom);
        }
    }

    static class UserDto {
        private String name;
        private int age;
        public UserDto() {}
        public UserDto(String name, int age) { this.name = name; this.age = age; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandlerTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ResponseBodyReturnValueHandlerTest {

    static class PlainController {
        public String hello() { return "hello"; }
    }

    static class MethodAnnotatedController {
        @ResponseBody
        public Map<String, String> getData() { return Map.of("key", "value"); }
        public String noAnnotation() { return "plain"; }
    }

    @ResponseBody
    static class ClassAnnotatedController {
        public Map<String, String> getData() { return Map.of("key", "value"); }
    }

    private ResponseBodyReturnValueHandler handler;
    private HttpServletRequest request;
    private HttpServletResponse response;
    private StringWriter responseBody;

    @BeforeEach
    void setUp() throws Exception {
        List<HttpMessageConverter> converters = List.of(new JacksonMessageConverter());
        handler = new ResponseBodyReturnValueHandler(converters);
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    @Nested
    @DisplayName("supportsReturnType()")
    class SupportsReturnType {
        @Test
        @DisplayName("should support method annotated with @ResponseBody")
        void shouldSupportMethodAnnotated() throws Exception {
            MethodAnnotatedController controller = new MethodAnnotatedController();
            Method method = controller.getClass().getMethod("getData");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            assertThat(handler.supportsReturnType(handlerMethod)).isTrue();
        }

        @Test
        @DisplayName("should support class annotated with @ResponseBody")
        void shouldSupportClassAnnotated() throws Exception {
            ClassAnnotatedController controller = new ClassAnnotatedController();
            Method method = controller.getClass().getMethod("getData");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            assertThat(handler.supportsReturnType(handlerMethod)).isTrue();
        }

        @Test
        @DisplayName("should not support method without @ResponseBody")
        void shouldNotSupportWithoutAnnotation() throws Exception {
            MethodAnnotatedController controller = new MethodAnnotatedController();
            Method method = controller.getClass().getMethod("noAnnotation");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            assertThat(handler.supportsReturnType(handlerMethod)).isFalse();
        }

        @Test
        @DisplayName("should not support plain controller without any @ResponseBody")
        void shouldNotSupportPlainController() throws Exception {
            PlainController controller = new PlainController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            assertThat(handler.supportsReturnType(handlerMethod)).isFalse();
        }
    }

    @Nested
    @DisplayName("handleReturnValue()")
    class HandleReturnValue {
        @Test
        @DisplayName("should serialize return value to JSON")
        void shouldSerializeToJson() throws Exception {
            MethodAnnotatedController controller = new MethodAnnotatedController();
            Method method = controller.getClass().getMethod("getData");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            handler.handleReturnValue(Map.of("name", "Alice"), handlerMethod, request, response);
            String json = responseBody.toString();
            assertThat(json).contains("\"name\"").contains("\"Alice\"");
            verify(response).setContentType("application/json");
        }

        @Test
        @DisplayName("should handle null return value without error")
        void shouldHandleNullReturnValue() throws Exception {
            MethodAnnotatedController controller = new MethodAnnotatedController();
            Method method = controller.getClass().getMethod("getData");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            handler.handleReturnValue(null, handlerMethod, request, response);
            assertThat(responseBody.toString()).isEmpty();
            verify(response).setStatus(HttpServletResponse.SC_OK);
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandlerCompositeTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

class HandlerMethodReturnValueHandlerCompositeTest {

    static class PlainController {
        public String hello() { return "hello"; }
    }

    static class JsonController {
        @ResponseBody
        public Map<String, String> getData() { return Map.of("key", "value"); }
    }

    private HandlerMethodReturnValueHandlerComposite composite;
    private HttpServletRequest request;
    private HttpServletResponse response;
    private StringWriter responseBody;

    @BeforeEach
    void setUp() throws Exception {
        composite = new HandlerMethodReturnValueHandlerComposite();
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    @Nested
    @DisplayName("handler selection")
    class HandlerSelection {
        @Test
        @DisplayName("should delegate to @ResponseBody handler for annotated methods")
        void shouldDelegateToResponseBodyHandler() throws Exception {
            composite.addHandler(new ResponseBodyReturnValueHandler(List.of(new JacksonMessageConverter())));
            composite.addHandler(new StringReturnValueHandler());
            JsonController controller = new JsonController();
            Method method = controller.getClass().getMethod("getData");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            composite.handleReturnValue(Map.of("k", "v"), handlerMethod, request, response);
            verify(response).setContentType("application/json");
        }

        @Test
        @DisplayName("should delegate to String handler for unannotated methods")
        void shouldDelegateToStringHandler() throws Exception {
            composite.addHandler(new ResponseBodyReturnValueHandler(List.of(new JacksonMessageConverter())));
            composite.addHandler(new StringReturnValueHandler());
            PlainController controller = new PlainController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            composite.handleReturnValue("hello", handlerMethod, request, response);
            verify(response).setContentType("text/plain");
            assertThat(responseBody.toString()).isEqualTo("hello");
        }

        @Test
        @DisplayName("should throw when no handler supports the return type")
        void shouldThrow_WhenNoHandlerMatches() throws Exception {
            PlainController controller = new PlainController();
            Method method = controller.getClass().getMethod("hello");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            assertThatThrownBy(() -> composite.handleReturnValue("hello", handlerMethod, request, response))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No suitable HandlerMethodReturnValueHandler");
        }
    }

    @Nested
    @DisplayName("handler management")
    class HandlerManagement {
        @Test
        @DisplayName("should return unmodifiable list of handlers")
        void shouldReturnUnmodifiableHandlerList() {
            composite.addHandler(new StringReturnValueHandler());
            assertThatThrownBy(() -> composite.getHandlers().add(new StringReturnValueHandler()))
                    .isInstanceOf(UnsupportedOperationException.class);
        }

        @Test
        @DisplayName("should support adding multiple handlers at once")
        void shouldAddMultipleHandlers() {
            composite.addHandlers(List.of(
                    new ResponseBodyReturnValueHandler(List.of(new JacksonMessageConverter())),
                    new StringReturnValueHandler()));
            assertThat(composite.getHandlers()).hasSize(2);
        }

        @Test
        @DisplayName("should use first-match ordering — @ResponseBody before String")
        void shouldUseFirstMatchOrdering() throws Exception {
            composite.addHandler(new ResponseBodyReturnValueHandler(List.of(new JacksonMessageConverter())));
            composite.addHandler(new StringReturnValueHandler());
            JsonController controller = new JsonController();
            Method method = controller.getClass().getMethod("getData");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);
            composite.handleReturnValue(Map.of("a", "b"), handlerMethod, request, response);
            verify(response).setContentType("application/json");
            verify(response, never()).setContentType("text/plain");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/ResponseBodyIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.*;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ResponseBodyIntegrationTest {

    @Controller
    static class MixedController {
        @RequestMapping(path = "/text", method = "GET")
        public String plainText() { return "hello world"; }

        @ResponseBody
        @RequestMapping(path = "/json/map", method = "GET")
        public Map<String, Object> jsonMap() { return Map.of("name", "Alice", "age", 30); }

        @ResponseBody
        @RequestMapping(path = "/json/list", method = "GET")
        public List<String> jsonList() { return List.of("spring", "mvc", "json"); }

        @ResponseBody
        @RequestMapping(path = "/json/greeting", method = "GET")
        public Map<String, String> jsonGreeting(@RequestParam("name") String name) {
            return Map.of("message", "Hello, " + name + "!");
        }

        @ResponseBody
        @RequestMapping(path = "/json/null", method = "GET")
        public Object jsonNull() { return null; }
    }

    @Controller
    @ResponseBody
    static class ApiController {
        @RequestMapping(path = "/api/status", method = "GET")
        public Map<String, String> status() { return Map.of("status", "ok"); }

        @RequestMapping(path = "/api/user", method = "GET")
        public UserDto user() { return new UserDto("Bob", 25); }
    }

    static class UserDto {
        private String name;
        private int age;
        public UserDto() {}
        public UserDto(String name, int age) { this.name = name; this.age = age; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient client;
    private String baseUrl;
    private final ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new MixedController());
        container.registerBean(new ApiController());
        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();
        baseUrl = "http://localhost:" + tomcat.getPort();
        client = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        if (tomcat != null) tomcat.stop();
    }

    @Nested
    @DisplayName("@ResponseBody on method")
    class MethodLevelResponseBody {
        @Test
        @DisplayName("should return JSON for @ResponseBody method returning a Map")
        void shouldReturnJsonForMap() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/json/map")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("application/json");
            @SuppressWarnings("unchecked")
            Map<String, Object> body = objectMapper.readValue(response.body(), LinkedHashMap.class);
            assertThat(body).containsEntry("name", "Alice");
            assertThat(body).containsEntry("age", 30);
        }

        @Test
        @DisplayName("should return JSON for @ResponseBody method returning a List")
        void shouldReturnJsonForList() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/json/list")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("application/json");
            assertThat(response.body()).isEqualTo("[\"spring\",\"mvc\",\"json\"]");
        }

        @Test
        @DisplayName("should combine @ResponseBody with @RequestParam")
        void shouldCombineWithRequestParam() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/json/greeting?name=Spring")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
            @SuppressWarnings("unchecked")
            Map<String, Object> body = objectMapper.readValue(response.body(), LinkedHashMap.class);
            assertThat(body).containsEntry("message", "Hello, Spring!");
        }

        @Test
        @DisplayName("should handle null return from @ResponseBody method")
        void shouldHandleNullReturn() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/json/null")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
        }
    }

    @Nested
    @DisplayName("plain methods (no @ResponseBody)")
    class PlainMethods {
        @Test
        @DisplayName("should return text/plain for methods without @ResponseBody")
        void shouldReturnTextPlain() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/text")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("text/plain");
            assertThat(response.body()).isEqualTo("hello world");
        }
    }

    @Nested
    @DisplayName("@ResponseBody on class")
    class ClassLevelResponseBody {
        @Test
        @DisplayName("should return JSON when @ResponseBody is on the class")
        void shouldReturnJsonForClassAnnotation() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/api/status")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("application/json");
            @SuppressWarnings("unchecked")
            Map<String, Object> body = objectMapper.readValue(response.body(), LinkedHashMap.class);
            assertThat(body).containsEntry("status", "ok");
        }

        @Test
        @DisplayName("should serialize POJO to JSON for class-level @ResponseBody")
        void shouldSerializePojoToJson() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/api/user")).GET().build(),
                    HttpResponse.BodyHandlers.ofString());
            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("application/json");
            UserDto user = objectMapper.readValue(response.body(), UserDto.class);
            assertThat(user.getName()).isEqualTo("Bob");
            assertThat(user.getAge()).isEqualTo(25);
        }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **HttpMessageConverter** | Strategy interface for serializing objects to a specific wire format (JSON, XML, etc.) |
| **HandlerMethodReturnValueHandler** | Strategy interface that decides *how* to write a handler method's return value based on annotations and type |
| **Composite pattern** | A list of handlers tried in order — first match wins, enabling pluggable response formats |
| **@ResponseBody** | Marker annotation that triggers JSON serialization instead of the default text/plain |
| **Registration order** | Priority mechanism — `@ResponseBody` handler registered before catch-all ensures annotation-based methods get JSON |

**Next: Chapter 7 — Path Variables & Pattern Matching** — Support URL templates like `/users/{id}` with `@PathVariable`, upgrading handler mapping from exact string comparison to pattern matching with variable extraction.
