# Chapter 14: Content Negotiation

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `@ResponseBody` writes JSON using the first compatible converter | No Accept header checking, no `produces`/`consumes` constraints, only one converter | Select the right converter and media type based on Accept header, with `produces`/`consumes` on `@RequestMapping` |

---

## 14.1 The Integration Point

There are two integration points for content negotiation: one for **response writing** and one for **handler matching**.

### Primary: `ResponseBodyReturnValueHandler.handleReturnValue()`

This is where the content negotiation algorithm replaces the old "first converter wins" logic. Previously, it iterated converters and used the first one that could write the value type. Now it implements the full negotiation pipeline:

**Modifying:** `src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java`

```java
@Override
public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                      HttpServletRequest request, HttpServletResponse response) throws Exception {
    if (returnValue == null) {
        response.setStatus(HttpServletResponse.SC_OK);
        return null;
    }

    Class<?> valueType = returnValue.getClass();

    // Step 1: Get acceptable media types from Accept header
    List<MediaType> acceptableTypes = contentNegotiationManager.resolveMediaTypes(request);

    // Step 2: Get producible media types
    List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType);

    // Step 3: Cross-match to find compatible types
    List<MediaType> compatibleTypes = new ArrayList<>();
    for (MediaType acceptable : acceptableTypes) {
        for (MediaType producible : producibleTypes) {
            if (acceptable.isCompatibleWith(producible)) {
                compatibleTypes.add(getMostSpecific(acceptable, producible));
            }
        }
    }

    if (compatibleTypes.isEmpty()) {
        throw new HttpMediaTypeNotAcceptableException(producibleTypes);
    }

    // Step 4: Sort by specificity and quality
    MediaType.sortBySpecificityAndQuality(compatibleTypes);

    // Step 5: Pick the first concrete type
    MediaType selectedType = null;
    for (MediaType type : compatibleTypes) {
        if (type.isConcrete()) {
            selectedType = type;
            break;
        }
        if (type.isWildcardType() || type.isWildcardSubtype()) {
            selectedType = MediaType.APPLICATION_OCTET_STREAM;
            break;
        }
    }

    if (selectedType == null) {
        throw new HttpMediaTypeNotAcceptableException(producibleTypes);
    }

    selectedType = selectedType.removeQualityValue();

    // Step 6: Find a converter and write
    for (HttpMessageConverter converter : messageConverters) {
        if (converter.canWrite(valueType, selectedType)) {
            converter.write(returnValue, selectedType, response);
            return null;
        }
    }

    throw new HttpMediaTypeNotAcceptableException(producibleTypes);
}
```

The old code was:
```java
// Before (ch13): first converter wins
for (HttpMessageConverter converter : messageConverters) {
    if (converter.canWrite(valueType)) {
        converter.write(returnValue, response);
        return null;
    }
}
```

### Secondary: `SimpleHandlerMapping.lookupHandler(HttpServletRequest)`

This is where `produces` and `consumes` conditions are checked during handler matching. A handler is only matched if the request's Content-Type satisfies `consumes` and the request's Accept header satisfies `produces`.

```java
public HandlerMethod lookupHandler(HttpServletRequest request) {
    // ...
    for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
        RouteKey key = entry.getKey();
        Map<String, String> pathVariables = key.matchAndExtract(requestPath, requestMethod);
        if (pathVariables != null) {
            // ch14: Check consumes condition against request Content-Type
            if (!key.matchesConsumes(request.getContentType())) {
                continue;
            }
            // ch14: Check produces condition against request Accept header
            if (!key.matchesProduces(request.getHeader("Accept"))) {
                continue;
            }
            // Store producible media types for content negotiation
            if (!key.getProduces().isEmpty()) {
                request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE,
                        new ArrayList<>(key.getProduces()));
            }
            return entry.getValue();
        }
    }
    return null;
}
```

Two subsystems connect here: the **Accept header parsing** (`ContentNegotiationManager` + `MediaType`) feeds into the **converter selection pipeline** (`ResponseBodyReturnValueHandler`). The `produces` attribute on `@RequestMapping` constrains both where the request is routed *and* what types content negotiation considers for the response.

To make this work, we need to build:
- `MediaType` -- the data type for media type parsing, matching, and sorting
- `ContentNegotiationManager` -- parses the Accept header into sorted MediaTypes
- `PlainTextMessageConverter` -- a second converter to make negotiation meaningful
- `HttpMediaTypeNotAcceptableException` -- signals 406 Not Acceptable
- `produces`/`consumes` attributes on `@RequestMapping` and composed annotations
- `RouteKey` media type matching
- `SimpleHandlerMapping` produces/consumes checking
- `DispatcherServlet` 406 handling

---

## 14.2 Supporting Components

### 1. MediaType -- the data type everything depends on

`MediaType` represents a MIME type as defined in RFC 2046. It parses strings like `"application/json"` or `"text/html;q=0.9"` and supports matching, compatibility checks, and quality-based sorting. In the real framework, `MimeType` holds the core logic and `MediaType` extends it with quality values and Accept header parsing. We collapse both into one class.

**New file:** `src/main/java/com/simplespringmvc/http/MediaType.java`

Key operations:

```java
// Asymmetric: does this type include the other?
// */* includes everything, text/* includes text/plain, but not vice versa
public boolean includes(MediaType other) {
    if (other == null) return false;
    if (isWildcardType()) return true;
    if (this.type.equals(other.type)) {
        if (isWildcardSubtype()) return true;
        return this.subtype.equals(other.subtype);
    }
    return false;
}

// Symmetric: are the two types compatible?
// Either side can have wildcards
public boolean isCompatibleWith(MediaType other) {
    if (other == null) return false;
    if (isWildcardType() || other.isWildcardType()) return true;
    if (this.type.equals(other.type)) {
        if (isWildcardSubtype() || other.isWildcardSubtype()) return true;
        return this.subtype.equals(other.subtype);
    }
    return false;
}
```

Parsing the comma-separated Accept header format:
```java
// "text/html, application/json;q=0.9, */*;q=0.1" → [text/html, application/json;q=0.9, */*;q=0.1]
public static List<MediaType> parseMediaTypes(String value) {
    if (value == null || value.isBlank()) {
        return new ArrayList<>(List.of(ALL));
    }
    List<MediaType> result = new ArrayList<>();
    for (String token : tokenize(value)) {
        result.add(parseMediaType(token));
    }
    return result;
}
```

Sorting by specificity and quality for content negotiation:
```java
// Higher quality first, then more specific first (concrete > wildcard subtype > wildcard type)
public static void sortBySpecificityAndQuality(List<MediaType> mediaTypes) {
    mediaTypes.sort((a, b) -> {
        int qCompare = Double.compare(b.getQualityValue(), a.getQualityValue());
        if (qCompare != 0) return qCompare;
        int specA = a.isWildcardType() ? 0 : (a.isWildcardSubtype() ? 1 : 2);
        int specB = b.isWildcardType() ? 0 : (b.isWildcardSubtype() ? 1 : 2);
        int specCompare = Integer.compare(specB, specA);
        if (specCompare != 0) return specCompare;
        return Integer.compare(b.getParameters().size(), a.getParameters().size());
    });
}
```

### 2. ContentNegotiationManager -- parses Accept header

The real framework's `ContentNegotiationManager` holds a `List<ContentNegotiationStrategy>` (header, path extension, query parameter). We simplify to just the Accept header strategy, which covers the overwhelming majority of real-world content negotiation.

**New file:** `src/main/java/com/simplespringmvc/http/ContentNegotiationManager.java`

```java
public class ContentNegotiationManager {

    public List<MediaType> resolveMediaTypes(HttpServletRequest request) {
        String acceptHeader = request.getHeader("Accept");

        if (acceptHeader == null || acceptHeader.isBlank()) {
            return List.of(MediaType.ALL);  // client accepts anything
        }

        List<MediaType> mediaTypes = MediaType.parseMediaTypes(acceptHeader);
        if (mediaTypes.isEmpty()) {
            return List.of(MediaType.ALL);
        }

        MediaType.sortBySpecificityAndQuality(mediaTypes);
        return mediaTypes;
    }
}
```

### 3. HttpMessageConverter interface changes -- adds MediaType awareness

The converter interface gains three things: `getSupportedMediaTypes()`, MediaType parameters on `canRead()`/`canWrite()`, and a MediaType parameter on `write()`. This enables content negotiation: the framework can now ask "can you write this object *as application/json*?" instead of just "can you write this object?"

**Modified file:** `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java`

```java
public interface HttpMessageConverter {

    List<MediaType> getSupportedMediaTypes();                        // NEW
    boolean canRead(Class<?> clazz, MediaType mediaType);           // NEW: was canRead(Class)
    Object read(Class<?> clazz, HttpServletRequest request) throws Exception;
    boolean canWrite(Class<?> clazz, MediaType mediaType);          // NEW: was canWrite(Class)
    void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception;  // NEW: takes MediaType
}
```

When `mediaType` is `null`, it means "discovery mode" -- the caller is asking "can you handle this type at all?" When a specific MediaType is given, it means "can you handle this type for this specific media type?" This dual mode is used during content negotiation: first, gather all producible types (null), then check if a converter can write the selected type.

### 4. JacksonMessageConverter updates

Now media-type-aware. `canRead()` and `canWrite()` check compatibility with `application/json`. `write()` uses the negotiated MediaType for the Content-Type header.

**Modified file:** `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java`

```java
@Override
public boolean canWrite(Class<?> clazz, MediaType mediaType) {
    if (mediaType == null) return true;    // discovery mode: Jackson can write anything
    return MediaType.APPLICATION_JSON.isCompatibleWith(mediaType);  // check compatibility
}

@Override
public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
    String contentType = (mediaType != null) ? mediaType.toString() : MediaType.APPLICATION_JSON.toString();
    response.setContentType(contentType);
    response.setCharacterEncoding("UTF-8");
    objectMapper.writeValue(response.getWriter(), value);
}
```

### 5. PlainTextMessageConverter -- makes negotiation meaningful

Content negotiation only matters when there are multiple converters for different media types. Without `PlainTextMessageConverter`, JacksonMessageConverter handles everything. With it, a client can request `Accept: text/plain` and get a raw string instead of a JSON-quoted string.

**New file:** `src/main/java/com/simplespringmvc/converter/PlainTextMessageConverter.java`

```java
public class PlainTextMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED = List.of(MediaType.TEXT_PLAIN);

    @Override
    public List<MediaType> getSupportedMediaTypes() { return SUPPORTED; }

    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (clazz != String.class) return false;
        if (mediaType == null) return true;
        return MediaType.TEXT_PLAIN.isCompatibleWith(mediaType);
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (clazz != String.class) return false;
        if (mediaType == null) return true;
        return MediaType.TEXT_PLAIN.isCompatibleWith(mediaType);
    }

    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        response.setContentType(MediaType.TEXT_PLAIN.toString());
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(value.toString());
        response.getWriter().flush();
    }
}
```

### 6. @RequestMapping produces/consumes attributes

All mapping annotations gain `produces()` and `consumes()` attributes. In the real framework, these are forwarded from composed annotations to `@RequestMapping` via `@AliasFor`. We read them directly via reflection.

**Modified file:** `src/main/java/com/simplespringmvc/annotation/RequestMapping.java`

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestMapping {
    String path() default "";
    String method() default "";
    String[] produces() default {};   // NEW
    String[] consumes() default {};   // NEW
}
```

The same attributes are added to `@GetMapping`, `@PostMapping`, `@PutMapping`, and `@DeleteMapping`:

```java
@GetMapping(value = "/json-only", produces = "application/json")
@PostMapping(value = "/consume-json", consumes = "application/json")
```

`MergedAnnotationUtils` gains `getStringArrayAttribute()` to read these `String[]` attributes from composed annotations via reflection.

### 7. RouteKey produces/consumes matching

`RouteKey` stores parsed `List<MediaType>` for both produces and consumes. Two new methods check compatibility at handler lookup time.

**Modified file:** `src/main/java/com/simplespringmvc/mapping/RouteKey.java`

```java
public boolean matchesConsumes(String contentType) {
    if (consumes.isEmpty()) return true;       // no restriction
    if (contentType == null || contentType.isBlank()) return false;
    MediaType requestType = MediaType.parseMediaType(contentType);
    for (MediaType consumed : consumes) {
        if (consumed.includes(requestType)) return true;
    }
    return false;
}

public boolean matchesProduces(String acceptHeader) {
    if (produces.isEmpty()) return true;       // no restriction
    List<MediaType> acceptedTypes;
    if (acceptHeader == null || acceptHeader.isBlank()) {
        acceptedTypes = List.of(MediaType.ALL);
    } else {
        acceptedTypes = MediaType.parseMediaTypes(acceptHeader);
    }
    for (MediaType produced : produces) {
        for (MediaType accepted : acceptedTypes) {
            if (produced.isCompatibleWith(accepted)) return true;
        }
    }
    return false;
}
```

`equals()` and `hashCode()` now include produces/consumes, so the same path+method with different produces values are different keys. This allows multiple handlers for the same URL differentiated by content type.

### 8. SimpleHandlerMapping produces/consumes checking

Handler detection now reads `produces` and `consumes` from annotations and stores them in `RouteKey`. During lookup, the new `PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE` is set on the request so the content negotiation algorithm in `ResponseBodyReturnValueHandler` can use the declared types.

**Modified file:** `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java`

```java
// During handler detection:
produces = MergedAnnotationUtils.getStringArrayAttribute(composed, "produces");
consumes = MergedAnnotationUtils.getStringArrayAttribute(composed, "consumes");
RouteKey routeKey = new RouteKey(fullPath, httpMethod, produces, consumes);

// During lookup, after path/method match:
if (!key.matchesConsumes(request.getContentType())) continue;
if (!key.matchesProduces(request.getHeader("Accept"))) continue;

// Store for content negotiation
if (!key.getProduces().isEmpty()) {
    request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE, new ArrayList<>(key.getProduces()));
}
```

### 9. DispatcherServlet 406 handling

`SimpleDispatcherServlet.doDispatch()` catches `HttpMediaTypeNotAcceptableException` and sends a 406 response. This maps to the real framework's `DefaultHandlerExceptionResolver` which maps this exception type to HTTP 406.

**Modified file:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
} catch (HttpMediaTypeNotAcceptableException ex) {
    // ch14: The client's Accept header doesn't match any type the server can produce.
    response.sendError(HttpServletResponse.SC_NOT_ACCEPTABLE, ex.getMessage());
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
    return;
} catch (Exception ex) {
    dispatchException = ex;
}
```

---

## 14.3 Try It Yourself

<details>
<summary>Challenge: Implement the cross-match algorithm in ResponseBodyReturnValueHandler</summary>

Given a list of acceptable types (from Accept header) and producible types (from converters or @RequestMapping(produces)), find all compatible pairs. For each compatible pair, pick the more specific type and carry the quality value from the acceptable type.

The algorithm:
1. For each acceptable type, for each producible type: if compatible, add the most specific to the result
2. Sort the result by specificity and quality
3. Pick the first concrete type
4. Find the converter that can write to that type

```java
List<MediaType> compatibleTypes = new ArrayList<>();
for (MediaType acceptable : acceptableTypes) {
    for (MediaType producible : producibleTypes) {
        if (acceptable.isCompatibleWith(producible)) {
            compatibleTypes.add(getMostSpecific(acceptable, producible));
        }
    }
}

if (compatibleTypes.isEmpty()) {
    throw new HttpMediaTypeNotAcceptableException(producibleTypes);
}

MediaType.sortBySpecificityAndQuality(compatibleTypes);

MediaType selectedType = null;
for (MediaType type : compatibleTypes) {
    if (type.isConcrete()) {
        selectedType = type;
        break;
    }
    if (type.isWildcardType() || type.isWildcardSubtype()) {
        selectedType = MediaType.APPLICATION_OCTET_STREAM;
        break;
    }
}
```

The `getMostSpecific()` helper picks the more concrete of two compatible types and transfers the quality value from the client's preference:

```java
private MediaType getMostSpecific(MediaType acceptable, MediaType producible) {
    if (acceptable.isWildcardType() || acceptable.isWildcardSubtype()) {
        return producible.copyQualityValue(acceptable);
    }
    return acceptable;
}
```

</details>

<details>
<summary>Challenge: Implement RouteKey.matchesProduces()</summary>

Given a handler's declared produces types and the client's Accept header, determine if the handler can serve this request. The handler matches if any declared produces type is compatible with any accepted type.

Think about edge cases:
- What if produces is empty? (match anything)
- What if Accept header is null? (client accepts anything -- `*/*`)
- What does "compatible" mean here? (symmetric -- either side can have wildcards)

```java
public boolean matchesProduces(String acceptHeader) {
    if (produces.isEmpty()) {
        return true;  // no restriction
    }

    List<MediaType> acceptedTypes;
    if (acceptHeader == null || acceptHeader.isBlank()) {
        acceptedTypes = List.of(MediaType.ALL);
    } else {
        acceptedTypes = MediaType.parseMediaTypes(acceptHeader);
    }

    for (MediaType produced : produces) {
        for (MediaType accepted : acceptedTypes) {
            if (produced.isCompatibleWith(accepted)) {
                return true;
            }
        }
    }
    return false;
}
```

</details>

---

## 14.4 Tests

### MediaTypeTest

```java
@Test
@DisplayName("should parse simple type/subtype")
void shouldParseSimpleMediaType() {
    MediaType type = MediaType.parseMediaType("application/json");
    assertThat(type.getType()).isEqualTo("application");
    assertThat(type.getSubtype()).isEqualTo("json");
    assertThat(type.getParameters()).isEmpty();
}

@Test
@DisplayName("should parse type with quality parameter")
void shouldParseWithQualityParameter() {
    MediaType type = MediaType.parseMediaType("text/html;q=0.9");
    assertThat(type.getQualityValue()).isEqualTo(0.9);
}

@Test
@DisplayName("*/* should include everything")
void wildcardShouldIncludeEverything() {
    assertThat(MediaType.ALL.includes(MediaType.APPLICATION_JSON)).isTrue();
    assertThat(MediaType.ALL.includes(MediaType.TEXT_PLAIN)).isTrue();
}

@Test
@DisplayName("application/json should NOT include */*")
void concreteShouldNotIncludeWildcard() {
    assertThat(MediaType.APPLICATION_JSON.includes(MediaType.ALL)).isFalse();
}

@Test
@DisplayName("should be symmetric for wildcards")
void shouldBeSymmetricForWildcards() {
    assertThat(MediaType.APPLICATION_JSON.isCompatibleWith(MediaType.ALL)).isTrue();
    assertThat(MediaType.ALL.isCompatibleWith(MediaType.APPLICATION_JSON)).isTrue();
}

@Test
@DisplayName("should sort by quality (highest first)")
void shouldSortByQuality() {
    List<MediaType> types = new ArrayList<>(List.of(
            MediaType.parseMediaType("text/html;q=0.5"),
            MediaType.parseMediaType("application/json;q=0.9"),
            MediaType.parseMediaType("text/plain;q=1.0")
    ));
    MediaType.sortBySpecificityAndQuality(types);
    assertThat(types.get(0)).isEqualTo(MediaType.TEXT_PLAIN);
    assertThat(types.get(1)).isEqualTo(MediaType.APPLICATION_JSON);
    assertThat(types.get(2)).isEqualTo(MediaType.TEXT_HTML);
}
```

### ContentNegotiationManagerTest

```java
@Test
@DisplayName("should return [*/*] when no Accept header")
void shouldReturnAll_WhenNoAcceptHeader() {
    when(request.getHeader("Accept")).thenReturn(null);
    List<MediaType> types = manager.resolveMediaTypes(request);
    assertThat(types).containsExactly(MediaType.ALL);
}

@Test
@DisplayName("should parse multiple Accept types sorted by quality")
void shouldParseMultipleTypes_SortedByQuality() {
    when(request.getHeader("Accept")).thenReturn(
            "text/html;q=0.9, application/json, */*;q=0.1");
    List<MediaType> types = manager.resolveMediaTypes(request);
    // Sorted: application/json (q=1.0), text/html (q=0.9), */* (q=0.1)
    assertThat(types.get(0)).isEqualTo(MediaType.APPLICATION_JSON);
    assertThat(types.get(1)).isEqualTo(MediaType.TEXT_HTML);
    assertThat(types.get(2)).isEqualTo(MediaType.ALL);
}
```

### PlainTextMessageConverterTest

```java
@Test
@DisplayName("should write String as text/plain")
void shouldWriteString_WhenTextPlain() {
    assertThat(converter.canWrite(String.class, MediaType.TEXT_PLAIN)).isTrue();
}

@Test
@DisplayName("should NOT write String as application/json")
void shouldNotWriteString_WhenApplicationJson() {
    assertThat(converter.canWrite(String.class, MediaType.APPLICATION_JSON)).isFalse();
}

@Test
@DisplayName("should write string value as plain text")
void shouldWritePlainText() throws Exception {
    converter.write("Hello, World!", MediaType.TEXT_PLAIN, response);
    assertThat(responseBody.toString()).isEqualTo("Hello, World!");
    verify(response).setContentType("text/plain");
}
```

### ContentNegotiationIntegrationTest

```java
@Test
@DisplayName("should return plain text for String when Accept is text/plain")
void shouldReturnPlainText_WhenAcceptIsTextPlain() throws Exception {
    HttpResponse<String> response = client.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/greeting"))
                    .header("Accept", "text/plain")
                    .GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.headers().firstValue("Content-Type").orElse(""))
            .contains("text/plain");
    assertThat(response.body()).isEqualTo("Hello, World!");
}

@Test
@DisplayName("should return 404 when Accept does not match produces")
void shouldReturn404_WhenAcceptDoesNotMatchProduces() throws Exception {
    // text/plain does not match produces="application/json" → handler not matched → 404
    HttpResponse<String> response = client.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/json-only"))
                    .header("Accept", "text/plain")
                    .GET().build(),
            HttpResponse.BodyHandlers.ofString());
    assertThat(response.statusCode()).isEqualTo(404);
}

@Test
@DisplayName("should return 406 when no converter can produce the requested type")
void shouldReturn406_WhenNoConverterCanProduceRequestedType() throws Exception {
    // Request XML, but we only have JSON and text/plain converters
    HttpResponse<String> response = client.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/data"))
                    .header("Accept", "application/xml")
                    .GET().build(),
            HttpResponse.BodyHandlers.ofString());
    assertThat(response.statusCode()).isEqualTo(406);
}

@Test
@DisplayName("should respect quality values in Accept header")
void shouldRespectQualityValues() throws Exception {
    // Prefer text/plain (q=1.0) over application/json (q=0.5)
    HttpResponse<String> response = client.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/greeting"))
                    .header("Accept", "application/json;q=0.5, text/plain")
                    .GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.headers().firstValue("Content-Type").orElse(""))
            .contains("text/plain");
    assertThat(response.body()).isEqualTo("Hello, World!");
}
```

---

## 14.5 Why This Works

> **Insight** -------------------------------------------
> - **The cross-match algorithm is O(n*m) -- and that is fine.** The content negotiation algorithm iterates every acceptable type against every producible type. With typical Accept headers containing 2-5 types and 2-3 converters, this means 4-15 iterations -- trivial. The real framework uses the same nested loop (`AbstractMessageConverterMethodProcessor` line 250-260). The cost of serializing the response body dwarfs the cost of picking which format to use. Over-optimizing type matching would add complexity (index structures, caching) for negligible gain.
> - **The quality value transfer via `copyQualityValue()` is subtle but important.** When `Accept: */*;q=0.1` matches `application/json`, the result is `application/json;q=0.1` -- the concrete type with the client's preference weight. Without this transfer, a low-priority wildcard match would sort equally with explicit high-priority types, breaking the client's stated preference ordering.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> - **Converter order matters: JacksonMessageConverter before PlainTextMessageConverter.** When `Accept: */*` (the default when no Accept header is sent), both converters are compatible. The algorithm picks the first concrete type after sorting, but since `*/*` matches both `application/json` and `text/plain` at equal quality, converter registration order becomes the tiebreaker. JSON first means REST API clients get JSON by default -- the most common expectation. Swapping the order would silently change the default behavior of every endpoint.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> - **How produces/consumes enables multiple handlers for the same URL.** Because `RouteKey.equals()` includes produces and consumes, you can register `GET /data produces=application/json` and `GET /data produces=text/xml` as separate handlers. The handler mapping checks the Accept header against each handler's produces condition and routes to the right one. This is why the real framework's `RequestMappingInfo` includes `ProducesRequestCondition` and `ConsumesRequestCondition` -- they are part of the mapping identity, not just a filter applied after matching.
> -----------------------------------------------------------

---

## 14.6 What We Enhanced

| Aspect | Before (ch13) | Current (ch14) | Real Framework |
|--------|--------------|----------------|----------------|
| Response format selection | First converter that `canWrite(valueType)` wins | Full content negotiation: Accept header parsing, cross-matching, quality sorting, converter selection | `AbstractMessageConverterMethodProcessor.writeWithMessageConverters()` (line 205) |
| `HttpMessageConverter` interface | `canRead(Class)`, `canWrite(Class)`, `write(Object, response)` | `canRead(Class, MediaType)`, `canWrite(Class, MediaType)`, `write(Object, MediaType, response)`, `getSupportedMediaTypes()` | `HttpMessageConverter<T>` with full MediaType parameters |
| Accept header handling | Completely ignored | Parsed by `ContentNegotiationManager`, sorted by quality and specificity | `ContentNegotiationManager` with pluggable strategies |
| `@RequestMapping` attributes | `path()` and `method()` only | Added `produces()` and `consumes()` | Full `produces`, `consumes`, `params`, `headers` |
| Handler matching | Path + HTTP method only | Path + HTTP method + Content-Type check + Accept header check | `RequestMappingInfo` with 8 request conditions |
| Unsupported Accept header | Silently returns whatever the first converter produces | 406 Not Acceptable with list of supported types | `HttpMediaTypeNotAcceptableException` handled by `DefaultHandlerExceptionResolver` |
| Message converters registered | Only `JacksonMessageConverter` | `JacksonMessageConverter` + `PlainTextMessageConverter` | Auto-detected from classpath (Jackson, JAXB, Gson, etc.) |
| `RequestBodyArgumentResolver` | Ignores Content-Type, picks first converter | Parses Content-Type and checks `canRead(type, mediaType)` | `AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()` |

---

## 14.7 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line (commit `11ab0b4351`) | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `MediaType` | `MediaType` extends `MimeType` | `MediaType.java:90`, `MimeType.java:67` | Real has suffix matching (`application/*+json`), full parameter comparison, caching |
| `ContentNegotiationManager` | `ContentNegotiationManager` | `ContentNegotiationManager.java:56` | Real holds `List<ContentNegotiationStrategy>` -- header, parameter, fixed strategies |
| `HttpMediaTypeNotAcceptableException` | `HttpMediaTypeNotAcceptableException` | `HttpMediaTypeNotAcceptableException.java:43` | Real extends `HttpMediaTypeException` extends `ServletException` |
| `PlainTextMessageConverter` | `StringHttpMessageConverter` | `StringHttpMessageConverter.java:53` | Real supports all `text/*` subtypes, charset negotiation, `*/*` fallback |
| `HttpMessageConverter` interface | `HttpMessageConverter<T>` | `HttpMessageConverter.java:47` | Real is generic, uses `HttpInputMessage`/`HttpOutputMessage` abstraction |
| `JacksonMessageConverter` | `MappingJackson2HttpMessageConverter` | `MappingJackson2HttpMessageConverter.java:60` | Real extends `AbstractJackson2HttpMessageConverter` with `application/*+json`, streaming |
| `ResponseBodyReturnValueHandler.handleReturnValue()` | `AbstractMessageConverterMethodProcessor.writeWithMessageConverters()` | `AbstractMessageConverterMethodProcessor.java:205` | Real has `ResponseBodyAdvice`, ProblemDetail handling, Resource/Range support |
| `RouteKey` produces/consumes | `ProducesRequestCondition` / `ConsumesRequestCondition` | `ProducesRequestCondition.java:42`, `ConsumesRequestCondition.java:43` | Real conditions support negation (`!text/html`), media type expressions, combining |
| `SimpleHandlerMapping.lookupHandler()` | `RequestMappingInfoHandlerMapping.getMatchingMapping()` | `RequestMappingInfoHandlerMapping.java:107` | Real uses `RequestMappingInfo.getMatchingCondition()` with all 8 conditions |
| `SimpleDispatcherServlet` 406 catch | `DefaultHandlerExceptionResolver` | `DefaultHandlerExceptionResolver.java:209` | Real resolves through exception resolver chain, not a direct catch block |
| `MergedAnnotationUtils.getStringArrayAttribute()` | `MergedAnnotation.getStringArray()` | `MergedAnnotation.java:220` | Real uses synthesized annotation proxies with `@AliasFor` attribute merging |

---

## 14.8 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/http/MediaType.java` [NEW]

```java
package com.simplespringmvc.http;

import java.util.*;

/**
 * Represents a MIME type as defined in RFC 2046 — the fundamental data type
 * for content negotiation. Parses media type strings like "application/json"
 * or "text/html;q=0.9" and supports matching, compatibility checks, and
 * quality-based sorting.
 *
 * Maps to: {@code org.springframework.http.MediaType} which extends
 * {@code org.springframework.util.MimeType}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   MimeType           → stores type/subtype/parameters, matching logic
 *     └── MediaType    → adds quality value, parsing of Accept headers,
 *                        sorting by specificity
 * </pre>
 *
 * We collapse both into a single class. Key operations:
 * <ul>
 *   <li>{@link #includes(MediaType)} — asymmetric: does this type include the other?</li>
 *   <li>{@link #isCompatibleWith(MediaType)} — symmetric: are the two types compatible?</li>
 *   <li>{@link #parseMediaTypes(String)} — parse comma-separated Accept header</li>
 *   <li>{@link #sortBySpecificityAndQuality(List)} — sort for content negotiation</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No suffix matching (e.g., application/*+xml)</li>
 *   <li>No parameter matching beyond quality value</li>
 *   <li>equals/hashCode use type+subtype only (real framework includes parameters)</li>
 *   <li>No caching of parsed media types</li>
 * </ul>
 */
public class MediaType implements Comparable<MediaType> {

    public static final String WILDCARD_TYPE = "*";
    private static final String PARAM_QUALITY = "q";

    // ─── Common media type constants ──────────────────────────────

    /** Matches any media type: {@code *}{@code /*} */
    public static final MediaType ALL = new MediaType("*", "*");

    /** JSON: {@code application/json} */
    public static final MediaType APPLICATION_JSON = new MediaType("application", "json");

    /** XML: {@code application/xml} */
    public static final MediaType APPLICATION_XML = new MediaType("application", "xml");

    /** Plain text: {@code text/plain} */
    public static final MediaType TEXT_PLAIN = new MediaType("text", "plain");

    /** HTML: {@code text/html} */
    public static final MediaType TEXT_HTML = new MediaType("text", "html");

    /** Binary fallback: {@code application/octet-stream} */
    public static final MediaType APPLICATION_OCTET_STREAM = new MediaType("application", "octet-stream");

    // ─── Instance fields ──────────────────────────────────────────

    private final String type;
    private final String subtype;
    private final Map<String, String> parameters;

    public MediaType(String type, String subtype) {
        this(type, subtype, Collections.emptyMap());
    }

    public MediaType(String type, String subtype, Map<String, String> parameters) {
        this.type = type.toLowerCase(Locale.ROOT);
        this.subtype = subtype.toLowerCase(Locale.ROOT);
        this.parameters = Collections.unmodifiableMap(new LinkedHashMap<>(parameters));
    }

    public String getType() {
        return type;
    }

    public String getSubtype() {
        return subtype;
    }

    public Map<String, String> getParameters() {
        return parameters;
    }

    // ─── Quality value ────────────────────────────────────────────

    /**
     * Return the quality value as indicated by the {@code q} parameter.
     * Defaults to {@code 1.0} (highest priority) when not specified.
     *
     * Maps to: {@code MediaType.getQualityValue()} (line 370)
     *
     * Quality values range from 0.0 (not acceptable) to 1.0 (most preferred).
     * Example: {@code text/html;q=0.9} means "I accept HTML but prefer other types".
     */
    public double getQualityValue() {
        String q = parameters.get(PARAM_QUALITY);
        return q != null ? Double.parseDouble(q) : 1.0;
    }

    /**
     * Return a copy with the quality value from another media type.
     * Used during content negotiation to transfer the client's preference
     * weight onto a producible type.
     *
     * Maps to: {@code MediaType.copyQualityValue()} (line 386)
     */
    public MediaType copyQualityValue(MediaType other) {
        double q = other.getQualityValue();
        if (q == 1.0 && !other.parameters.containsKey(PARAM_QUALITY)) {
            return this;
        }
        Map<String, String> params = new LinkedHashMap<>(this.parameters);
        params.put(PARAM_QUALITY, String.valueOf(q));
        return new MediaType(this.type, this.subtype, params);
    }

    /**
     * Return a copy without the quality value parameter.
     * Called before setting the Content-Type header — quality is a client
     * preference, not a response property.
     *
     * Maps to: {@code MediaType.removeQualityValue()} (line 400)
     */
    public MediaType removeQualityValue() {
        if (!parameters.containsKey(PARAM_QUALITY)) {
            return this;
        }
        Map<String, String> params = new LinkedHashMap<>(this.parameters);
        params.remove(PARAM_QUALITY);
        return new MediaType(this.type, this.subtype, params);
    }

    // ─── Wildcard checks ──────────────────────────────────────────

    /** True if the primary type is {@code *} (e.g., {@code * / *}). */
    public boolean isWildcardType() {
        return WILDCARD_TYPE.equals(type);
    }

    /** True if the subtype is {@code *} (e.g., {@code text/*}). */
    public boolean isWildcardSubtype() {
        return WILDCARD_TYPE.equals(subtype) || subtype.startsWith("*+");
    }

    /** True if neither type nor subtype is a wildcard. */
    public boolean isConcrete() {
        return !isWildcardType() && !isWildcardSubtype();
    }

    // ─── Matching ─────────────────────────────────────────────────

    /**
     * Whether this media type includes the given media type.
     * Asymmetric: {@code * / *} includes {@code application/json}, but not vice versa.
     *
     * Maps to: {@code MimeType.includes(MimeType)} (line 253)
     *
     * <pre>
     *   * / *         includes everything
     *   text/*        includes text/plain, text/html
     *   text/plain    includes only text/plain
     *   text/plain    does NOT include * / *
     * </pre>
     */
    public boolean includes(MediaType other) {
        if (other == null) {
            return false;
        }
        if (isWildcardType()) {
            return true;
        }
        if (this.type.equals(other.type)) {
            if (isWildcardSubtype()) {
                return true;
            }
            return this.subtype.equals(other.subtype);
        }
        return false;
    }

    /**
     * Whether this media type is compatible with the given media type.
     * Symmetric: either side can have wildcards.
     *
     * Maps to: {@code MimeType.isCompatibleWith(MimeType)} (line 296)
     *
     * <pre>
     *   application/json.isCompatibleWith(application/json)   → true
     *   application/json.isCompatibleWith(* / *)              → true
     *   * / *.isCompatibleWith(application/json)              → true
     *   text/*.isCompatibleWith(text/plain)                   → true
     *   text/plain.isCompatibleWith(text/*)                   → true
     *   application/json.isCompatibleWith(text/plain)         → false
     * </pre>
     */
    public boolean isCompatibleWith(MediaType other) {
        if (other == null) {
            return false;
        }
        if (isWildcardType() || other.isWildcardType()) {
            return true;
        }
        if (this.type.equals(other.type)) {
            if (isWildcardSubtype() || other.isWildcardSubtype()) {
                return true;
            }
            return this.subtype.equals(other.subtype);
        }
        return false;
    }

    // ─── Parsing ──────────────────────────────────────────────────

    /**
     * Parse a single media type string.
     * Examples: "application/json", "text/html;charset=UTF-8", "text/html;q=0.9"
     *
     * Maps to: {@code MediaType.parseMediaType(String)} → {@code MimeTypeUtils.parseMimeTypeInternal()}
     */
    public static MediaType parseMediaType(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("MediaType must not be blank");
        }
        value = value.trim();
        String[] parts = value.split(";");
        String fullType = parts[0].trim();

        int slashIndex = fullType.indexOf('/');
        String type;
        String subtype;
        if (slashIndex < 0) {
            // No slash — treat as type with wildcard subtype
            type = fullType;
            subtype = "*";
        } else {
            type = fullType.substring(0, slashIndex).trim();
            subtype = fullType.substring(slashIndex + 1).trim();
        }

        Map<String, String> parameters = new LinkedHashMap<>();
        for (int i = 1; i < parts.length; i++) {
            String param = parts[i].trim();
            int eqIdx = param.indexOf('=');
            if (eqIdx > 0) {
                String key = param.substring(0, eqIdx).trim().toLowerCase(Locale.ROOT);
                String val = param.substring(eqIdx + 1).trim();
                // Strip quotes from parameter values
                if (val.length() >= 2 && val.startsWith("\"") && val.endsWith("\"")) {
                    val = val.substring(1, val.length() - 1);
                }
                parameters.put(key, val);
            }
        }

        return new MediaType(type, subtype, parameters);
    }

    /**
     * Parse a comma-separated list of media types (the format used in the Accept header).
     * Example: "text/html, application/json;q=0.9, *&#47;*;q=0.1"
     *
     * Maps to: {@code MediaType.parseMediaTypes(String)} → {@code MimeTypeUtils.tokenize()}
     */
    public static List<MediaType> parseMediaTypes(String value) {
        if (value == null || value.isBlank()) {
            return new ArrayList<>(List.of(ALL));
        }
        List<MediaType> result = new ArrayList<>();
        for (String token : tokenize(value)) {
            result.add(parseMediaType(token));
        }
        return result;
    }

    /**
     * Split a comma-separated media type string.
     * Handles the fact that parameters contain semicolons (not commas),
     * so splitting on comma is safe.
     */
    private static List<String> tokenize(String value) {
        List<String> tokens = new ArrayList<>();
        StringBuilder current = new StringBuilder();
        for (int i = 0; i < value.length(); i++) {
            char c = value.charAt(i);
            if (c == ',') {
                String token = current.toString().trim();
                if (!token.isEmpty()) {
                    tokens.add(token);
                }
                current = new StringBuilder();
            } else {
                current.append(c);
            }
        }
        String last = current.toString().trim();
        if (!last.isEmpty()) {
            tokens.add(last);
        }
        return tokens;
    }

    // ─── Sorting ──────────────────────────────────────────────────

    /**
     * Sort media types by specificity and quality for content negotiation.
     * Most preferred types come first.
     *
     * Sort order:
     * <ol>
     *   <li>Higher quality value first (q=1.0 before q=0.9)</li>
     *   <li>More specific type first (concrete > wildcard subtype > wildcard type)</li>
     *   <li>More parameters = more specific</li>
     * </ol>
     *
     * Maps to: {@code MimeTypeUtils.sortBySpecificity()} combined with
     * quality value comparison from {@code MediaType.isMoreSpecific()}
     */
    public static void sortBySpecificityAndQuality(List<MediaType> mediaTypes) {
        mediaTypes.sort((a, b) -> {
            // Higher quality first
            int qCompare = Double.compare(b.getQualityValue(), a.getQualityValue());
            if (qCompare != 0) return qCompare;

            // More specific first: concrete (2) > wildcard subtype (1) > wildcard type (0)
            int specA = a.isWildcardType() ? 0 : (a.isWildcardSubtype() ? 1 : 2);
            int specB = b.isWildcardType() ? 0 : (b.isWildcardSubtype() ? 1 : 2);
            int specCompare = Integer.compare(specB, specA);
            if (specCompare != 0) return specCompare;

            // More parameters = more specific
            return Integer.compare(b.getParameters().size(), a.getParameters().size());
        });
    }

    // ─── Object methods ───────────────────────────────────────────

    @Override
    public int compareTo(MediaType other) {
        int specA = isWildcardType() ? 0 : (isWildcardSubtype() ? 1 : 2);
        int specB = other.isWildcardType() ? 0 : (other.isWildcardSubtype() ? 1 : 2);
        int specCompare = Integer.compare(specB, specA);
        if (specCompare != 0) return specCompare;
        return Double.compare(other.getQualityValue(), getQualityValue());
    }

    /**
     * Two media types are equal if they have the same type and subtype.
     * Parameters (including quality) are ignored for equality.
     *
     * Simplification: the real framework includes parameters in equals().
     */
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof MediaType that)) return false;
        return type.equals(that.type) && subtype.equals(that.subtype);
    }

    @Override
    public int hashCode() {
        return Objects.hash(type, subtype);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append(type).append('/').append(subtype);
        parameters.forEach((key, value) -> sb.append(';').append(key).append('=').append(value));
        return sb.toString();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/http/ContentNegotiationManager.java` [NEW]

```java
package com.simplespringmvc.http;

import jakarta.servlet.http.HttpServletRequest;

import java.util.List;

/**
 * Central class to determine the requested media types for an HTTP request,
 * reading the {@code Accept} header to determine what content types the client
 * wants.
 *
 * Maps to: {@code org.springframework.web.accept.ContentNegotiationManager}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   ContentNegotiationManager
 *     holds List&lt;ContentNegotiationStrategy&gt;
 *       ├── HeaderContentNegotiationStrategy  (reads Accept header — default)
 *       ├── PathExtensionContentNegotiationStrategy (deprecated)
 *       └── ParameterContentNegotiationStrategy (?format=json)
 *
 *   resolveMediaTypes() iterates strategies, first non-wildcard result wins.
 * </pre>
 *
 * We simplify to a single strategy: read the Accept header. This covers the
 * overwhelming majority of real-world content negotiation.
 *
 * Simplifications:
 * <ul>
 *   <li>No strategy pattern — just Accept header parsing</li>
 *   <li>No path extension or query parameter strategies</li>
 *   <li>No media type mappings configuration</li>
 * </ul>
 */
public class ContentNegotiationManager {

    /**
     * Resolve the list of media types the client wants, based on the Accept header.
     * Returns types sorted by quality and specificity (most preferred first).
     *
     * Maps to: {@code ContentNegotiationManager.resolveMediaTypes(NativeWebRequest)}
     * → delegates to {@code HeaderContentNegotiationStrategy.resolveMediaTypes()}
     *
     * <h3>Algorithm:</h3>
     * <ol>
     *   <li>Read the {@code Accept} header from the request</li>
     *   <li>If absent or empty, return {@code [*&#47;*]} (client accepts anything)</li>
     *   <li>Parse into a list of MediaTypes</li>
     *   <li>Sort by specificity and quality (highest quality, most specific first)</li>
     * </ol>
     *
     * @param request the current HTTP request
     * @return sorted list of acceptable media types, never empty
     */
    public List<MediaType> resolveMediaTypes(HttpServletRequest request) {
        String acceptHeader = request.getHeader("Accept");

        if (acceptHeader == null || acceptHeader.isBlank()) {
            // No Accept header → client accepts anything.
            // Maps to: HeaderContentNegotiationStrategy returning MEDIA_TYPE_ALL_LIST
            return List.of(MediaType.ALL);
        }

        List<MediaType> mediaTypes = MediaType.parseMediaTypes(acceptHeader);

        if (mediaTypes.isEmpty()) {
            return List.of(MediaType.ALL);
        }

        MediaType.sortBySpecificityAndQuality(mediaTypes);
        return mediaTypes;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/http/HttpMediaTypeNotAcceptableException.java` [NEW]

```java
package com.simplespringmvc.http;

import java.util.List;

/**
 * Exception thrown when the server cannot produce a response in a format
 * acceptable to the client (based on the Accept header). Results in HTTP 406
 * Not Acceptable.
 *
 * Maps to: {@code org.springframework.web.HttpMediaTypeNotAcceptableException}
 *
 * In the real framework, this extends {@code HttpMediaTypeException} which extends
 * {@code ServletException}. It's handled by {@code DefaultHandlerExceptionResolver}
 * which maps it to a 406 status code.
 *
 * We use a RuntimeException for simplicity. The DispatcherServlet catches it
 * directly and sends a 406 response.
 */
public class HttpMediaTypeNotAcceptableException extends RuntimeException {

    private final List<MediaType> supportedMediaTypes;

    public HttpMediaTypeNotAcceptableException(List<MediaType> supportedMediaTypes) {
        super("Could not find acceptable representation. Supported media types: " + supportedMediaTypes);
        this.supportedMediaTypes = supportedMediaTypes;
    }

    public HttpMediaTypeNotAcceptableException(String message) {
        super(message);
        this.supportedMediaTypes = List.of();
    }

    /**
     * The media types the server can produce for the requested resource.
     */
    public List<MediaType> getSupportedMediaTypes() {
        return supportedMediaTypes;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/converter/PlainTextMessageConverter.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.nio.charset.StandardCharsets;
import java.util.List;

/**
 * An {@link HttpMessageConverter} for reading and writing plain text ({@code text/plain}).
 *
 * Maps to: {@code org.springframework.http.converter.StringHttpMessageConverter}
 *
 * The real framework's StringHttpMessageConverter supports all text/* types and
 * can also handle *&#47;* (making it a very flexible fallback). We only support
 * text/plain to keep the content negotiation demo clean — JSON for objects,
 * plain text for strings when explicitly requested.
 *
 * <h3>Why this converter exists:</h3>
 * Content negotiation is only meaningful when there are multiple converters that
 * can handle different media types. Without this converter, JacksonMessageConverter
 * handles everything as JSON. With it, a client can request {@code Accept: text/plain}
 * and get a plain string instead of a JSON-quoted string.
 *
 * Simplifications:
 * <ul>
 *   <li>Only handles String.class (real version handles CharSequence subtypes too)</li>
 *   <li>Only text/plain (real version supports all text/* subtypes)</li>
 *   <li>No charset negotiation (always UTF-8)</li>
 * </ul>
 */
public class PlainTextMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED = List.of(MediaType.TEXT_PLAIN);

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return SUPPORTED;
    }

    /**
     * Can read into String.class when the Content-Type is text/plain (or unknown).
     *
     * @param clazz     the target class (must be String.class)
     * @param mediaType the Content-Type of the request (null = don't care)
     */
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (clazz != String.class) {
            return false;
        }
        if (mediaType == null) {
            return true;
        }
        return MediaType.TEXT_PLAIN.isCompatibleWith(mediaType);
    }

    /**
     * Read the request body as a String.
     */
    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        return new String(request.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    }

    /**
     * Can write String values as text/plain.
     *
     * @param clazz     the value class (must be String.class)
     * @param mediaType the target media type (null = discovery mode)
     */
    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (clazz != String.class) {
            return false;
        }
        if (mediaType == null) {
            return true;
        }
        return MediaType.TEXT_PLAIN.isCompatibleWith(mediaType);
    }

    /**
     * Write the value as plain text to the response.
     */
    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        response.setContentType(MediaType.TEXT_PLAIN.toString());
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(value.toString());
        response.getWriter().flush();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/converter/HttpMessageConverter.java` [MODIFIED]

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

/**
 * Strategy interface for converting between HTTP messages and Java objects.
 * Each implementation handles one or more media types (e.g., JSON, XML, plain text).
 *
 * Maps to: {@code org.springframework.http.converter.HttpMessageConverter<T>}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   HttpMessageConverter&lt;T&gt;
 *     canRead(Class, MediaType)       → can this converter read the given type?
 *     read(Class, HttpInputMessage)   → deserialize from the input
 *     canWrite(Class, MediaType)      → can this converter write the given type?
 *     write(T, MediaType, HttpOutputMessage) → serialize to the output
 *     getSupportedMediaTypes()        → what media types does this converter handle?
 * </pre>
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@link MediaType} parameters to {@code canRead}, {@code canWrite}, and
 * {@code write}, plus a {@link #getSupportedMediaTypes()} method. This enables
 * content negotiation: the framework can now ask "can you write this object
 * <em>as application/json</em>?" instead of just "can you write this object?"
 *
 * <h3>How MediaType parameters work:</h3>
 * <ul>
 *   <li>{@code canRead/canWrite} with {@code mediaType = null} → "can you handle
 *       this type at all?" (discovery mode, used when collecting producible types)</li>
 *   <li>{@code canRead/canWrite} with a specific MediaType → "can you handle this
 *       type for this specific media type?" (selection mode)</li>
 *   <li>{@code write} receives the negotiated MediaType so the converter can set
 *       the correct Content-Type header</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Not generic (uses Object instead of type parameter T)</li>
 *   <li>Takes HttpServletRequest/Response directly instead of HttpInputMessage/HttpOutputMessage</li>
 *   <li>read() does not take MediaType (the Content-Type was already checked via canRead)</li>
 * </ul>
 */
public interface HttpMessageConverter {

    /**
     * Return the list of media types this converter supports.
     * Used during content negotiation to determine what types the server can produce.
     *
     * Maps to: {@code HttpMessageConverter.getSupportedMediaTypes()}
     */
    List<MediaType> getSupportedMediaTypes();

    /**
     * Whether this converter can read the given class from the given media type.
     *
     * @param clazz     the class to test for readability
     * @param mediaType the media type to read (null = can you read this class at all?)
     * @return true if readable
     */
    boolean canRead(Class<?> clazz, MediaType mediaType);

    /**
     * Read an object from the request body.
     *
     * @param clazz   the type to read into
     * @param request the HTTP request containing the body
     * @return the deserialized object
     */
    Object read(Class<?> clazz, HttpServletRequest request) throws Exception;

    /**
     * Whether this converter can write the given class to the given media type.
     *
     * @param clazz     the class to test for writability
     * @param mediaType the media type to write (null = can you write this class at all?)
     * @return true if writable
     */
    boolean canWrite(Class<?> clazz, MediaType mediaType);

    /**
     * Write an object to the response body in the given media type.
     *
     * @param value     the object to write
     * @param mediaType the negotiated media type to use for the response Content-Type
     * @param response  the HTTP response to write to
     */
    void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/converter/JacksonMessageConverter.java` [MODIFIED]

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

/**
 * An {@link HttpMessageConverter} that converts between Java objects and JSON
 * using Jackson's {@link ObjectMapper}.
 *
 * Maps to: {@code org.springframework.http.converter.json.MappingJackson2HttpMessageConverter}
 *
 * <h3>ch14 Enhancement:</h3>
 * Now media-type-aware. {@code canRead()} and {@code canWrite()} check compatibility
 * with {@code application/json}. {@code write()} uses the negotiated MediaType to set
 * the response Content-Type header, instead of hard-coding it.
 *
 * Simplifications:
 * <ul>
 *   <li>Supports only application/json (real version also supports application/*+json)</li>
 *   <li>No generic type support (real version uses GenericHttpMessageConverter)</li>
 *   <li>No encoding negotiation — always UTF-8</li>
 * </ul>
 */
public class JacksonMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED = List.of(MediaType.APPLICATION_JSON);

    private final ObjectMapper objectMapper;

    public JacksonMessageConverter() {
        this.objectMapper = new ObjectMapper();
    }

    public JacksonMessageConverter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return SUPPORTED;
    }

    /**
     * Jackson can deserialize JSON into most Java types.
     * When mediaType is null (discovery mode), returns true.
     * When a specific mediaType is given, checks compatibility with application/json.
     *
     * Maps to: {@code AbstractJackson2HttpMessageConverter.canRead(Class, MediaType)}
     */
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (mediaType == null) {
            return true;
        }
        return MediaType.APPLICATION_JSON.isCompatibleWith(mediaType);
    }

    /**
     * Deserialize JSON from the request body into an object of the given type.
     *
     * Maps to: {@code AbstractJackson2HttpMessageConverter.readInternal()}
     */
    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        return objectMapper.readValue(request.getInputStream(), clazz);
    }

    /**
     * Jackson can write most Java objects as JSON.
     * When mediaType is null (discovery mode), returns true.
     * When a specific mediaType is given, checks compatibility with application/json.
     *
     * <h3>ch14 Enhancement:</h3>
     * Previously returned true unconditionally. Now checks media type compatibility
     * so that a request for text/plain won't match this converter.
     */
    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (mediaType == null) {
            return true;
        }
        return MediaType.APPLICATION_JSON.isCompatibleWith(mediaType);
    }

    /**
     * Serialize the given object to JSON and write to the response.
     *
     * <h3>ch14 Enhancement:</h3>
     * Uses the negotiated {@code mediaType} for the Content-Type header
     * instead of hard-coding "application/json".
     */
    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        String contentType = (mediaType != null) ? mediaType.toString() : MediaType.APPLICATION_JSON.toString();
        response.setContentType(contentType);
        response.setCharacterEncoding("UTF-8");
        objectMapper.writeValue(response.getWriter(), value);
    }

    public ObjectMapper getObjectMapper() {
        return objectMapper;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.http.ContentNegotiationManager;
import com.simplespringmvc.http.HttpMediaTypeNotAcceptableException;
import com.simplespringmvc.http.MediaType;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.ArrayList;
import java.util.List;

/**
 * Handles return values from methods annotated with {@code @ResponseBody} by
 * serializing them to the HTTP response using content negotiation to select
 * the appropriate {@link HttpMessageConverter} and media type.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor}
 * which extends {@code AbstractMessageConverterMethodProcessor}
 *
 * <h3>ch14 Enhancement — Content Negotiation Algorithm:</h3>
 * This is the main integration point for content negotiation. Previously, this handler
 * simply picked the first converter that could write the value type. Now it implements
 * the full negotiation algorithm from {@code AbstractMessageConverterMethodProcessor.writeWithMessageConverters()}:
 *
 * <pre>
 *   1. Determine acceptable types from the client's Accept header
 *   2. Determine producible types (from @RequestMapping(produces=...) or from all converters)
 *   3. Cross-match acceptable × producible to find compatible types
 *   4. Sort compatible types by specificity and quality
 *   5. Pick the first concrete type
 *   6. Find the converter that can write to that type
 *   7. Write the response
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No ResponseBodyAdvice support (beforeBodyWrite hook)</li>
 *   <li>No ProblemDetail fallback</li>
 *   <li>No special handling for Resource types or HTTP Range</li>
 * </ul>
 */
public class ResponseBodyReturnValueHandler implements HandlerMethodReturnValueHandler {

    private final List<HttpMessageConverter> messageConverters;
    private final ContentNegotiationManager contentNegotiationManager;

    public ResponseBodyReturnValueHandler(List<HttpMessageConverter> messageConverters,
                                          ContentNegotiationManager contentNegotiationManager) {
        this.messageConverters = messageConverters;
        this.contentNegotiationManager = contentNegotiationManager;
    }

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        if (handlerMethod.getMethod().isAnnotationPresent(ResponseBody.class)) {
            return true;
        }
        return MergedAnnotationUtils.hasAnnotation(
                handlerMethod.getBeanType(), ResponseBody.class);
    }

    /**
     * Serialize the return value using content negotiation.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.handleReturnValue()} →
     * {@code AbstractMessageConverterMethodProcessor.writeWithMessageConverters()}
     * (lines 205-368)
     *
     * <h3>The content negotiation algorithm:</h3>
     * <ol>
     *   <li><b>Acceptable types</b> — what the client wants (from Accept header)</li>
     *   <li><b>Producible types</b> — what the server can produce (from converters or @RequestMapping(produces))</li>
     *   <li><b>Compatible types</b> — intersection of acceptable and producible</li>
     *   <li><b>Selected type</b> — the most specific, highest quality compatible type</li>
     *   <li><b>Converter</b> — the first converter that can write the value as the selected type</li>
     * </ol>
     */
    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue == null) {
            response.setStatus(HttpServletResponse.SC_OK);
            return null;
        }

        Class<?> valueType = returnValue.getClass();

        // ── Step 1: Get acceptable media types from Accept header ───────
        // Maps to: AbstractMessageConverterMethodProcessor line 238:
        //   List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
        List<MediaType> acceptableTypes = contentNegotiationManager.resolveMediaTypes(request);

        // ── Step 2: Get producible media types ──────────────────────────
        // Maps to: AbstractMessageConverterMethodProcessor line 240:
        //   List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
        List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType);

        // ── Step 3: Cross-match to find compatible types ────────────────
        // Maps to: AbstractMessageConverterMethodProcessor line 250-260
        List<MediaType> compatibleTypes = new ArrayList<>();
        for (MediaType acceptable : acceptableTypes) {
            for (MediaType producible : producibleTypes) {
                if (acceptable.isCompatibleWith(producible)) {
                    compatibleTypes.add(getMostSpecific(acceptable, producible));
                }
            }
        }

        if (compatibleTypes.isEmpty()) {
            throw new HttpMediaTypeNotAcceptableException(producibleTypes);
        }

        // ── Step 4: Sort by specificity and quality ─────────────────────
        MediaType.sortBySpecificityAndQuality(compatibleTypes);

        // ── Step 5: Pick the first concrete type ────────────────────────
        // Maps to: AbstractMessageConverterMethodProcessor line 272-283
        MediaType selectedType = null;
        for (MediaType type : compatibleTypes) {
            if (type.isConcrete()) {
                selectedType = type;
                break;
            }
            if (type.isWildcardType() || type.isWildcardSubtype()) {
                selectedType = MediaType.APPLICATION_OCTET_STREAM;
                break;
            }
        }

        if (selectedType == null) {
            throw new HttpMediaTypeNotAcceptableException(producibleTypes);
        }

        // Strip quality value — it's a client preference, not a response property
        selectedType = selectedType.removeQualityValue();

        // ── Step 6: Find a converter and write ──────────────────────────
        // Maps to: AbstractMessageConverterMethodProcessor line 288-352
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(valueType, selectedType)) {
                converter.write(returnValue, selectedType, response);
                return null;
            }
        }

        throw new HttpMediaTypeNotAcceptableException(producibleTypes);
    }

    /**
     * Determine what media types the server can produce for the given value type.
     *
     * Maps to: {@code AbstractMessageConverterMethodProcessor.getProducibleMediaTypes()}
     *
     * Two sources:
     * <ol>
     *   <li>Request attribute from @RequestMapping(produces=...) — takes priority</li>
     *   <li>Query all converters that canWrite the value type</li>
     * </ol>
     */
    @SuppressWarnings("unchecked")
    private List<MediaType> getProducibleMediaTypes(HttpServletRequest request, Class<?> valueType) {
        List<MediaType> producesFromMapping = (List<MediaType>)
                request.getAttribute(SimpleHandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
        if (producesFromMapping != null && !producesFromMapping.isEmpty()) {
            return producesFromMapping;
        }

        List<MediaType> result = new ArrayList<>();
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(valueType, null)) {
                result.addAll(converter.getSupportedMediaTypes());
            }
        }
        return result.isEmpty() ? List.of(MediaType.APPLICATION_OCTET_STREAM) : result;
    }

    /**
     * Pick the more specific of two compatible media types, carrying the quality
     * value from the acceptable (client) type.
     *
     * Maps to: {@code AbstractMessageConverterMethodProcessor.getMostSpecificMediaType()}
     */
    private MediaType getMostSpecific(MediaType acceptable, MediaType producible) {
        if (acceptable.isWildcardType() || acceptable.isWildcardSubtype()) {
            return producible.copyQualityValue(acceptable);
        }
        return acceptable;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/RequestBodyArgumentResolver.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.http.MediaType;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

import java.util.List;

/**
 * Resolves method parameters annotated with {@code @RequestBody} by deserializing
 * the request body using the appropriate {@link HttpMessageConverter}.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor}
 * (the argument resolver side)
 *
 * <h3>ch14 Enhancement:</h3>
 * Now checks the request's Content-Type header to select the right converter. Previously,
 * it picked the first converter that could read the target type regardless of media type.
 * Now it parses Content-Type and calls {@code canRead(targetType, contentType)} to ensure
 * the converter supports the request's content format.
 *
 * Simplifications:
 * <ul>
 *   <li>No HttpMediaTypeNotSupportedException (would be 415 Unsupported Media Type)</li>
 *   <li>No empty body detection</li>
 *   <li>No @RequestBody(required) handling</li>
 * </ul>
 */
public class RequestBodyArgumentResolver implements HandlerMethodArgumentResolver {

    private final List<HttpMessageConverter> messageConverters;

    public RequestBodyArgumentResolver(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestBody.class);
    }

    /**
     * Deserialize the request body into the target parameter type.
     *
     * <h3>ch14 Enhancement:</h3>
     * Now parses the Content-Type header and passes it to {@code canRead()} so converters
     * can check media type compatibility.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.readWithMessageConverters()}
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();

        // ch14: Parse the Content-Type header to determine the request's media type.
        // Maps to: AbstractMessageConverterMethodArgumentResolver line 155:
        //   MediaType contentType = inputMessage.getHeaders().getContentType();
        MediaType contentType = null;
        String contentTypeHeader = request.getContentType();
        if (contentTypeHeader != null && !contentTypeHeader.isBlank()) {
            contentType = MediaType.parseMediaType(contentTypeHeader);
        }

        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canRead(targetType, contentType)) {
                return converter.read(targetType, request);
            }
        }

        throw new IllegalStateException(
                "No HttpMessageConverter found that can read request body"
                        + " (Content-Type: " + contentTypeHeader + ")"
                        + " into type '" + targetType.getName()
                        + "' for parameter '" + parameter.getParameterName()
                        + "'. Check that a converter supporting this media type is registered.");
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.convert.SimpleConversionService;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.converter.PlainTextMessageConverter;
import com.simplespringmvc.http.ContentNegotiationManager;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;

/**
 * The adapter that bridges the DispatcherServlet's generic handler invocation
 * to our specific HandlerMethod-based invocation pipeline.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 *
 * <h3>ch14 Enhancement:</h3>
 * <ul>
 *   <li>Creates a {@link ContentNegotiationManager} for Accept header parsing</li>
 *   <li>Registers a {@link PlainTextMessageConverter} alongside JacksonMessageConverter
 *       — this makes content negotiation meaningful with two competing converters</li>
 *   <li>Passes the ContentNegotiationManager to {@link ResponseBodyReturnValueHandler}</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Single adapter instance (not a chain of adapters)</li>
 *   <li>No ModelAndViewContainer — return value handlers return ModelAndView directly</li>
 *   <li>No WebDataBinderFactory</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();
    private final List<HttpMessageConverter> messageConverters = new ArrayList<>();
    private final ConversionService conversionService;
    private final ContentNegotiationManager contentNegotiationManager;

    public SimpleHandlerAdapter() {
        // ch14: Create the ContentNegotiationManager for Accept header parsing
        contentNegotiationManager = new ContentNegotiationManager();

        // ch14: Register both JSON and plain-text converters.
        // ORDER MATTERS: JacksonMessageConverter first means JSON is the default
        // when Accept is */* (most common case). PlainTextMessageConverter handles
        // text/plain requests for String return values.
        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());

        conversionService = new SimpleConversionService();

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));

        // ch14: ResponseBodyReturnValueHandler now takes a ContentNegotiationManager
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                messageConverters, contentNegotiationManager));
        returnValueHandlers.addHandler(new ModelAndViewReturnValueHandler());
        returnValueHandlers.addHandler(new ViewNameReturnValueHandler());
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    public SimpleHandlerAdapter(List<HandlerMethodArgumentResolver> customResolvers) {
        contentNegotiationManager = new ContentNegotiationManager();

        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());

        conversionService = new SimpleConversionService();

        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));

        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                messageConverters, contentNegotiationManager));
        returnValueHandlers.addHandler(new ModelAndViewReturnValueHandler());
        returnValueHandlers.addHandler(new ViewNameReturnValueHandler());
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Object[] args = resolveArguments(handlerMethod, request);
        Object result = invokeHandlerMethod(handlerMethod, args);
        return returnValueHandlers.handleReturnValue(result, handlerMethod, request, response);
    }

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

    private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    public HandlerMethodArgumentResolverComposite getArgumentResolvers() { return argumentResolvers; }
    public HandlerMethodReturnValueHandlerComposite getReturnValueHandlers() { return returnValueHandlers; }
    public List<HttpMessageConverter> getMessageConverters() { return messageConverters; }
    public ConversionService getConversionService() { return conversionService; }
    public ContentNegotiationManager getContentNegotiationManager() { return contentNegotiationManager; }
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/RequestMapping.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Maps an HTTP request to a handler method based on URL path, HTTP method,
 * and content type constraints.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.RequestMapping}
 *
 * Can be placed on both type (class) and method level:
 * <ul>
 *   <li>Type-level: defines a base path prefix for all methods in the controller</li>
 *   <li>Method-level: defines the specific path, HTTP method, and content type
 *       constraints for this handler</li>
 * </ul>
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@link #produces()} and {@link #consumes()} attributes for content negotiation.
 * These restrict which requests a handler can serve:
 * <ul>
 *   <li>{@code consumes} — restricts by request Content-Type</li>
 *   <li>{@code produces} — restricts by client Accept header and constrains
 *       which media types content negotiation considers for the response</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No params or headers attributes</li>
 *   <li>Single path string instead of String array</li>
 *   <li>Single method string instead of RequestMethod enum array</li>
 *   <li>No @AliasFor between value() and path()</li>
 * </ul>
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestMapping {

    /**
     * The URL path this mapping applies to.
     */
    String path() default "";

    /**
     * The HTTP method to match (GET, POST, PUT, DELETE).
     * Empty string means match any HTTP method.
     */
    String method() default "";

    /**
     * Restricts the handler to requests whose Accept header is compatible
     * with the specified media types.
     *
     * Maps to: {@code RequestMapping.produces()} in the real framework.
     *
     * When set, the specified types are also stored as a request attribute
     * to constrain content negotiation for the response.
     *
     * Example: {@code @RequestMapping(produces = "application/json")}
     */
    String[] produces() default {};

    /**
     * Restricts the handler to requests whose Content-Type header matches
     * one of the specified media types.
     *
     * Maps to: {@code RequestMapping.consumes()} in the real framework.
     *
     * Example: {@code @RequestMapping(consumes = "application/json")}
     */
    String[] consumes() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/GetMapping.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "GET")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.GetMapping}
 *
 * A composed annotation meta-annotated with {@code @RequestMapping(method = "GET")}.
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@link #produces()} and {@link #consumes()} attributes for content negotiation.
 * In the real framework, these are forwarded to @RequestMapping via @AliasFor.
 * We read them directly from the composed annotation via reflection.
 *
 * Simplifications:
 * <ul>
 *   <li>Uses value() instead of @AliasFor to forward path</li>
 *   <li>No params or headers attributes</li>
 * </ul>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "GET")
public @interface GetMapping {

    /**
     * The URL path for this mapping.
     * Equivalent to {@link RequestMapping#path()}.
     */
    String value() default "";

    /**
     * Restricts by Accept header — only serve clients requesting these types.
     * Equivalent to {@link RequestMapping#produces()}.
     */
    String[] produces() default {};

    /**
     * Restricts by Content-Type — only handle requests with matching body type.
     * Equivalent to {@link RequestMapping#consumes()}.
     */
    String[] consumes() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/PostMapping.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "POST")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.PostMapping}
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@link #produces()} and {@link #consumes()} attributes for content negotiation.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "POST")
public @interface PostMapping {

    String value() default "";

    String[] produces() default {};

    String[] consumes() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/PutMapping.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "PUT")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.PutMapping}
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@link #produces()} and {@link #consumes()} attributes for content negotiation.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "PUT")
public @interface PutMapping {

    String value() default "";

    String[] produces() default {};

    String[] consumes() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/DeleteMapping.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "DELETE")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.DeleteMapping}
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@link #produces()} and {@link #consumes()} attributes for content negotiation.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "DELETE")
public @interface DeleteMapping {

    String value() default "";

    String[] produces() default {};

    String[] consumes() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/MergedAnnotationUtils.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;

/**
 * Utility for walking meta-annotation hierarchies to detect annotations that
 * are present either directly or as meta-annotations on composed annotations.
 *
 * Maps to: {@code org.springframework.core.annotation.AnnotatedElementUtils}
 * which delegates to the {@code MergedAnnotations} API internally.
 *
 * <h3>What is a meta-annotation?</h3>
 * A meta-annotation is an annotation placed on another annotation's definition.
 * For example, {@code @GetMapping} is annotated with {@code @RequestMapping(method = "GET")},
 * making {@code @RequestMapping} a meta-annotation of {@code @GetMapping}.
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. AnnotatedElementUtils.hasAnnotation() — uses SearchStrategy.TYPE_HIERARCHY
 *      to exhaustively search superclasses, interfaces, and all meta-annotations
 *   2. AnnotatedElementUtils.findMergedAnnotation() — searches + synthesizes an
 *      annotation proxy with merged attribute values via @AliasFor
 *   3. MergedAnnotations.from(element, strategy) — the low-level API that builds
 *      a tree of TypeMappedAnnotations with merged attribute views
 * </pre>
 *
 * <h3>Simplifications:</h3>
 * <ul>
 *   <li>One-level-deep meta-annotation search (enough for @GetMapping → @RequestMapping)</li>
 *   <li>No @AliasFor attribute merging — we extract attributes via reflection</li>
 *   <li>No SearchStrategy (no superclass/interface traversal)</li>
 *   <li>No annotation synthesis — we return the raw annotation instances</li>
 *   <li>No caching of resolved annotations</li>
 * </ul>
 */
public class MergedAnnotationUtils {

    /**
     * Check if an annotation type is present on the element — either directly
     * or as a meta-annotation on one of the element's annotations.
     *
     * Maps to: {@code AnnotatedElementUtils.hasAnnotation()} (line 537)
     * Real version uses find semantics (TYPE_HIERARCHY search), walking
     * superclasses, interfaces, and the full meta-annotation depth.
     * We search one level deep, which covers all standard composed annotations.
     *
     * Examples:
     * <ul>
     *   <li>{@code hasAnnotation(MyController.class, Controller.class)} → true
     *       if class has @Controller directly</li>
     *   <li>{@code hasAnnotation(MyRestController.class, Controller.class)} → true
     *       if class has @RestController (which is meta-annotated with @Controller)</li>
     * </ul>
     *
     * @param element        the class, method, or field to inspect
     * @param annotationType the annotation type to search for
     * @return true if the annotation is present directly or as a meta-annotation
     */
    public static boolean hasAnnotation(AnnotatedElement element,
                                        Class<? extends Annotation> annotationType) {
        // Direct presence check — O(1) via JDK annotation cache
        if (element.isAnnotationPresent(annotationType)) {
            return true;
        }

        // Meta-annotation check — look at each annotation on the element,
        // and check if THAT annotation is itself annotated with annotationType.
        // This is the core of the meta-annotation walking pattern.
        for (Annotation ann : element.getAnnotations()) {
            if (ann.annotationType().isAnnotationPresent(annotationType)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Find the annotation of the given type on the element — either directly
     * present or as a meta-annotation on one of the element's annotations.
     *
     * Maps to: {@code AnnotatedElementUtils.findMergedAnnotation()} (line 631)
     * Real version returns a synthesized annotation proxy with merged attributes
     * from @AliasFor declarations. We return the raw annotation instance.
     *
     * <p>When found as a meta-annotation (e.g., @RequestMapping on @GetMapping),
     * the returned instance is the meta-annotation declaration — its attributes
     * reflect what was declared on the composed annotation definition, NOT what
     * the user wrote. For example, @GetMapping("/users") → the returned
     * @RequestMapping has method="GET" but path="" (the path is on @GetMapping itself).
     *
     * @param element        the class, method, or field to inspect
     * @param annotationType the annotation type to search for
     * @return the annotation instance, or null if not found
     */
    public static <A extends Annotation> A findAnnotation(AnnotatedElement element,
                                                           Class<A> annotationType) {
        // Direct presence — return the annotation as-is
        A direct = element.getAnnotation(annotationType);
        if (direct != null) {
            return direct;
        }

        // Meta-annotation — return the meta-annotation instance from
        // whichever composed annotation carries it
        for (Annotation ann : element.getAnnotations()) {
            A meta = ann.annotationType().getAnnotation(annotationType);
            if (meta != null) {
                return meta;
            }
        }

        return null;
    }

    /**
     * Find the composed annotation on the element that carries the given
     * meta-annotation type. Returns the "carrier" annotation (e.g., @GetMapping),
     * not the meta-annotation itself (e.g., @RequestMapping).
     *
     * This is useful when you need to read attributes from the composed annotation
     * that aren't on the meta-annotation — for example, reading the path from
     * @GetMapping("/users") when searching for @RequestMapping.
     *
     * Maps to: There's no direct equivalent in the real framework because
     * the MergedAnnotation API handles attribute merging transparently.
     * We need this because we don't implement @AliasFor.
     *
     * @param element            the class, method, or field to inspect
     * @param metaAnnotationType the meta-annotation type to search for
     * @return the composed annotation carrying the meta-annotation, or null
     */
    public static Annotation findComposedAnnotation(AnnotatedElement element,
                                                     Class<? extends Annotation> metaAnnotationType) {
        // If the annotation is directly present, it's not "composed" — skip
        if (element.isAnnotationPresent(metaAnnotationType)) {
            return null;
        }

        for (Annotation ann : element.getAnnotations()) {
            if (ann.annotationType().isAnnotationPresent(metaAnnotationType)) {
                return ann;
            }
        }

        return null;
    }

    /**
     * Extract a String attribute value from an annotation via reflection.
     *
     * This is our simplified substitute for Spring's @AliasFor mechanism.
     * Instead of synthesizing annotation proxies that merge attributes,
     * we read attributes directly from whichever annotation has them.
     *
     * Maps to: {@code MergedAnnotation.getString()} which reads attribute
     * values from the merged annotation view, honoring @AliasFor declarations.
     *
     * @param annotation    the annotation to read from
     * @param attributeName the name of the attribute to read
     * @return the attribute value, or empty string if not found
     */
    public static String getStringAttribute(Annotation annotation, String attributeName) {
        try {
            Method attrMethod = annotation.annotationType().getMethod(attributeName);
            Object value = attrMethod.invoke(annotation);
            return value instanceof String s ? s : "";
        } catch (NoSuchMethodException e) {
            return "";
        } catch (ReflectiveOperationException e) {
            throw new IllegalStateException(
                    "Failed to read attribute '" + attributeName + "' from " + annotation, e);
        }
    }

    /**
     * Extract a String[] attribute value from an annotation via reflection.
     *
     * <h3>ch14 Enhancement:</h3>
     * Added to support reading {@code produces()} and {@code consumes()} attributes
     * from composed annotations like @GetMapping. These return String[] instead of
     * String, so we need a separate method.
     *
     * Maps to: {@code MergedAnnotation.getStringArray()} which reads array attribute
     * values from the merged annotation view.
     *
     * @param annotation    the annotation to read from
     * @param attributeName the name of the attribute to read
     * @return the attribute value, or empty array if not found
     */
    public static String[] getStringArrayAttribute(Annotation annotation, String attributeName) {
        try {
            Method attrMethod = annotation.annotationType().getMethod(attributeName);
            Object value = attrMethod.invoke(annotation);
            if (value instanceof String[] arr) {
                return arr;
            }
            return new String[0];
        } catch (NoSuchMethodException e) {
            return new String[0];
        } catch (ReflectiveOperationException e) {
            throw new IllegalStateException(
                    "Failed to read attribute '" + attributeName + "' from " + annotation, e);
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/RouteKey.java` [MODIFIED]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.http.MediaType;

import java.util.*;

/**
 * A composite key combining URL pattern, HTTP method, and content type conditions
 * for handler lookup.
 *
 * This has no direct equivalent in real Spring — the real framework uses
 * {@code RequestMappingInfo} (a composite of 8 request conditions) as the
 * registry key. We simplify to path pattern + method + produces/consumes.
 *
 * <h3>ch07 Enhancement:</h3>
 * RouteKey uses {@link PathPattern} instead of raw String comparison.
 *
 * <h3>ch14 Enhancement:</h3>
 * Added {@code produces} and {@code consumes} conditions for content negotiation.
 * <ul>
 *   <li>{@code consumes} — restricts by request Content-Type: the request's Content-Type
 *       must be compatible with at least one declared consume type</li>
 *   <li>{@code produces} — restricts by client Accept header: at least one declared
 *       produce type must be compatible with at least one accepted type</li>
 * </ul>
 *
 * In the real framework, these are represented by {@code ConsumesRequestCondition}
 * and {@code ProducesRequestCondition} within {@code RequestMappingInfo}.
 *
 * Note: equals/hashCode include produces/consumes, so the same path+method with
 * different produces values are different keys (allowing multiple handlers for
 * the same URL differentiated by content type).
 */
public final class RouteKey {

    private final String path;
    private final String httpMethod;
    private final PathPattern pathPattern;
    private final List<MediaType> produces;
    private final List<MediaType> consumes;

    public RouteKey(String path, String httpMethod) {
        this(path, httpMethod, new String[0], new String[0]);
    }

    /**
     * Create a RouteKey with content negotiation conditions.
     *
     * @param path       the URL pattern (e.g., "/users/{id}")
     * @param httpMethod the HTTP method (e.g., "GET")
     * @param produces   media types this handler can produce (empty = any)
     * @param consumes   media types this handler can consume (empty = any)
     */
    public RouteKey(String path, String httpMethod, String[] produces, String[] consumes) {
        this.path = normalizePath(path);
        this.httpMethod = httpMethod != null ? httpMethod.toUpperCase() : "";
        this.pathPattern = new PathPattern(this.path);
        this.produces = parseMediaTypes(produces);
        this.consumes = parseMediaTypes(consumes);
    }

    private static List<MediaType> parseMediaTypes(String[] types) {
        if (types == null || types.length == 0) {
            return List.of();
        }
        List<MediaType> result = new ArrayList<>();
        for (String type : types) {
            result.add(MediaType.parseMediaType(type));
        }
        return Collections.unmodifiableList(result);
    }

    public String getPath() {
        return path;
    }

    public String getHttpMethod() {
        return httpMethod;
    }

    /**
     * The media types this handler can produce.
     * Empty list means no restriction (handler can produce any type).
     */
    public List<MediaType> getProduces() {
        return produces;
    }

    /**
     * The media types this handler can consume.
     * Empty list means no restriction (handler accepts any Content-Type).
     */
    public List<MediaType> getConsumes() {
        return consumes;
    }

    /**
     * Checks if this RouteKey matches a request path and HTTP method.
     * Does NOT check produces/consumes — use the overload with headers for that.
     */
    public boolean matches(String requestPath, String requestMethod) {
        String normalizedRequestPath = normalizePath(requestPath);
        if (!pathPattern.matches(normalizedRequestPath)) {
            return false;
        }
        if (this.httpMethod.isEmpty()) {
            return true;
        }
        return this.httpMethod.equalsIgnoreCase(requestMethod);
    }

    /**
     * Match the request and extract path variable values.
     *
     * Maps to: {@code PathPattern.matchAndExtract(PathContainer)}
     *
     * @return extracted path variables, or null if no match
     */
    public Map<String, String> matchAndExtract(String requestPath, String requestMethod) {
        String normalizedRequestPath = normalizePath(requestPath);

        if (!this.httpMethod.isEmpty() && !this.httpMethod.equalsIgnoreCase(requestMethod)) {
            return null;
        }

        return pathPattern.matchAndExtract(normalizedRequestPath);
    }

    // ─── Content negotiation matching ────────────────────────────

    /**
     * Check if the request's Content-Type is compatible with this handler's
     * {@code consumes} condition.
     *
     * Maps to: {@code ConsumesRequestCondition.getMatchingCondition()}
     *
     * <h3>Algorithm:</h3>
     * <ol>
     *   <li>If no consumes declared, match any Content-Type</li>
     *   <li>Parse the request Content-Type</li>
     *   <li>Check if any declared consumes type includes the request Content-Type</li>
     * </ol>
     *
     * @param contentType the request's Content-Type header value (may be null)
     * @return true if the request's Content-Type is acceptable
     */
    public boolean matchesConsumes(String contentType) {
        if (consumes.isEmpty()) {
            return true; // no restriction
        }
        if (contentType == null || contentType.isBlank()) {
            // No Content-Type on request; consumes is declared → no match.
            // Real framework defaults to application/octet-stream.
            return false;
        }
        MediaType requestType = MediaType.parseMediaType(contentType);
        for (MediaType consumed : consumes) {
            if (consumed.includes(requestType)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Check if the client's Accept header is compatible with this handler's
     * {@code produces} condition.
     *
     * Maps to: {@code ProducesRequestCondition.getMatchingCondition()}
     *
     * <h3>Algorithm:</h3>
     * <ol>
     *   <li>If no produces declared, match any Accept</li>
     *   <li>Parse the Accept header into media types</li>
     *   <li>Check if any declared produces type is compatible with any accepted type</li>
     *   <li>If *&#47;* is in accepted types, always match</li>
     * </ol>
     *
     * @param acceptHeader the request's Accept header value (may be null)
     * @return true if the client accepts at least one of the handler's produced types
     */
    public boolean matchesProduces(String acceptHeader) {
        if (produces.isEmpty()) {
            return true; // no restriction
        }
        List<MediaType> acceptedTypes;
        if (acceptHeader == null || acceptHeader.isBlank()) {
            acceptedTypes = List.of(MediaType.ALL);
        } else {
            acceptedTypes = MediaType.parseMediaTypes(acceptHeader);
        }
        for (MediaType produced : produces) {
            for (MediaType accepted : acceptedTypes) {
                if (produced.isCompatibleWith(accepted)) {
                    return true;
                }
            }
        }
        return false;
    }

    // ─── Normalization ──────────────────────────────────────────

    private static String normalizePath(String path) {
        if (path == null || path.isEmpty()) {
            return "/";
        }
        if (!path.startsWith("/")) {
            path = "/" + path;
        }
        if (path.length() > 1 && path.endsWith("/")) {
            path = path.substring(0, path.length() - 1);
        }
        return path;
    }

    // ─── Object methods ─────────────────────────────────────────

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof RouteKey that)) return false;
        return path.equals(that.path) && httpMethod.equals(that.httpMethod)
                && produces.equals(that.produces) && consumes.equals(that.consumes);
    }

    @Override
    public int hashCode() {
        return Objects.hash(path, httpMethod, produces, consumes);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        if (httpMethod.isEmpty()) {
            sb.append("ALL ");
        } else {
            sb.append(httpMethod).append(' ');
        }
        sb.append(path);
        if (!produces.isEmpty()) {
            sb.append(" produces=").append(produces);
        }
        if (!consumes.isEmpty()) {
            sb.append(" consumes=").append(consumes);
        }
        return sb.toString();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java` [MODIFIED]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Scans all beans in the container for @Controller classes, discovers methods
 * annotated with @RequestMapping, and builds a registry that maps
 * (URL pattern + HTTP method + content type conditions) → HandlerMethod.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping}
 *
 * <h3>ch14 Enhancement:</h3>
 * Handler detection now reads {@code produces} and {@code consumes} attributes from
 * @RequestMapping (and composed annotations). During lookup, requests are matched
 * against these conditions by checking the Accept and Content-Type headers. When
 * {@code produces} is specified, the declared types are stored as a request attribute
 * for the content negotiation algorithm in ResponseBodyReturnValueHandler.
 *
 * Simplifications:
 * <ul>
 *   <li>No MappingRegistry inner class with ReadWriteLock</li>
 *   <li>No RequestMappingInfo composite — RouteKey handles path + method + produces/consumes</li>
 *   <li>No direct-path optimization (we scan all entries on every request)</li>
 * </ul>
 */
public class SimpleHandlerMapping {

    /**
     * Request attribute key for extracted path variables.
     * Maps to: {@code HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE}
     */
    public static final String PATH_VARIABLES_ATTRIBUTE =
            "com.simplespringmvc.mapping.SimpleHandlerMapping.pathVariables";

    /**
     * Request attribute key for the producible media types declared by the matched
     * handler's {@code produces} condition. When present, the content negotiation
     * algorithm uses these types instead of querying all converters.
     *
     * Maps to: {@code HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE}
     *
     * <h3>ch14 Enhancement:</h3>
     * Set when the matched handler has a non-empty {@code produces} attribute.
     * This constrains the set of media types that content negotiation considers,
     * preventing a converter from producing a type the handler didn't declare.
     */
    public static final String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE =
            "com.simplespringmvc.mapping.SimpleHandlerMapping.producibleMediaTypes";

    private final Map<RouteKey, HandlerMethod> registry = new LinkedHashMap<>();

    /**
     * Scan all beans in the container and register handler methods.
     * Maps to: {@code AbstractHandlerMethodMapping.initHandlerMethods()}
     */
    public void init(BeanContainer beanContainer) {
        for (String beanName : beanContainer.getBeanNames()) {
            Object bean = beanContainer.getBean(beanName);
            if (isHandler(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    /**
     * Check if a bean type is a handler (annotated with @Controller or @RestController).
     * Maps to: {@code RequestMappingHandlerMapping.isHandler()}
     */
    private boolean isHandler(Class<?> beanType) {
        return MergedAnnotationUtils.hasAnnotation(beanType, Controller.class);
    }

    /**
     * Discover all @RequestMapping methods on a handler bean and register them.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.detectHandlerMethods()}
     *
     * <h3>ch14 Enhancement:</h3>
     * Now reads {@code produces} and {@code consumes} attributes from @RequestMapping
     * and composed annotations. These are stored in the RouteKey for matching during
     * handler lookup.
     */
    private void detectHandlerMethods(Object bean) {
        Class<?> beanType = bean.getClass();

        // Check for type-level @RequestMapping (base path prefix)
        String basePath = "";
        RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
        if (typeMapping != null) {
            basePath = typeMapping.path();
        }

        // Scan all declared methods for @RequestMapping (direct or composed)
        for (Method method : beanType.getDeclaredMethods()) {
            String path;
            String httpMethod;
            String[] produces;
            String[] consumes;

            // Try direct @RequestMapping first
            RequestMapping directMapping = method.getAnnotation(RequestMapping.class);
            if (directMapping != null) {
                path = directMapping.path();
                httpMethod = directMapping.method();
                produces = directMapping.produces();
                consumes = directMapping.consumes();
            } else {
                // ch10: Look for composed annotations meta-annotated with @RequestMapping
                Annotation composed = MergedAnnotationUtils.findComposedAnnotation(
                        method, RequestMapping.class);
                if (composed == null) {
                    continue;
                }

                // HTTP method from the @RequestMapping meta-annotation
                RequestMapping metaMapping = composed.annotationType()
                        .getAnnotation(RequestMapping.class);
                httpMethod = metaMapping.method();

                // Path from the composed annotation's value() attribute
                path = MergedAnnotationUtils.getStringAttribute(composed, "value");

                // ch14: produces/consumes from the composed annotation
                produces = MergedAnnotationUtils.getStringArrayAttribute(composed, "produces");
                consumes = MergedAnnotationUtils.getStringArrayAttribute(composed, "consumes");
            }

            // Combine type-level base path + method-level path
            String fullPath = combinePaths(basePath, path);

            method.setAccessible(true);

            RouteKey routeKey = new RouteKey(fullPath, httpMethod, produces, consumes);
            HandlerMethod handlerMethod = new HandlerMethod(bean, method);

            // Check for duplicate mappings
            if (registry.containsKey(routeKey)) {
                HandlerMethod existing = registry.get(routeKey);
                throw new IllegalStateException(
                        "Ambiguous mapping: " + routeKey + " is already mapped to " + existing
                                + ". Cannot map " + handlerMethod);
            }

            registry.put(routeKey, handlerMethod);
        }
    }

    /**
     * Look up the handler method for a given request path and HTTP method.
     * Does NOT check produces/consumes conditions.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.lookupHandlerMethod()}
     *
     * @return the matching HandlerMethod, or null if no handler found
     */
    public HandlerMethod lookupHandler(String requestPath, String requestMethod) {
        for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
            RouteKey key = entry.getKey();
            if (key.matches(requestPath, requestMethod)) {
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Look up the handler method for a given HTTP request, checking path variables,
     * produces, and consumes conditions.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.getHandlerInternal()}
     *
     * <h3>ch14 Enhancement:</h3>
     * Now checks {@code consumes} against the request's Content-Type and
     * {@code produces} against the request's Accept header. When a handler with
     * non-empty {@code produces} matches, stores the producible types as a
     * request attribute for the content negotiation algorithm.
     *
     * @return the matching HandlerMethod, or null if no handler found
     */
    public HandlerMethod lookupHandler(HttpServletRequest request) {
        String requestPath = request.getRequestURI();
        String requestMethod = request.getMethod();

        for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
            RouteKey key = entry.getKey();
            Map<String, String> pathVariables = key.matchAndExtract(requestPath, requestMethod);
            if (pathVariables != null) {
                // ch14: Check consumes condition against request Content-Type
                if (!key.matchesConsumes(request.getContentType())) {
                    continue;
                }

                // ch14: Check produces condition against request Accept header
                if (!key.matchesProduces(request.getHeader("Accept"))) {
                    continue;
                }

                // Store extracted path variables
                request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);

                // ch14: Store producible media types for content negotiation.
                // When produces is declared, the ResponseBodyReturnValueHandler
                // uses these types instead of querying all converters.
                // Maps to: AbstractHandlerMethodMapping.handleMatch() which calls
                // request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE, ...)
                if (!key.getProduces().isEmpty()) {
                    request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE,
                            new ArrayList<>(key.getProduces()));
                }

                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Returns an unmodifiable view of all registered mappings.
     */
    public Map<RouteKey, HandlerMethod> getRegisteredMappings() {
        return Map.copyOf(registry);
    }

    private String combinePaths(String basePath, String methodPath) {
        if (basePath == null || basePath.isEmpty()) {
            return methodPath;
        }
        if (methodPath == null || methodPath.isEmpty()) {
            return basePath;
        }
        String base = basePath.endsWith("/") ? basePath.substring(0, basePath.length() - 1) : basePath;
        String method = methodPath.startsWith("/") ? methodPath : "/" + methodPath;
        return base + method;
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
import com.simplespringmvc.http.HttpMediaTypeNotAcceptableException;
import com.simplespringmvc.interceptor.HandlerExecutionChain;
import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.view.ModelAndView;
import com.simplespringmvc.view.View;
import com.simplespringmvc.view.ViewResolver;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * The front controller that receives ALL HTTP requests and dispatches them
 * to the appropriate handler.
 *
 * Maps to: {@code org.springframework.web.servlet.DispatcherServlet}
 *
 * The real DispatcherServlet sits at the bottom of a three-class hierarchy:
 * <pre>
 *   HttpServletBean          → maps init-params to bean properties
 *     └── FrameworkServlet   → manages WebApplicationContext, request lifecycle
 *           └── DispatcherServlet → the actual dispatch logic
 * </pre>
 *
 * We collapse all three layers into one class. The key method is {@link #doDispatch}
 * which is the central routing point — every future feature adds a step here:
 * <ul>
 *   <li>ch03: handler mapping lookup</li>
 *   <li>ch04: handler adapter invocation</li>
 *   <li>ch11: interceptor pre/post processing</li>
 *   <li>ch12: exception handler resolution</li>
 *   <li>ch13: view resolution and rendering ✓</li>
 * </ul>
 *
 * <h3>ch13 Enhancement:</h3>
 * The dispatch pipeline now includes view resolution and rendering:
 * <pre>
 *   1. getHandler(request) → HandlerExecutionChain
 *   2. chain.applyPreHandle()
 *   3. ha.handle() → returns ModelAndView (null if response handled directly)
 *   4. chain.applyPostHandle()
 *   5. chain.triggerAfterCompletion() (always)
 *   6. If exception → processHandlerException()
 *   7. If ModelAndView non-null → render(mv, request, response)
 * </pre>
 *
 * The render() step resolves the view name via ViewResolvers and delegates
 * to the View object to write the response. If no ViewResolver resolves
 * the view name, the view name is written directly as text/plain (a
 * simplification that preserves backward compatibility with previous chapters).
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No locale/theme resolution (added in ch20)</li>
 *   <li>No flash map management (added in ch18)</li>
 *   <li>Falls back to text/plain when view can't be resolved (real framework throws)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    /**
     * Global interceptors applied to all handlers in order.
     *
     * Maps to: {@code AbstractHandlerMapping.adaptedInterceptors} (line 123)
     */
    private final List<HandlerInterceptor> interceptors = new ArrayList<>();

    /**
     * Exception resolvers consulted when a handler throws an exception.
     *
     * Maps to: {@code DispatcherServlet.handlerExceptionResolvers} (line 191)
     */
    private final List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<>();

    /**
     * ViewResolvers that translate logical view names into View objects.
     *
     * Maps to: {@code DispatcherServlet.viewResolvers} (line 197)
     * Real version is populated from ViewResolver beans in the context
     * (or from DispatcherServlet.properties defaults). Iterated in order —
     * first resolver that returns a non-null View wins.
     *
     * <h3>ch13 Enhancement:</h3>
     * Added to support view resolution and rendering.
     */
    private final List<ViewResolver> viewResolvers = new ArrayList<>();

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Called by Tomcat once the Servlet is registered. Initializes strategy
     * components (handler mappings, adapters, etc.) from the bean container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     */
    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    /**
     * Initialize all strategy beans from the container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * Real version initializes 9 strategy types. We initialize them
     * incrementally as features are added.
     */
    protected void initStrategies() {
        initHandlerMapping();
        initHandlerAdapter();
        initExceptionResolvers();
        // ch13: ViewResolvers are added programmatically via addViewResolver()
        // rather than initialized from the container. The real framework finds
        // ViewResolver beans in the ApplicationContext.
    }

    private void initHandlerMapping() {
        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(beanContainer);
    }

    private void initHandlerAdapter() {
        handlerAdapter = new SimpleHandlerAdapter();
    }

    private void initExceptionResolvers() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(beanContainer);
        exceptionResolvers.add(resolver);
    }

    /**
     * Override service() to route ALL HTTP methods through doDispatch().
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870)
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        }
    }

    /**
     * The central dispatch method — the heart of the framework.
     *
     * Maps to: {@code DispatcherServlet.doDispatch()} (line 935)
     *
     * <h3>ch13 Enhancement:</h3>
     * The dispatch flow now includes view resolution:
     * <pre>
     *   1. mappedHandler = getHandler(request)
     *   2. mappedHandler.applyPreHandle()
     *   3. mv = ha.handle(request, response, handler) ← now returns ModelAndView
     *   4. mappedHandler.applyPostHandle()
     *   5. mappedHandler.triggerAfterCompletion() (always)
     *   6. processHandlerException() if exception
     *   7. render(mv, request, response) if mv != null  ← NEW
     * </pre>
     *
     * The key change: handle() now returns a ModelAndView. If it's non-null,
     * we resolve the view name and render. If it's null, the response was
     * already written (e.g., by @ResponseBody).
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerExecutionChain mappedHandler = null;
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // Step 1: Look up the handler + interceptor chain for this request
            mappedHandler = getHandler(request);

            if (mappedHandler == null) {
                response.sendError(HttpServletResponse.SC_NOT_FOUND,
                        "No handler found for " + request.getMethod() + " " + request.getRequestURI());
                return;
            }

            // Step 2: Apply interceptor preHandle — forward order
            if (!mappedHandler.applyPreHandle(request, response)) {
                return;
            }

            // Step 3: Find a HandlerAdapter and invoke the handler.
            // ch13: handle() now returns ModelAndView instead of void.
            // Maps to: DispatcherServlet.doDispatch() line 963:
            //   mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
            mv = adapter.handle(request, response, mappedHandler.getHandler());

            // Step 4: Apply interceptor postHandle — reverse order
            mappedHandler.applyPostHandle(request, response);

        } catch (HttpMediaTypeNotAcceptableException ex) {
            // ch14: The client's Accept header doesn't match any type the server can produce.
            // Maps to: DefaultHandlerExceptionResolver mapping this to 406 Not Acceptable.
            response.sendError(HttpServletResponse.SC_NOT_ACCEPTABLE, ex.getMessage());
            if (mappedHandler != null) {
                mappedHandler.triggerAfterCompletion(request, response, null);
            }
            return;
        } catch (Exception ex) {
            dispatchException = ex;
        }

        // Step 5: afterCompletion — always runs
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, dispatchException);
        }

        // Step 6: Exception handling
        if (dispatchException != null) {
            Object handler = (mappedHandler != null) ? mappedHandler.getHandler() : null;
            if (!processHandlerException(request, response, handler, dispatchException)) {
                throw dispatchException;
            }
            // Exception was handled — don't render a view
            return;
        }

        // Step 7 (ch13): View resolution and rendering.
        // Maps to: DispatcherServlet.processDispatchResult() (line 1022)
        //        → render() (line 1267)
        //
        // If the handler returned a ModelAndView, resolve the view name
        // and render. If handle() returned null, the response was already
        // written (e.g., @ResponseBody) — nothing more to do.
        if (mv != null && mv.getViewName() != null) {
            render(mv, request, response);
        }
    }

    /**
     * Resolve the view name and render the view with model data.
     *
     * Maps to: {@code DispatcherServlet.render()} (line 1267)
     *
     * The real render() method:
     * <pre>
     *   1. Determine locale (via LocaleResolver)
     *   2. Resolve view name → View object (via ViewResolver chain)
     *   3. Set response status if configured
     *   4. Call view.render(model, request, response)
     * </pre>
     *
     * We skip step 1 (no locale support yet — ch20) and step 3 (no status code
     * on ModelAndView). If no ViewResolver resolves the view name, we fall back
     * to writing the view name as text/plain. This is a simplification — the
     * real framework throws a {@code ServletException} when a view can't be
     * resolved.
     *
     * @param mv       the ModelAndView holding view name and model data
     * @param request  current HTTP request
     * @param response current HTTP response
     */
    private void render(ModelAndView mv, HttpServletRequest request,
                        HttpServletResponse response) throws Exception {
        String viewName = mv.getViewName();

        // Try to resolve the view name via the ViewResolver chain.
        // Maps to: DispatcherServlet.resolveViewName() (line 1339)
        View view = resolveViewName(viewName);

        if (view != null) {
            // Delegate to the View object for rendering.
            // Maps to: DispatcherServlet.render() (line 1305):
            //   view.render(mv.getModelInternal(), request, response)
            view.render(mv.getModel(), request, response);
        } else {
            // Fallback: write the view name directly as text/plain.
            // This is a simplification — the real framework throws:
            //   throw new ServletException("Could not resolve view with name '" + viewName + "'");
            // We fall back gracefully to preserve backward compatibility with
            // String-returning controllers that don't have templates yet.
            response.setContentType("text/plain");
            response.setCharacterEncoding("UTF-8");
            response.getWriter().write(viewName);
            response.getWriter().flush();
        }
    }

    /**
     * Resolve a view name to a View object by iterating the ViewResolver chain.
     *
     * Maps to: {@code DispatcherServlet.resolveViewName()} (line 1339)
     * → iterates {@code viewResolvers} and returns the first non-null result.
     *
     * @param viewName the logical view name to resolve
     * @return the View object, or null if no resolver can resolve it
     */
    private View resolveViewName(String viewName) throws Exception {
        for (ViewResolver resolver : viewResolvers) {
            View view = resolver.resolveViewName(viewName);
            if (view != null) {
                return view;
            }
        }
        return null;
    }

    /**
     * Iterate exception resolvers to handle the given exception.
     *
     * Maps to: {@code DispatcherServlet.processHandlerException()} (line 1207)
     */
    private boolean processHandlerException(HttpServletRequest request, HttpServletResponse response,
                                            Object handler, Exception ex) {
        for (HandlerExceptionResolver resolver : exceptionResolvers) {
            if (resolver.resolveException(request, response, handler, ex)) {
                return true;
            }
        }
        return false;
    }

    private HandlerExecutionChain getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        HandlerMethod handler = handlerMapping.lookupHandler(request);
        if (handler == null) {
            return null;
        }
        return new HandlerExecutionChain(handler, interceptors);
    }

    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
    }

    // ─── ViewResolver registration ──────────────────────────────────

    /**
     * Register a ViewResolver to resolve view names to View objects.
     *
     * Maps to: In the real framework, ViewResolvers are registered as beans in
     * the ApplicationContext and auto-detected by DispatcherServlet.initViewResolvers().
     * We use programmatic registration for simplicity.
     *
     * <h3>ch13 Enhancement:</h3>
     * Added to support view resolution.
     *
     * @param viewResolver the resolver to add
     */
    public void addViewResolver(ViewResolver viewResolver) {
        this.viewResolvers.add(viewResolver);
    }

    /**
     * Returns the registered ViewResolvers (for testing/inspection).
     */
    public List<ViewResolver> getViewResolvers() {
        return List.copyOf(viewResolvers);
    }

    // ─── Interceptor registration ────────────────────────────────────

    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptors.add(interceptor);
    }

    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptors);
    }

    // ─── Accessors ───────────────────────────────────────────────────

    public BeanContainer getBeanContainer() {
        return beanContainer;
    }

    public SimpleHandlerMapping getHandlerMapping() {
        return handlerMapping;
    }

    public HandlerAdapter getHandlerAdapter() {
        return handlerAdapter;
    }

    public List<HandlerExceptionResolver> getExceptionResolvers() {
        return List.copyOf(exceptionResolvers);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/exception/ExceptionHandlerExceptionResolver.java` [MODIFIED]

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

        // ch14: ResponseBodyReturnValueHandler now requires a ContentNegotiationManager
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                converters, new com.simplespringmvc.http.ContentNegotiationManager()));
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
            // ch13: handleReturnValue() now returns ModelAndView, but for exception
            // handlers we always write directly (ResponseBody → JSON, String → text/plain).
            // We intentionally ignore the returned ModelAndView.
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

### Test Code

#### File: `src/test/java/com/simplespringmvc/http/MediaTypeTest.java` [NEW]

```java
package com.simplespringmvc.http;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link MediaType}.
 */
class MediaTypeTest {

    // ─── Parsing ──────────────────────────────────────────────────

    @Nested
    @DisplayName("parseMediaType()")
    class ParseMediaType {

        @Test
        @DisplayName("should parse simple type/subtype")
        void shouldParseSimpleMediaType() {
            MediaType type = MediaType.parseMediaType("application/json");

            assertThat(type.getType()).isEqualTo("application");
            assertThat(type.getSubtype()).isEqualTo("json");
            assertThat(type.getParameters()).isEmpty();
        }

        @Test
        @DisplayName("should parse type with quality parameter")
        void shouldParseWithQualityParameter() {
            MediaType type = MediaType.parseMediaType("text/html;q=0.9");

            assertThat(type.getType()).isEqualTo("text");
            assertThat(type.getSubtype()).isEqualTo("html");
            assertThat(type.getQualityValue()).isEqualTo(0.9);
        }

        @Test
        @DisplayName("should parse type with multiple parameters")
        void shouldParseWithMultipleParameters() {
            MediaType type = MediaType.parseMediaType("text/html;charset=UTF-8;q=0.8");

            assertThat(type.getType()).isEqualTo("text");
            assertThat(type.getSubtype()).isEqualTo("html");
            assertThat(type.getParameters()).containsEntry("charset", "UTF-8");
            assertThat(type.getQualityValue()).isEqualTo(0.8);
        }

        @Test
        @DisplayName("should parse wildcard type")
        void shouldParseWildcardType() {
            MediaType type = MediaType.parseMediaType("*/*");

            assertThat(type.isWildcardType()).isTrue();
            assertThat(type.isWildcardSubtype()).isTrue();
            assertThat(type.isConcrete()).isFalse();
        }

        @Test
        @DisplayName("should parse wildcard subtype")
        void shouldParseWildcardSubtype() {
            MediaType type = MediaType.parseMediaType("text/*");

            assertThat(type.isWildcardType()).isFalse();
            assertThat(type.isWildcardSubtype()).isTrue();
            assertThat(type.isConcrete()).isFalse();
        }

        @Test
        @DisplayName("should normalize to lowercase")
        void shouldNormalizeToLowercase() {
            MediaType type = MediaType.parseMediaType("Application/JSON");

            assertThat(type.getType()).isEqualTo("application");
            assertThat(type.getSubtype()).isEqualTo("json");
        }

        @Test
        @DisplayName("should default quality to 1.0 when not specified")
        void shouldDefaultQualityToOne() {
            MediaType type = MediaType.parseMediaType("application/json");

            assertThat(type.getQualityValue()).isEqualTo(1.0);
        }

        @Test
        @DisplayName("should throw on blank input")
        void shouldThrow_WhenInputIsBlank() {
            assertThatThrownBy(() -> MediaType.parseMediaType(""))
                    .isInstanceOf(IllegalArgumentException.class);
        }
    }

    // ─── parseMediaTypes() ────────────────────────────────────────

    @Nested
    @DisplayName("parseMediaTypes()")
    class ParseMediaTypes {

        @Test
        @DisplayName("should parse comma-separated Accept header")
        void shouldParseCommaSeparatedTypes() {
            List<MediaType> types = MediaType.parseMediaTypes(
                    "text/html, application/json;q=0.9, */*;q=0.1");

            assertThat(types).hasSize(3);
            assertThat(types.get(0)).isEqualTo(MediaType.TEXT_HTML);
            assertThat(types.get(1)).isEqualTo(MediaType.APPLICATION_JSON);
            assertThat(types.get(2)).isEqualTo(MediaType.ALL);
            assertThat(types.get(1).getQualityValue()).isEqualTo(0.9);
            assertThat(types.get(2).getQualityValue()).isEqualTo(0.1);
        }

        @Test
        @DisplayName("should return [*/*] for null input")
        void shouldReturnAll_WhenNull() {
            List<MediaType> types = MediaType.parseMediaTypes(null);

            assertThat(types).containsExactly(MediaType.ALL);
        }

        @Test
        @DisplayName("should return [*/*] for blank input")
        void shouldReturnAll_WhenBlank() {
            List<MediaType> types = MediaType.parseMediaTypes("  ");

            assertThat(types).containsExactly(MediaType.ALL);
        }
    }

    // ─── includes() ───────────────────────────────────────────────

    @Nested
    @DisplayName("includes()")
    class Includes {

        @Test
        @DisplayName("*/* should include everything")
        void wildcardShouldIncludeEverything() {
            assertThat(MediaType.ALL.includes(MediaType.APPLICATION_JSON)).isTrue();
            assertThat(MediaType.ALL.includes(MediaType.TEXT_PLAIN)).isTrue();
            assertThat(MediaType.ALL.includes(MediaType.TEXT_HTML)).isTrue();
        }

        @Test
        @DisplayName("text/* should include text/plain")
        void wildcardSubtypeShouldIncludeConcreteSubtype() {
            MediaType textWildcard = new MediaType("text", "*");

            assertThat(textWildcard.includes(MediaType.TEXT_PLAIN)).isTrue();
            assertThat(textWildcard.includes(MediaType.TEXT_HTML)).isTrue();
        }

        @Test
        @DisplayName("text/* should NOT include application/json")
        void wildcardSubtypeShouldNotIncludeDifferentType() {
            MediaType textWildcard = new MediaType("text", "*");

            assertThat(textWildcard.includes(MediaType.APPLICATION_JSON)).isFalse();
        }

        @Test
        @DisplayName("application/json should NOT include */*")
        void concreteShouldNotIncludeWildcard() {
            assertThat(MediaType.APPLICATION_JSON.includes(MediaType.ALL)).isFalse();
        }

        @Test
        @DisplayName("application/json should include application/json")
        void sameShouldIncludeSame() {
            assertThat(MediaType.APPLICATION_JSON.includes(MediaType.APPLICATION_JSON)).isTrue();
        }
    }

    // ─── isCompatibleWith() ───────────────────────────────────────

    @Nested
    @DisplayName("isCompatibleWith()")
    class IsCompatibleWith {

        @Test
        @DisplayName("should be symmetric for wildcards")
        void shouldBeSymmetricForWildcards() {
            assertThat(MediaType.APPLICATION_JSON.isCompatibleWith(MediaType.ALL)).isTrue();
            assertThat(MediaType.ALL.isCompatibleWith(MediaType.APPLICATION_JSON)).isTrue();
        }

        @Test
        @DisplayName("same type should be compatible")
        void sameTypeShouldBeCompatible() {
            assertThat(MediaType.APPLICATION_JSON.isCompatibleWith(MediaType.APPLICATION_JSON)).isTrue();
        }

        @Test
        @DisplayName("different types should NOT be compatible")
        void differentTypesShouldNotBeCompatible() {
            assertThat(MediaType.APPLICATION_JSON.isCompatibleWith(MediaType.TEXT_PLAIN)).isFalse();
        }

        @Test
        @DisplayName("wildcard subtype should be compatible with concrete subtype")
        void wildcardSubtypeShouldBeCompatible() {
            MediaType textWildcard = new MediaType("text", "*");

            assertThat(textWildcard.isCompatibleWith(MediaType.TEXT_PLAIN)).isTrue();
            assertThat(MediaType.TEXT_PLAIN.isCompatibleWith(textWildcard)).isTrue();
        }
    }

    // ─── Quality value operations ─────────────────────────────────

    @Nested
    @DisplayName("quality value operations")
    class QualityValue {

        @Test
        @DisplayName("should copy quality value from another type")
        void shouldCopyQualityValue() {
            MediaType source = MediaType.parseMediaType("text/html;q=0.7");
            MediaType target = MediaType.APPLICATION_JSON;

            MediaType result = target.copyQualityValue(source);

            assertThat(result.getQualityValue()).isEqualTo(0.7);
            assertThat(result.getType()).isEqualTo("application");
            assertThat(result.getSubtype()).isEqualTo("json");
        }

        @Test
        @DisplayName("should remove quality value")
        void shouldRemoveQualityValue() {
            MediaType withQ = MediaType.parseMediaType("application/json;q=0.8");

            MediaType result = withQ.removeQualityValue();

            assertThat(result.getParameters()).doesNotContainKey("q");
            assertThat(result.getType()).isEqualTo("application");
        }

        @Test
        @DisplayName("removeQualityValue should return same instance when no quality")
        void shouldReturnSame_WhenNoQuality() {
            MediaType noQ = MediaType.APPLICATION_JSON;

            assertThat(noQ.removeQualityValue()).isSameAs(noQ);
        }
    }

    // ─── Sorting ──────────────────────────────────────────────────

    @Nested
    @DisplayName("sortBySpecificityAndQuality()")
    class Sort {

        @Test
        @DisplayName("should sort by quality (highest first)")
        void shouldSortByQuality() {
            List<MediaType> types = new ArrayList<>(List.of(
                    MediaType.parseMediaType("text/html;q=0.5"),
                    MediaType.parseMediaType("application/json;q=0.9"),
                    MediaType.parseMediaType("text/plain;q=1.0")
            ));

            MediaType.sortBySpecificityAndQuality(types);

            assertThat(types.get(0)).isEqualTo(MediaType.TEXT_PLAIN);
            assertThat(types.get(1)).isEqualTo(MediaType.APPLICATION_JSON);
            assertThat(types.get(2)).isEqualTo(MediaType.TEXT_HTML);
        }

        @Test
        @DisplayName("should sort by specificity when quality is equal")
        void shouldSortBySpecificity_WhenQualityEqual() {
            List<MediaType> types = new ArrayList<>(List.of(
                    MediaType.ALL,
                    new MediaType("text", "*"),
                    MediaType.TEXT_PLAIN
            ));

            MediaType.sortBySpecificityAndQuality(types);

            assertThat(types.get(0)).isEqualTo(MediaType.TEXT_PLAIN); // concrete
            assertThat(types.get(1)).isEqualTo(new MediaType("text", "*")); // wildcard subtype
            assertThat(types.get(2)).isEqualTo(MediaType.ALL); // full wildcard
        }
    }

    // ─── toString() ───────────────────────────────────────────────

    @Nested
    @DisplayName("toString()")
    class ToStringMethod {

        @Test
        @DisplayName("should produce standard format")
        void shouldProduceStandardFormat() {
            assertThat(MediaType.APPLICATION_JSON.toString()).isEqualTo("application/json");
            assertThat(MediaType.TEXT_PLAIN.toString()).isEqualTo("text/plain");
            assertThat(MediaType.ALL.toString()).isEqualTo("*/*");
        }

        @Test
        @DisplayName("should include parameters")
        void shouldIncludeParameters() {
            MediaType withParams = MediaType.parseMediaType("text/html;charset=UTF-8");

            assertThat(withParams.toString()).isEqualTo("text/html;charset=UTF-8");
        }
    }

    // ─── equals()/hashCode() ──────────────────────────────────────

    @Nested
    @DisplayName("equals() and hashCode()")
    class Equality {

        @Test
        @DisplayName("should be equal for same type/subtype regardless of quality")
        void shouldBeEqual_WhenSameTypeAndSubtype() {
            MediaType a = MediaType.parseMediaType("application/json");
            MediaType b = MediaType.parseMediaType("application/json;q=0.5");

            assertThat(a).isEqualTo(b);
            assertThat(a.hashCode()).isEqualTo(b.hashCode());
        }

        @Test
        @DisplayName("should not be equal for different types")
        void shouldNotBeEqual_WhenDifferentTypes() {
            assertThat(MediaType.APPLICATION_JSON).isNotEqualTo(MediaType.TEXT_PLAIN);
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/http/ContentNegotiationManagerTest.java` [NEW]

```java
package com.simplespringmvc.http;

import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

/**
 * Unit tests for {@link ContentNegotiationManager}.
 */
class ContentNegotiationManagerTest {

    private ContentNegotiationManager manager;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        manager = new ContentNegotiationManager();
        request = mock(HttpServletRequest.class);
    }

    @Nested
    @DisplayName("resolveMediaTypes()")
    class ResolveMediaTypes {

        @Test
        @DisplayName("should return [*/*] when no Accept header")
        void shouldReturnAll_WhenNoAcceptHeader() {
            when(request.getHeader("Accept")).thenReturn(null);

            List<MediaType> types = manager.resolveMediaTypes(request);

            assertThat(types).containsExactly(MediaType.ALL);
        }

        @Test
        @DisplayName("should return [*/*] when Accept header is blank")
        void shouldReturnAll_WhenAcceptHeaderBlank() {
            when(request.getHeader("Accept")).thenReturn("  ");

            List<MediaType> types = manager.resolveMediaTypes(request);

            assertThat(types).containsExactly(MediaType.ALL);
        }

        @Test
        @DisplayName("should parse single Accept type")
        void shouldParseSingleAcceptType() {
            when(request.getHeader("Accept")).thenReturn("application/json");

            List<MediaType> types = manager.resolveMediaTypes(request);

            assertThat(types).containsExactly(MediaType.APPLICATION_JSON);
        }

        @Test
        @DisplayName("should parse multiple Accept types sorted by quality")
        void shouldParseMultipleTypes_SortedByQuality() {
            when(request.getHeader("Accept")).thenReturn(
                    "text/html;q=0.9, application/json, */*;q=0.1");

            List<MediaType> types = manager.resolveMediaTypes(request);

            // Sorted: application/json (q=1.0), text/html (q=0.9), */* (q=0.1)
            assertThat(types.get(0)).isEqualTo(MediaType.APPLICATION_JSON);
            assertThat(types.get(1)).isEqualTo(MediaType.TEXT_HTML);
            assertThat(types.get(2)).isEqualTo(MediaType.ALL);
        }

        @Test
        @DisplayName("should sort concrete types before wildcards at same quality")
        void shouldSortConcreteBeforeWildcards() {
            when(request.getHeader("Accept")).thenReturn("*/*,application/json");

            List<MediaType> types = manager.resolveMediaTypes(request);

            // application/json is more specific than */*
            assertThat(types.get(0)).isEqualTo(MediaType.APPLICATION_JSON);
            assertThat(types.get(1)).isEqualTo(MediaType.ALL);
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/converter/PlainTextMessageConverterTest.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
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
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link PlainTextMessageConverter}.
 */
class PlainTextMessageConverterTest {

    private PlainTextMessageConverter converter;
    private HttpServletResponse response;
    private StringWriter responseBody;

    @BeforeEach
    void setUp() throws Exception {
        converter = new PlainTextMessageConverter();
        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    // ─── getSupportedMediaTypes() ────────────────────────────────

    @Test
    @DisplayName("should support text/plain")
    void shouldSupportTextPlain() {
        assertThat(converter.getSupportedMediaTypes()).containsExactly(MediaType.TEXT_PLAIN);
    }

    // ─── canWrite() ──────────────────────────────────────────────

    @Nested
    @DisplayName("canWrite()")
    class CanWrite {

        @Test
        @DisplayName("should write String as text/plain")
        void shouldWriteString_WhenTextPlain() {
            assertThat(converter.canWrite(String.class, MediaType.TEXT_PLAIN)).isTrue();
        }

        @Test
        @DisplayName("should write String in discovery mode (null media type)")
        void shouldWriteString_WhenDiscoveryMode() {
            assertThat(converter.canWrite(String.class, null)).isTrue();
        }

        @Test
        @DisplayName("should NOT write String as application/json")
        void shouldNotWriteString_WhenApplicationJson() {
            assertThat(converter.canWrite(String.class, MediaType.APPLICATION_JSON)).isFalse();
        }

        @Test
        @DisplayName("should NOT write non-String types")
        void shouldNotWriteNonString() {
            assertThat(converter.canWrite(Map.class, MediaType.TEXT_PLAIN)).isFalse();
            assertThat(converter.canWrite(Integer.class, null)).isFalse();
        }
    }

    // ─── canRead() ───────────────────────────────────────────────

    @Nested
    @DisplayName("canRead()")
    class CanRead {

        @Test
        @DisplayName("should read String from text/plain")
        void shouldReadString_WhenTextPlain() {
            assertThat(converter.canRead(String.class, MediaType.TEXT_PLAIN)).isTrue();
        }

        @Test
        @DisplayName("should read String in discovery mode")
        void shouldReadString_WhenDiscoveryMode() {
            assertThat(converter.canRead(String.class, null)).isTrue();
        }

        @Test
        @DisplayName("should NOT read non-String types")
        void shouldNotReadNonString() {
            assertThat(converter.canRead(Map.class, MediaType.TEXT_PLAIN)).isFalse();
        }
    }

    // ─── write() ─────────────────────────────────────────────────

    @Nested
    @DisplayName("write()")
    class Write {

        @Test
        @DisplayName("should write string value as plain text")
        void shouldWritePlainText() throws Exception {
            converter.write("Hello, World!", MediaType.TEXT_PLAIN, response);

            assertThat(responseBody.toString()).isEqualTo("Hello, World!");
            verify(response).setContentType("text/plain");
            verify(response).setCharacterEncoding("UTF-8");
        }
    }

    // ─── read() ──────────────────────────────────────────────────

    @Nested
    @DisplayName("read()")
    class ReadBody {

        @Test
        @DisplayName("should read request body as String")
        void shouldReadBodyAsString() throws Exception {
            HttpServletRequest request = mock(HttpServletRequest.class);
            byte[] bodyBytes = "plain text body".getBytes(StandardCharsets.UTF_8);
            ByteArrayInputStream byteStream = new ByteArrayInputStream(bodyBytes);
            ServletInputStream sis = new ServletInputStream() {
                @Override public int read() { return byteStream.read(); }
                @Override public boolean isFinished() { return byteStream.available() == 0; }
                @Override public boolean isReady() { return true; }
                @Override public void setReadListener(ReadListener listener) {}
            };
            when(request.getInputStream()).thenReturn(sis);

            Object result = converter.read(String.class, request);

            assertThat(result).isEqualTo("plain text body");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/ContentNegotiationIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.simplespringmvc.annotation.*;
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
 * Integration tests for content negotiation (ch14).
 * Starts a real Tomcat server and verifies that the framework selects the correct
 * HttpMessageConverter based on the Accept header and produces/consumes attributes.
 */
class ContentNegotiationIntegrationTest {

    // ─── Test controllers ────────────────────────────────────────

    @Controller
    @ResponseBody
    static class NegotiationController {

        /**
         * Returns a Map — Jackson can write this as JSON.
         * No produces constraint, so any Accept header works.
         */
        @GetMapping("/data")
        public Map<String, Object> getData() {
            return Map.of("name", "Alice", "age", 30);
        }

        /**
         * Returns a String — both JacksonMessageConverter and
         * PlainTextMessageConverter can handle this, depending on Accept.
         */
        @GetMapping("/greeting")
        public String getGreeting() {
            return "Hello, World!";
        }

        /**
         * Constrained to produce only application/json.
         * Clients requesting text/plain will not match this handler.
         */
        @GetMapping(value = "/json-only", produces = "application/json")
        public Map<String, String> jsonOnly() {
            return Map.of("format", "json");
        }

        /**
         * Constrained to consume only application/json.
         * Requests with Content-Type: text/plain will not match.
         */
        @PostMapping(value = "/consume-json", consumes = "application/json")
        public Map<String, String> consumeJson(@RequestBody Map<String, String> body) {
            return Map.of("received", body.getOrDefault("msg", "none"));
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient client;
    private String baseUrl;
    private final ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new NegotiationController());

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

    // ─── Accept header negotiation ───────────────────────────────

    @Nested
    @DisplayName("Accept header negotiation")
    class AcceptHeaderNegotiation {

        @Test
        @DisplayName("should return JSON when Accept is application/json")
        void shouldReturnJson_WhenAcceptIsJson() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/data"))
                            .header("Accept", "application/json")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("application/json");

            @SuppressWarnings("unchecked")
            Map<String, Object> body = objectMapper.readValue(response.body(), Map.class);
            assertThat(body).containsEntry("name", "Alice");
        }

        @Test
        @DisplayName("should return JSON for String when Accept is application/json")
        void shouldReturnJsonString_WhenAcceptIsJson() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/greeting"))
                            .header("Accept", "application/json")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("application/json");
            // Jackson serializes String as a JSON-quoted string
            assertThat(response.body()).isEqualTo("\"Hello, World!\"");
        }

        @Test
        @DisplayName("should return plain text for String when Accept is text/plain")
        void shouldReturnPlainText_WhenAcceptIsTextPlain() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/greeting"))
                            .header("Accept", "text/plain")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("text/plain");
            // PlainTextMessageConverter writes the raw string
            assertThat(response.body()).isEqualTo("Hello, World!");
        }

        @Test
        @DisplayName("should default to JSON when no Accept header")
        void shouldDefaultToJson_WhenNoAcceptHeader() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/data"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("application/json");
        }

        @Test
        @DisplayName("should respect quality values in Accept header")
        void shouldRespectQualityValues() throws Exception {
            // Prefer text/plain (q=1.0) over application/json (q=0.5)
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/greeting"))
                            .header("Accept", "application/json;q=0.5, text/plain")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("text/plain");
            assertThat(response.body()).isEqualTo("Hello, World!");
        }
    }

    // ─── produces constraint ─────────────────────────────────────

    @Nested
    @DisplayName("@RequestMapping(produces)")
    class ProducesConstraint {

        @Test
        @DisplayName("should return JSON when Accept matches produces")
        void shouldReturnJson_WhenAcceptMatchesProduces() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/json-only"))
                            .header("Accept", "application/json")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.headers().firstValue("Content-Type").orElse(""))
                    .contains("application/json");
        }

        @Test
        @DisplayName("should return 404 when Accept does not match produces")
        void shouldReturn404_WhenAcceptDoesNotMatchProduces() throws Exception {
            // text/plain does not match produces="application/json"
            // So the handler is not matched → 404
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/json-only"))
                            .header("Accept", "text/plain")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(404);
        }
    }

    // ─── consumes constraint ─────────────────────────────────────

    @Nested
    @DisplayName("@RequestMapping(consumes)")
    class ConsumesConstraint {

        @Test
        @DisplayName("should accept request when Content-Type matches consumes")
        void shouldAcceptRequest_WhenContentTypeMatchesConsumes() throws Exception {
            String json = "{\"msg\":\"hello\"}";

            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/consume-json"))
                            .POST(HttpRequest.BodyPublishers.ofString(json))
                            .header("Content-Type", "application/json")
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);

            @SuppressWarnings("unchecked")
            Map<String, String> body = objectMapper.readValue(response.body(), Map.class);
            assertThat(body).containsEntry("received", "hello");
        }

        @Test
        @DisplayName("should return 404 when Content-Type does not match consumes")
        void shouldReturn404_WhenContentTypeDoesNotMatchConsumes() throws Exception {
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/consume-json"))
                            .POST(HttpRequest.BodyPublishers.ofString("plain text"))
                            .header("Content-Type", "text/plain")
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            // text/plain does not match consumes="application/json"
            // So the handler is not matched → 404
            assertThat(response.statusCode()).isEqualTo(404);
        }
    }

    // ─── 406 Not Acceptable ──────────────────────────────────────

    @Nested
    @DisplayName("406 Not Acceptable")
    class NotAcceptable {

        @Test
        @DisplayName("should return 406 when no converter can produce the requested type")
        void shouldReturn406_WhenNoConverterCanProduceRequestedType() throws Exception {
            // Request XML, but we only have JSON and text/plain converters
            HttpResponse<String> response = client.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/data"))
                            .header("Accept", "application/xml")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(406);
        }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **MediaType** | Represents a MIME type (`application/json`, `text/plain;q=0.9`) with parsing, matching, and quality-based sorting for content negotiation |
| **ContentNegotiationManager** | Parses the `Accept` header into a sorted list of `MediaType`s representing what the client wants, most preferred first |
| **Content negotiation algorithm** | Cross-matches client-acceptable types with server-producible types, sorts by specificity and quality, picks the first concrete match, then finds a converter |
| **`produces` / `consumes`** | `@RequestMapping` attributes that constrain which requests a handler can serve by Content-Type (`consumes`) and Accept header (`produces`) |
| **`getSupportedMediaTypes()`** | New `HttpMessageConverter` method that declares what media types a converter handles, used to determine producible types |
| **406 Not Acceptable** | HTTP status returned when the content negotiation algorithm finds no compatible media type between what the client wants and what the server can produce |
| **Converter order** | JacksonMessageConverter registered before PlainTextMessageConverter -- JSON is the default for `*/*` requests |

**Next: Chapter 15 -- Data Binding** -- Bind request parameters directly into Java objects using `@ModelAttribute`, replacing the need to annotate each parameter individually with `@RequestParam`.
