# Chapter 8: @RequestBody Deserialization

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Controllers can receive data from query strings (`@RequestParam`) and URL paths (`@PathVariable`), and can serialize return values to JSON (`@ResponseBody`). `HttpMessageConverter` is write-only. | Controllers cannot accept JSON request bodies — POST/PUT endpoints that need structured input have no way to deserialize incoming JSON into Java objects. | Extend `HttpMessageConverter` with read support and create a `RequestBodyArgumentResolver` that deserializes `@RequestBody`-annotated parameters from the request input stream. |

---

## 8.1 The Integration Point

The integration point is `SimpleHandlerAdapter`'s constructor — the exact place where argument resolvers are registered. Adding a `RequestBodyArgumentResolver` to the resolver chain connects the existing argument resolution pipeline to the message converter infrastructure.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Register `RequestBodyArgumentResolver` in the default resolver chain, and reorder initialization so message converters are created before resolvers (since the new resolver depends on them).

```java
public SimpleHandlerAdapter() {
    // Register the default message converters FIRST — resolvers and handlers depend on them.
    messageConverters.add(new JacksonMessageConverter());

    // Register the default argument resolvers.
    // @RequestBody comes after @RequestParam — same as real framework line 663.
    argumentResolvers.addResolver(new PathVariableArgumentResolver());
    argumentResolvers.addResolver(new RequestParamArgumentResolver());
    argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));  // NEW

    // Return value handlers unchanged
    returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
    returnValueHandlers.addHandler(new StringReturnValueHandler());
}
```

Two key decisions here:

1. **Converters initialized before resolvers.** In ch06, message converters were initialized between resolvers and return value handlers. Now that `RequestBodyArgumentResolver` *also* depends on the converter list, converters must be initialized first. This mirrors the real `RequestMappingHandlerAdapter` where converters are a shared dependency.

2. **Same converter list for read and write.** Both `RequestBodyArgumentResolver` and `ResponseBodyReturnValueHandler` receive the *same* `messageConverters` list. This is the real framework's design — one `JacksonMessageConverter` handles both directions, and the framework uses `canRead()`/`canWrite()` to determine which direction a converter supports.

This connects the **argument resolution pipeline** to the **message converter infrastructure**. To make it work, we need to build:
- `@RequestBody` annotation — marks parameters for body deserialization
- `canRead()` + `read()` on `HttpMessageConverter` — the read side of the converter contract
- `JacksonMessageConverter.read()` — Jackson-based JSON deserialization
- `RequestBodyArgumentResolver` — the resolver that bridges `@RequestBody` to converters

## 8.2 The @RequestBody Annotation

**New file:** `src/main/java/com/simplespringmvc/annotation/RequestBody.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestBody {
}
```

Unlike `@ResponseBody` (which targets methods *and* types), `@RequestBody` only targets parameters — it makes no sense on a return type or class.

## 8.3 Extending HttpMessageConverter with Read Support

**Modifying:** `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java`
**Change:** Add `canRead()` and `read()` methods to the interface, mirroring the existing `canWrite()` and `write()`.

```java
boolean canRead(Class<?> clazz);

Object read(Class<?> clazz, HttpServletRequest request) throws Exception;
```

The interface now has four methods — the complete read/write symmetry:

| Direction | Guard | Action |
|-----------|-------|--------|
| **Read** (request → object) | `canRead(Class<?>)` | `read(Class<?>, HttpServletRequest)` |
| **Write** (object → response) | `canWrite(Class<?>)` | `write(Object, HttpServletResponse)` |

## 8.4 Adding Read Support to JacksonMessageConverter

**Modifying:** `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java`
**Change:** Implement `canRead()` and `read()` methods.

```java
@Override
public boolean canRead(Class<?> clazz) {
    return true;
}

@Override
public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
    return objectMapper.readValue(request.getInputStream(), clazz);
}
```

The `read()` implementation is a single line — Jackson's `ObjectMapper.readValue()` handles the entire deserialization pipeline: reading bytes from the input stream, parsing JSON tokens, and mapping them to the target type's fields via reflection (matching JSON keys to setter methods or fields).

## 8.5 The RequestBodyArgumentResolver

**New file:** `src/main/java/com/simplespringmvc/adapter/RequestBodyArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

import java.util.List;

public class RequestBodyArgumentResolver implements HandlerMethodArgumentResolver {

    private final List<HttpMessageConverter> messageConverters;

    public RequestBodyArgumentResolver(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestBody.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();

        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canRead(targetType)) {
                return converter.read(targetType, request);
            }
        }

        throw new IllegalStateException(
                "No HttpMessageConverter found that can read request body into type '"
                        + targetType.getName() + "' for parameter '" + parameter.getParameterName()
                        + "'. Real framework would throw HttpMediaTypeNotSupportedException (415).");
    }
}
```

The resolver follows the same first-match pattern as `ResponseBodyReturnValueHandler` from ch06 — iterate converters, find the first one that says "I can handle this", and delegate to it. With only `JacksonMessageConverter` registered, all types match. When ch14 adds content negotiation, the `canRead()` check will also consider the `Content-Type` header.

## 8.6 Try It Yourself

<details>
<summary>Challenge: Add read support to HttpMessageConverter and JacksonMessageConverter</summary>

Starting from the ch07 codebase where `HttpMessageConverter` only has `canWrite()` and `write()`:

1. Add `canRead(Class<?>)` and `read(Class<?>, HttpServletRequest)` to `HttpMessageConverter`
2. Implement both in `JacksonMessageConverter` using `objectMapper.readValue()`
3. Verify with a test that sends `{"name":"Alice","age":30}` and deserializes to a POJO

The solution is in:
- `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java`
- `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java`

</details>

<details>
<summary>Challenge: Wire RequestBodyArgumentResolver into the adapter</summary>

Given the `RequestBodyArgumentResolver` class, update `SimpleHandlerAdapter` to:

1. Initialize message converters **before** argument resolvers
2. Register `RequestBodyArgumentResolver` with the shared converter list
3. Ensure the custom-resolvers constructor also follows this pattern

Hint: The resolver needs the `messageConverters` list passed to its constructor — the same list that `ResponseBodyReturnValueHandler` already uses.

The solution is in `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`.

</details>

## 8.7 Tests

### Unit Tests

**Modified file:** `src/test/java/com/simplespringmvc/converter/JacksonMessageConverterTest.java`

New test sections added for `canRead()` and `read()`:

```java
@Test
@DisplayName("should deserialize JSON into a POJO")
void shouldDeserializeJsonIntoPojo() throws Exception {
    when(request.getInputStream()).thenReturn(
            createJsonInputStream("{\"name\":\"Alice\",\"age\":30}"));

    Object result = converter.read(UserDto.class, request);

    assertThat(result).isInstanceOf(UserDto.class);
    UserDto user = (UserDto) result;
    assertThat(user.getName()).isEqualTo("Alice");
    assertThat(user.getAge()).isEqualTo(30);
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/RequestBodyArgumentResolverTest.java`

Key test cases:
- `shouldSupport_WhenAnnotatedWithRequestBody` — verifies annotation detection
- `shouldNotSupport_WhenAnnotatedWithRequestParam` — verifies exclusivity
- `shouldDeserializeJsonBody_WhenValidJson` — core happy path
- `shouldDeserializePartialJson_WhenSomeFieldsMissing` — missing fields get defaults
- `shouldThrow_WhenJsonIsMalformed` — error propagation
- `shouldThrow_WhenNoConverterCanRead` — no-match scenario

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/RequestBodyIntegrationTest.java`

Tests the full stack with a real Tomcat server:
- POST with JSON body → deserialized and echoed back as JSON response
- PUT with JSON body → same flow, different HTTP method
- `@RequestBody` + `@RequestParam` combined in the same method

**Run:** `./gradlew test` — expected: all 100+ tests pass (including all prior features' tests)

---

## 8.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Symmetric interface design.** `HttpMessageConverter` now has a read/write symmetry: `canRead`/`read` mirrors `canWrite`/`write`. This isn't just aesthetic — it means a single converter class (like `JacksonMessageConverter`) can handle both directions, and both `RequestBodyArgumentResolver` and `ResponseBodyReturnValueHandler` share the same converter instance list. In the real framework, `RequestResponseBodyMethodProcessor` takes this further by being a single class that implements *both* `HandlerMethodArgumentResolver` and `HandlerMethodReturnValueHandler`, using the same converters for both directions. We keep them separate here for clarity, but the shared converter list achieves the same effect.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The resolver is thin; the converter does the work.** Notice how `RequestBodyArgumentResolver.resolveArgument()` is essentially just three lines: get the target type, find a converter, call `read()`. All the actual deserialization complexity lives in `JacksonMessageConverter.read()` → `ObjectMapper.readValue()`. This separation means adding XML support later would require zero changes to the resolver — just register an `XmlMessageConverter` that handles `application/xml`. The Strategy pattern here isn't about making the code "extensible for the sake of extensibility" — it's about ensuring each converter encapsulates a complete serialization format without the resolver needing to know format-specific details.
> -----------------------------------------------------------

## 8.9 What We Enhanced

| Aspect | Before (ch06–ch07) | Current (ch08) | Real Framework |
|--------|---------------------|----------------|----------------|
| `HttpMessageConverter` | Write-only: `canWrite()` + `write()` | Full read/write: adds `canRead()` + `read()` | Generic `<T>` with `MediaType` params on both directions (`HttpMessageConverter.java:58-104`) |
| `JacksonMessageConverter` | Only serialized objects to JSON responses | Also deserializes JSON request bodies to POJOs | 4-level hierarchy: `MappingJackson2HttpMessageConverter` → `AbstractJackson2HttpMessageConverter` → `AbstractGenericHttpMessageConverter` → `AbstractHttpMessageConverter` |
| `SimpleHandlerAdapter` init order | Converters initialized between resolvers and handlers | Converters initialized first (resolvers now also depend on them) | `RequestMappingHandlerAdapter.afterPropertiesSet()` initializes converters, then builds resolver and handler lists (`RequestMappingHandlerAdapter.java:344`) |
| Request body handling | No support — POST/PUT bodies ignored | `@RequestBody` parameters deserialized via converter chain | `RequestResponseBodyMethodProcessor` with `Content-Type` checking, empty body detection, `RequestBodyAdvice` hooks, and `@Valid` integration |

## 8.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@RequestBody` (parameter-only, no attributes) | `@RequestBody` | `RequestBody.java:48` | Real has `required` attribute (default `true`) — when `false`, missing body passes `null` instead of throwing |
| `HttpMessageConverter.canRead(Class<?>)` | `HttpMessageConverter.canRead(Class<?>, MediaType)` | `HttpMessageConverter.java:58` | Real also checks media type compatibility (`Content-Type` header) |
| `HttpMessageConverter.read(Class<?>, HttpServletRequest)` | `HttpMessageConverter.read(Class<? extends T>, HttpInputMessage)` | `HttpMessageConverter.java:73` | Real uses `HttpInputMessage` abstraction with headers; generic `<T>` for type safety |
| `JacksonMessageConverter.read()` — single line | `AbstractJackson2HttpMessageConverter.readInternal()` | `AbstractJackson2HttpMessageConverter.java:311` | Real handles encoding detection, `JsonParser` config, Jackson view filtering, `@JsonView` support |
| `RequestBodyArgumentResolver` (argument resolver only) | `RequestResponseBodyMethodProcessor` | `RequestResponseBodyMethodProcessor.java:134-189` | Real implements BOTH `ArgumentResolver` and `ReturnValueHandler`; delegates read to `AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()` which wraps body in `EmptyBodyCheckingHttpInputMessage`, runs `RequestBodyAdvice` before/after hooks, and triggers validation |
| Converter iteration: first `canRead()` match | Three-tier converter check: `GenericHttpMessageConverter` → `SmartHttpMessageConverter` → base `HttpMessageConverter` | `AbstractMessageConverterMethodArgumentResolver.java:178-214` | Real supports generic types via `GenericHttpMessageConverter.canRead(Type, Class, MediaType)` and `ResolvableType` via `SmartHttpMessageConverter` |

## 8.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/RequestBody.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation indicating a method parameter should be bound to the body of
 * the HTTP request. The body is deserialized to the parameter type using an
 * {@link com.simplespringmvc.converter.HttpMessageConverter}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.RequestBody}
 *
 * The real annotation:
 * <ul>
 *   <li>Has a {@code required} attribute (default true) — when true, a missing
 *       body causes a 400 error; when false, null is passed</li>
 *   <li>Triggers validation when used with {@code @Valid} (ch16)</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code required} attribute — body is always required</li>
 *   <li>No validation integration (ch16)</li>
 * </ul>
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestBody {
}
```

#### File: `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java` [MODIFIED]

```java
package com.simplespringmvc.converter;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for converting between Java objects and HTTP messages.
 * Supports both reading (deserialization from request body) and writing
 * (serialization to response body).
 *
 * Maps to: {@code org.springframework.http.converter.HttpMessageConverter<T>}
 *
 * The real interface is generic ({@code <T>}) and supports full media type negotiation:
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
 *   <li>No generic type parameter — all converters handle {@code Object}</li>
 *   <li>No MediaType parameter — reads/writes its default content type (ch14 adds content negotiation)</li>
 *   <li>Takes {@code HttpServletRequest}/{@code HttpServletResponse} directly instead of
 *       {@code HttpInputMessage}/{@code HttpOutputMessage}</li>
 * </ul>
 */
public interface HttpMessageConverter {

    /**
     * Whether this converter can deserialize the given class from a request body.
     *
     * Maps to: {@code HttpMessageConverter.canRead(Class<?>, MediaType)} (line 58)
     * Real version also checks media type compatibility.
     *
     * @param clazz the target class to check
     * @return true if this converter can deserialize into instances of the class
     */
    boolean canRead(Class<?> clazz);

    /**
     * Deserialize an object of the given class from the HTTP request body.
     *
     * Maps to: {@code HttpMessageConverter.read(Class, HttpInputMessage)} (line 73)
     * Real version reads from an HttpInputMessage abstraction.
     * We read directly from HttpServletRequest for simplicity.
     *
     * The caller must verify {@link #canRead(Class)} before calling this method.
     *
     * @param clazz   the target class to deserialize into
     * @param request the HTTP request to read the body from
     * @return the deserialized object
     * @throws Exception if deserialization fails
     */
    Object read(Class<?> clazz, HttpServletRequest request) throws Exception;

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

#### File: `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java` [MODIFIED]

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * An {@link HttpMessageConverter} that converts between Java objects and JSON
 * using Jackson's {@link ObjectMapper}. Supports both reading (deserialization
 * from request body) and writing (serialization to response body).
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
 *   <li>Read side uses {@code EmptyBodyCheckingHttpInputMessage} to detect empty bodies</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No builder — uses a plain {@code new ObjectMapper()}</li>
 *   <li>Always reads/writes {@code application/json} — no media type negotiation</li>
 *   <li>No empty body detection — relies on Jackson to throw on empty input</li>
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
     * Jackson can deserialize JSON into most Java types.
     *
     * Maps to: {@code AbstractJackson2HttpMessageConverter.canRead(Class, MediaType)} (line 253)
     * Real version also checks media type (must be application/json or application/*+json)
     * and calls {@code objectMapper.canDeserialize(javaType)}.
     *
     * We return true for all types — Jackson handles most objects, and for those
     * it can't deserialize, it will throw at read time with a clear error.
     */
    @Override
    public boolean canRead(Class<?> clazz) {
        return true;
    }

    /**
     * Deserialize JSON from the request body into an object of the given type.
     *
     * Maps to: {@code AbstractJackson2HttpMessageConverter.readInternal()} (line 311)
     * Real version handles encoding detection from Content-Type charset,
     * JsonParser configuration, and Jackson view filtering.
     *
     * @param clazz   the target type to deserialize into
     * @param request the HTTP request containing the JSON body
     * @return the deserialized object
     * @throws Exception if the body is missing, malformed, or cannot be mapped to the target type
     */
    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        return objectMapper.readValue(request.getInputStream(), clazz);
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

#### File: `src/main/java/com/simplespringmvc/adapter/RequestBodyArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

import java.util.List;

/**
 * Resolves method arguments annotated with {@link RequestBody} by deserializing
 * the HTTP request body using an {@link HttpMessageConverter}.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor}
 * (the argument-resolution side only)
 *
 * <h3>How the real framework does it:</h3>
 * The real {@code RequestResponseBodyMethodProcessor} implements BOTH
 * {@code HandlerMethodArgumentResolver} and {@code HandlerMethodReturnValueHandler}.
 * It delegates to {@code AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()}
 * which:
 * <ol>
 *   <li>Parses the Content-Type header to determine the media type</li>
 *   <li>Wraps the input in {@code EmptyBodyCheckingHttpInputMessage} to detect empty bodies</li>
 *   <li>Iterates converters checking {@code canRead(targetClass, contentType)}</li>
 *   <li>Calls {@code RequestBodyAdvice.beforeBodyRead()} for interceptors</li>
 *   <li>Calls {@code converter.read(targetClass, inputMessage)}</li>
 *   <li>Calls {@code RequestBodyAdvice.afterBodyRead()}</li>
 *   <li>Optionally validates with {@code @Valid} (ch16)</li>
 * </ol>
 *
 * We simplify to: iterate converters, find one that {@code canRead()}, call {@code read()}.
 *
 * Simplifications:
 * <ul>
 *   <li>No Content-Type/MediaType checking — assumes JSON (ch14 adds content negotiation)</li>
 *   <li>No empty body detection — Jackson will throw on empty input</li>
 *   <li>No RequestBodyAdvice interceptors</li>
 *   <li>No validation integration (ch16)</li>
 *   <li>No {@code required=false} support — body is always required</li>
 *   <li>Separate class from the return-value handler (real framework combines both)</li>
 * </ul>
 */
public class RequestBodyArgumentResolver implements HandlerMethodArgumentResolver {

    private final List<HttpMessageConverter> messageConverters;

    public RequestBodyArgumentResolver(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters;
    }

    /**
     * Supports parameters annotated with {@code @RequestBody}.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.supportsParameter()} (line 134)
     * which simply checks {@code parameter.hasParameterAnnotation(RequestBody.class)}.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestBody.class);
    }

    /**
     * Deserialize the request body into the parameter's target type.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.resolveArgument()} (line 152)
     * → {@code AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()} (line 147)
     *
     * The real version:
     * <pre>
     *   1. Parse Content-Type header
     *   2. Check for empty body
     *   3. For each converter:
     *        if converter.canRead(targetClass, contentType):
     *          body = advice.beforeBodyRead(...)
     *          body = converter.read(targetClass, inputMessage)
     *          body = advice.afterBodyRead(...)
     *          break
     *   4. If no converter matched: throw HttpMediaTypeNotSupportedException (415)
     *   5. If body is null and required=true: throw HttpMessageNotReadableException
     *   6. Validate if @Valid present
     * </pre>
     *
     * We follow the same converter-iteration pattern but skip media type checking,
     * advice interceptors, and validation.
     *
     * @throws IllegalStateException if no converter can read the target type
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();

        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canRead(targetType)) {
                return converter.read(targetType, request);
            }
        }

        throw new IllegalStateException(
                "No HttpMessageConverter found that can read request body into type '"
                        + targetType.getName() + "' for parameter '" + parameter.getParameterName()
                        + "'. Real framework would throw HttpMediaTypeNotSupportedException (415).");
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
 *   <li>No WebDataBinderFactory — ch15 adds data binding</li>
 *   <li>No ModelFactory — ch13 adds model support</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();
    private final List<HttpMessageConverter> messageConverters = new ArrayList<>();

    public SimpleHandlerAdapter() {
        // Register the default message converters FIRST — resolvers and handlers depend on them.
        // Maps to: RequestMappingHandlerAdapter.getMessageConverters()
        // Real version registers ~7 converters. We start with Jackson.
        messageConverters.add(new JacksonMessageConverter());

        // Register the default argument resolvers.
        // Maps to: RequestMappingHandlerAdapter.getDefaultArgumentResolvers() (line 644)
        // Real version registers ~30 resolvers in a specific order.
        // ORDER MATTERS: resolvers are checked in registration order. @PathVariable
        // comes before @RequestParam to match the real framework's ordering.
        // @RequestBody comes after @RequestParam — same as real framework line 663.
        argumentResolvers.addResolver(new PathVariableArgumentResolver());
        argumentResolvers.addResolver(new RequestParamArgumentResolver());
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));

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
        messageConverters.add(new JacksonMessageConverter());
        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        argumentResolvers.addResolver(new PathVariableArgumentResolver());
        argumentResolvers.addResolver(new RequestParamArgumentResolver());
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
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

#### File: `src/test/java/com/simplespringmvc/adapter/RequestBodyArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.ReadListener;
import jakarta.servlet.ServletInputStream;
import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.lang.reflect.Method;
import java.nio.charset.StandardCharsets;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

/**
 * Unit tests for {@link RequestBodyArgumentResolver}.
 */
class RequestBodyArgumentResolverTest {

    // ─── Test fixtures ─────────────────────────────────────────────

    static class CreateUserRequest {
        private String name;
        private int age;

        public CreateUserRequest() {}

        public CreateUserRequest(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    @SuppressWarnings("unused")
    static class TestController {

        public String withRequestBody(@RequestBody CreateUserRequest user) {
            return user.getName();
        }

        public String withRequestParam(@RequestParam String name) {
            return name;
        }

        public String noAnnotation(String value) {
            return value;
        }
    }

    private RequestBodyArgumentResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        List<HttpMessageConverter> converters = List.of(new JacksonMessageConverter());
        resolver = new RequestBodyArgumentResolver(converters);
        request = mock(HttpServletRequest.class);
    }

    /**
     * Creates a mock ServletInputStream wrapping a JSON string.
     */
    private ServletInputStream createJsonInputStream(String json) {
        ByteArrayInputStream byteStream = new ByteArrayInputStream(json.getBytes(StandardCharsets.UTF_8));
        return new ServletInputStream() {
            @Override public int read() { return byteStream.read(); }
            @Override public boolean isFinished() { return byteStream.available() == 0; }
            @Override public boolean isReady() { return true; }
            @Override public void setReadListener(ReadListener readListener) {}
        };
    }

    // ─── supportsParameter() ──────────────────────────────────────

    @Nested
    @DisplayName("supportsParameter()")
    class SupportsParameter {

        @Test
        @DisplayName("should support parameter with @RequestBody")
        void shouldSupport_WhenAnnotatedWithRequestBody() throws Exception {
            Method method = TestController.class.getMethod("withRequestBody", CreateUserRequest.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(resolver.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should not support parameter with @RequestParam")
        void shouldNotSupport_WhenAnnotatedWithRequestParam() throws Exception {
            Method method = TestController.class.getMethod("withRequestParam", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(resolver.supportsParameter(param)).isFalse();
        }

        @Test
        @DisplayName("should not support parameter without annotations")
        void shouldNotSupport_WhenNoAnnotation() throws Exception {
            Method method = TestController.class.getMethod("noAnnotation", String.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThat(resolver.supportsParameter(param)).isFalse();
        }
    }

    // ─── resolveArgument() ────────────────────────────────────────

    @Nested
    @DisplayName("resolveArgument()")
    class ResolveArgument {

        @Test
        @DisplayName("should deserialize JSON body into target POJO")
        void shouldDeserializeJsonBody_WhenValidJson() throws Exception {
            Method method = TestController.class.getMethod("withRequestBody", CreateUserRequest.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getInputStream()).thenReturn(
                    createJsonInputStream("{\"name\":\"Alice\",\"age\":30}"));

            Object result = resolver.resolveArgument(param, request);

            assertThat(result).isInstanceOf(CreateUserRequest.class);
            CreateUserRequest user = (CreateUserRequest) result;
            assertThat(user.getName()).isEqualTo("Alice");
            assertThat(user.getAge()).isEqualTo(30);
        }

        @Test
        @DisplayName("should deserialize JSON with partial fields")
        void shouldDeserializePartialJson_WhenSomeFieldsMissing() throws Exception {
            Method method = TestController.class.getMethod("withRequestBody", CreateUserRequest.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getInputStream()).thenReturn(
                    createJsonInputStream("{\"name\":\"Bob\"}"));

            Object result = resolver.resolveArgument(param, request);

            CreateUserRequest user = (CreateUserRequest) result;
            assertThat(user.getName()).isEqualTo("Bob");
            assertThat(user.getAge()).isEqualTo(0); // default int value
        }

        @Test
        @DisplayName("should throw when JSON is malformed")
        void shouldThrow_WhenJsonIsMalformed() throws Exception {
            Method method = TestController.class.getMethod("withRequestBody", CreateUserRequest.class);
            MethodParameter param = new MethodParameter(method, 0);
            when(request.getInputStream()).thenReturn(
                    createJsonInputStream("{invalid json}"));

            assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                    .isInstanceOf(Exception.class);
        }
    }

    // ─── no converter match ─────────────────────────────────────

    @Nested
    @DisplayName("no converter match")
    class NoConverterMatch {

        @Test
        @DisplayName("should throw when no converter can read the target type")
        void shouldThrow_WhenNoConverterCanRead() throws Exception {
            // Create a resolver with a converter that can't read anything
            HttpMessageConverter noReadConverter = new HttpMessageConverter() {
                @Override public boolean canRead(Class<?> clazz) { return false; }
                @Override public Object read(Class<?> clazz, HttpServletRequest req) { return null; }
                @Override public boolean canWrite(Class<?> clazz) { return true; }
                @Override public void write(Object value, jakarta.servlet.http.HttpServletResponse resp) {}
            };
            RequestBodyArgumentResolver noMatchResolver =
                    new RequestBodyArgumentResolver(List.of(noReadConverter));

            Method method = TestController.class.getMethod("withRequestBody", CreateUserRequest.class);
            MethodParameter param = new MethodParameter(method, 0);

            assertThatThrownBy(() -> noMatchResolver.resolveArgument(param, request))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No HttpMessageConverter found");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/converter/JacksonMessageConverterTest.java` [MODIFIED]

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.ReadListener;
import jakarta.servlet.ServletInputStream;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.io.PrintWriter;
import java.io.StringWriter;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link JacksonMessageConverter}.
 */
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

    // ─── canWrite() ─────────────────────────────────────────────

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

    // ─── write() ────────────────────────────────────────────────

    @Nested
    @DisplayName("write()")
    class Write {

        @Test
        @DisplayName("should serialize object to JSON")
        void shouldSerializeObjectToJson() throws Exception {
            Map<String, Object> data = Map.of("name", "Alice", "age", 30);

            converter.write(data, response);

            String json = responseBody.toString();
            assertThat(json).contains("\"name\"");
            assertThat(json).contains("\"Alice\"");
            assertThat(json).contains("\"age\"");
            assertThat(json).contains("30");
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
            assertThat(json).contains("\"name\":\"Bob\"");
            assertThat(json).contains("\"age\":25");
        }
    }

    // ─── canRead() ──────────────────────────────────────────────

    @Nested
    @DisplayName("canRead()")
    class CanRead {

        @Test
        @DisplayName("should support any class")
        void shouldSupportAnyClass() {
            assertThat(converter.canRead(String.class)).isTrue();
            assertThat(converter.canRead(Integer.class)).isTrue();
            assertThat(converter.canRead(Map.class)).isTrue();
            assertThat(converter.canRead(UserDto.class)).isTrue();
        }
    }

    // ─── read() ─────────────────────────────────────────────────

    @Nested
    @DisplayName("read()")
    class ReadBody {

        private HttpServletRequest request;

        @BeforeEach
        void setUpRequest() {
            request = mock(HttpServletRequest.class);
        }

        private ServletInputStream createJsonInputStream(String json) {
            ByteArrayInputStream byteStream = new ByteArrayInputStream(json.getBytes(StandardCharsets.UTF_8));
            return new ServletInputStream() {
                @Override public int read() { return byteStream.read(); }
                @Override public boolean isFinished() { return byteStream.available() == 0; }
                @Override public boolean isReady() { return true; }
                @Override public void setReadListener(ReadListener readListener) {}
            };
        }

        @Test
        @DisplayName("should deserialize JSON into a POJO")
        void shouldDeserializeJsonIntoPojo() throws Exception {
            when(request.getInputStream()).thenReturn(
                    createJsonInputStream("{\"name\":\"Alice\",\"age\":30}"));

            Object result = converter.read(UserDto.class, request);

            assertThat(result).isInstanceOf(UserDto.class);
            UserDto user = (UserDto) result;
            assertThat(user.getName()).isEqualTo("Alice");
            assertThat(user.getAge()).isEqualTo(30);
        }

        @Test
        @DisplayName("should deserialize JSON into a Map")
        void shouldDeserializeJsonIntoMap() throws Exception {
            when(request.getInputStream()).thenReturn(
                    createJsonInputStream("{\"key\":\"value\"}"));

            Object result = converter.read(Map.class, request);

            assertThat(result).isInstanceOf(Map.class);
            @SuppressWarnings("unchecked")
            Map<String, Object> map = (Map<String, Object>) result;
            assertThat(map).containsEntry("key", "value");
        }

        @Test
        @DisplayName("should throw on malformed JSON")
        void shouldThrow_WhenJsonIsMalformed() throws Exception {
            when(request.getInputStream()).thenReturn(
                    createJsonInputStream("{not valid json}"));

            assertThatThrownBy(() -> converter.read(UserDto.class, request))
                    .isInstanceOf(Exception.class);
        }
    }

    // ─── Custom ObjectMapper ────────────────────────────────────

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

    // ─── Test DTO ───────────────────────────────────────────────

    static class UserDto {
        private String name;
        private int age;

        public UserDto() {}

        public UserDto(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/RequestBodyIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestBody;
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
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for @RequestBody deserialization.
 * Starts a real Tomcat server and sends HTTP requests with JSON bodies
 * to verify that @RequestBody parameters are correctly deserialized.
 */
class RequestBodyIntegrationTest {

    // ─── Test DTOs ────────────────────────────────────────────────

    static class CreateUserRequest {
        private String name;
        private int age;

        public CreateUserRequest() {}

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    static class UserResponse {
        private long id;
        private String name;
        private int age;

        public UserResponse() {}

        public UserResponse(long id, String name, int age) {
            this.id = id;
            this.name = name;
            this.age = age;
        }

        public long getId() { return id; }
        public void setId(long id) { this.id = id; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    // ─── Test controller ─────────────────────────────────────────

    @Controller
    @ResponseBody
    static class UserController {

        /**
         * POST with @RequestBody — the core feature under test.
         */
        @RequestMapping(path = "/users", method = "POST")
        public UserResponse createUser(@RequestBody CreateUserRequest request) {
            return new UserResponse(1L, request.getName(), request.getAge());
        }

        /**
         * PUT with @RequestBody — update scenario.
         */
        @RequestMapping(path = "/users/update", method = "PUT")
        public UserResponse updateUser(@RequestBody CreateUserRequest request) {
            return new UserResponse(42L, request.getName(), request.getAge());
        }

        /**
         * @RequestBody combined with @RequestParam — both in the same method.
         */
        @RequestMapping(path = "/users/tagged", method = "POST")
        public Map<String, Object> createTaggedUser(
                @RequestBody CreateUserRequest request,
                @RequestParam("tag") String tag) {
            return Map.of(
                    "name", request.getName(),
                    "age", request.getAge(),
                    "tag", tag
            );
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient client;
    private String baseUrl;
    private final ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new UserController());

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        baseUrl = "http://localhost:" + tomcat.getPort();
        client = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        if (tomcat != null) {
            tomcat.stop();
        }
    }

    // ─── POST with @RequestBody ─────────────────────────────────

    @Nested
    @DisplayName("POST with @RequestBody")
    class PostWithRequestBody {

        @Test
        @DisplayName("should deserialize JSON body and return JSON response")
        void shouldDeserializeJsonBody_WhenPostWithJson() throws Exception {
            String json = "{\"name\":\"Alice\",\"age\":30}";

            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/users"))
                            .POST(HttpRequest.BodyPublishers.ofString(json))
                            .header("Content-Type", "application/json")
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("application/json");

            UserResponse user = objectMapper.readValue(response.body(), UserResponse.class);
            assertThat(user.getId()).isEqualTo(1L);
            assertThat(user.getName()).isEqualTo("Alice");
            assertThat(user.getAge()).isEqualTo(30);
        }

        @Test
        @DisplayName("should handle partial JSON body with missing fields")
        void shouldHandlePartialJson_WhenFieldsMissing() throws Exception {
            String json = "{\"name\":\"Bob\"}";

            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/users"))
                            .POST(HttpRequest.BodyPublishers.ofString(json))
                            .header("Content-Type", "application/json")
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);

            UserResponse user = objectMapper.readValue(response.body(), UserResponse.class);
            assertThat(user.getName()).isEqualTo("Bob");
            assertThat(user.getAge()).isEqualTo(0); // default value
        }
    }

    // ─── PUT with @RequestBody ──────────────────────────────────

    @Nested
    @DisplayName("PUT with @RequestBody")
    class PutWithRequestBody {

        @Test
        @DisplayName("should deserialize JSON body on PUT request")
        void shouldDeserializeJsonBody_WhenPutRequest() throws Exception {
            String json = "{\"name\":\"Charlie\",\"age\":35}";

            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/users/update"))
                            .PUT(HttpRequest.BodyPublishers.ofString(json))
                            .header("Content-Type", "application/json")
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);

            UserResponse user = objectMapper.readValue(response.body(), UserResponse.class);
            assertThat(user.getId()).isEqualTo(42L);
            assertThat(user.getName()).isEqualTo("Charlie");
            assertThat(user.getAge()).isEqualTo(35);
        }
    }

    // ─── @RequestBody + @RequestParam ──────────────────────────

    @Nested
    @DisplayName("@RequestBody combined with @RequestParam")
    class RequestBodyWithRequestParam {

        @Test
        @DisplayName("should resolve both @RequestBody and @RequestParam in same method")
        void shouldResolveBothAnnotations_WhenCombined() throws Exception {
            String json = "{\"name\":\"Diana\",\"age\":28}";

            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/users/tagged?tag=vip"))
                            .POST(HttpRequest.BodyPublishers.ofString(json))
                            .header("Content-Type", "application/json")
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);

            @SuppressWarnings("unchecked")
            Map<String, Object> body = objectMapper.readValue(response.body(), Map.class);
            assertThat(body).containsEntry("name", "Diana");
            assertThat(body).containsEntry("age", 28);
            assertThat(body).containsEntry("tag", "vip");
        }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@RequestBody`** | Marks a method parameter to be deserialized from the HTTP request body |
| **`HttpMessageConverter.canRead()/read()`** | The read side of the converter contract — mirrors `canWrite()/write()` for symmetry |
| **`RequestBodyArgumentResolver`** | Bridges the annotation detection (which parameter?) with the converter selection (how to deserialize?) |
| **Shared converter list** | Both `RequestBodyArgumentResolver` and `ResponseBodyReturnValueHandler` use the same `messageConverters` list — one `JacksonMessageConverter` handles both directions |
| **First-match converter iteration** | Converters are checked in order; the first one whose `canRead()` returns `true` wins — identical pattern to the write side |

**Next: Chapter 9 — Type Conversion** — Currently `@RequestParam` and `@PathVariable` only produce `String` values. Chapter 9 adds a `ConversionService` with pluggable `Converter<S, T>` implementations so that `@RequestParam int page` and `@PathVariable Long id` work automatically.
