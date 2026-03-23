# Chapter 19: CORS Configuration

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The framework dispatches requests through handler mappings, interceptors, and adapters -- but has no awareness of cross-origin requests | A browser-based JavaScript application running on `http://localhost:3000` cannot call an API on `http://localhost:8080` -- the browser blocks cross-origin requests with no mechanism for the server to allow them | Build a CORS subsystem that detects cross-origin requests, validates them against per-handler and global configuration, writes the required CORS response headers, and handles preflight OPTIONS requests -- all integrated at the handler mapping level rather than as a filter |

---

## 19.1 The Integration Point -- `getHandler()` in SimpleDispatcherServlet

The integration point for CORS is the `getHandler()` method in `SimpleDispatcherServlet`. After resolving the handler and building the initial execution chain, the method checks for CORS configuration and inserts a `CorsInterceptor` at position 0.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
private HandlerExecutionChain getHandler(HttpServletRequest request) {
    if (handlerMapping == null) {
        return null;
    }
    HandlerMethod handler = handlerMapping.lookupHandler(request);

    // ch19: Handle preflight requests that have no explicit OPTIONS handler
    if (handler == null && CorsUtils.isPreFlightRequest(request)) {
        // Look up the handler for the *actual* method in the preflight
        // Access-Control-Request-Method header to get its CORS config
        HandlerMethod actualHandler = findPreflightHandler(request);
        if (actualHandler != null) {
            // Create a no-op handler for the preflight -- the CorsInterceptor
            // will write the CORS headers, and this handler does nothing.
            HandlerMethod noOpHandler = createNoOpHandler();
            HandlerExecutionChain chain = new HandlerExecutionChain(noOpHandler, interceptors);
            applyCorsProcessing(chain, actualHandler, request);
            return chain;
        }
        // No CORS config found -- fall through to return null (404)
        return null;
    }

    if (handler == null) {
        return null;
    }

    HandlerExecutionChain chain = new HandlerExecutionChain(handler, interceptors);

    // ch19: Apply CORS processing for actual CORS requests
    if (CorsUtils.isCorsRequest(request)
            || handlerMapping.hasCorsConfiguration(handler)) {
        applyCorsProcessing(chain, handler, request);
    }

    return chain;
}
```

This maps to `AbstractHandlerMapping.getHandler()` (lines 569-588). Three key paths through this method:

1. **Normal request (no CORS):** Handler is found, chain is built, no CORS processing. Same as before.
2. **Actual CORS request:** Handler is found, chain is built, then `applyCorsProcessing()` inserts a `CorsInterceptor` at position 0 so CORS validation runs before any other interceptors.
3. **Preflight request (OPTIONS + Origin + Access-Control-Request-Method):** No handler exists for OPTIONS, so we look up the handler for the *actual* method (from the preflight header), create a no-op handler, and attach CORS processing. The `CorsInterceptor` writes the CORS headers, and the no-op handler does nothing.

The merging logic in `applyCorsProcessing()` combines handler-level config (from `@CrossOrigin`) with global config (from `CorsRegistry`):

```java
private void applyCorsProcessing(HandlerExecutionChain chain,
                                  HandlerMethod handler,
                                  HttpServletRequest request) {
    // Get handler-level config (from @CrossOrigin)
    CorsConfiguration handlerConfig = handlerMapping.getCorsConfiguration(handler);

    // Get global config (from CorsRegistry)
    CorsConfiguration globalConfig = handlerMapping.getGlobalCorsConfiguration(
            request.getRequestURI());

    // Merge: global is base, handler overlays
    CorsConfiguration mergedConfig;
    if (globalConfig != null && handlerConfig != null) {
        mergedConfig = globalConfig.combine(handlerConfig);
    } else if (globalConfig != null) {
        mergedConfig = globalConfig;
    } else if (handlerConfig != null) {
        mergedConfig = handlerConfig;
    } else {
        return; // no CORS config at all
    }

    // Validate
    mergedConfig.validateAllowCredentials();

    // Insert CorsInterceptor at position 0
    chain.addInterceptorFirst(new CorsInterceptor(mergedConfig, corsProcessor));
}
```

This maps to `AbstractHandlerMapping.getCorsHandlerExecutionChain()` (line 746). The merging strategy is: **global config is the base, handler config is overlaid** via `globalConfig.combine(handlerConfig)`. This means handler-level origins are *added* to global origins, not replacing them.

Three design decisions in this integration point:

1. **CORS integrates at the handler mapping level, not as a servlet filter.** This allows per-handler CORS configuration via `@CrossOrigin`. A filter would only support global configuration.
2. **The `CorsInterceptor` is inserted at position 0** using the new `addInterceptorFirst()` method on `HandlerExecutionChain`. This ensures CORS validation happens before authentication interceptors, logging interceptors, or any other interceptors that might reject the request.
3. **Preflight requests get a no-op handler.** Since there is no explicit OPTIONS mapping, the framework creates a synthetic handler that does nothing. The `CorsInterceptor` writes all the necessary response headers, and the no-op handler is invoked but returns null.

---

## 19.2 CorsConfiguration -- The Central Data Structure

`CorsConfiguration` holds all CORS settings: allowed origins, methods, headers, exposed headers, credentials, and max age. It provides three check methods for validation and a `combine()` method for merging configurations.

**New file:** `src/main/java/com/simplespringmvc/cors/CorsConfiguration.java`

The three check methods form the validation core:

```java
public String checkOrigin(String origin) {
    if (origin == null) {
        return null;
    }
    if (allowedOrigins == null) {
        return null;
    }
    if (allowedOrigins.contains(ALL)) {
        // When credentials are enabled, can't use "*" -- echo back the origin
        if (Boolean.TRUE.equals(allowCredentials)) {
            return origin;
        }
        return ALL;
    }
    for (String allowed : allowedOrigins) {
        if (origin.equalsIgnoreCase(allowed)) {
            return origin;
        }
    }
    return null;
}
```

`checkOrigin()` returns the value to use in the `Access-Control-Allow-Origin` response header, or `null` to reject. When `allowCredentials` is true and `allowedOrigins` contains `"*"`, the response echoes back the specific origin instead of `"*"` -- this is a W3C spec requirement because `"*"` and credentials are incompatible.

```java
public List<String> checkHttpMethod(String method) {
    if (method == null) {
        return null;
    }
    if (allowedMethods == null) {
        return null;
    }
    if (allowedMethods.contains(ALL)) {
        return List.of(method);
    }
    for (String allowed : allowedMethods) {
        if (allowed.equalsIgnoreCase(method)) {
            return Collections.unmodifiableList(allowedMethods);
        }
    }
    return null;
}
```

`checkHttpMethod()` returns the list of allowed methods (for the `Access-Control-Allow-Methods` header) or `null` to reject. When the wildcard `"*"` is used, it returns just the requested method rather than `"*"`.

```java
public List<String> checkHeaders(List<String> headers) {
    if (headers == null || headers.isEmpty()) {
        return Collections.emptyList();
    }
    if (allowedHeaders == null) {
        return null;
    }
    if (allowedHeaders.contains(ALL)) {
        return headers;
    }
    List<String> result = new ArrayList<>();
    for (String header : headers) {
        boolean found = false;
        for (String allowed : allowedHeaders) {
            if (header.equalsIgnoreCase(allowed)) {
                found = true;
                result.add(header);
                break;
            }
        }
        if (!found) {
            return null;
        }
    }
    return result;
}
```

`checkHeaders()` validates preflight request headers. If *any* requested header is not in the allowed list, the entire request is rejected (returns `null`). This is an all-or-nothing check.

The `combine()` method merges two configurations additively for lists, with "other overrides this" for scalar values:

```java
public CorsConfiguration combine(CorsConfiguration other) {
    if (other == null) {
        return this;
    }
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(combineLists(this.allowedOrigins, other.allowedOrigins));
    config.setAllowedMethods(combineLists(this.allowedMethods, other.allowedMethods));
    config.setAllowedHeaders(combineLists(this.allowedHeaders, other.allowedHeaders));
    config.setExposedHeaders(combineLists(this.exposedHeaders, other.exposedHeaders));
    config.setAllowCredentials(other.allowCredentials != null
            ? other.allowCredentials : this.allowCredentials);
    config.setMaxAge(other.maxAge != null ? other.maxAge : this.maxAge);
    return config;
}
```

The `combineLists()` helper has a special rule: if either list contains `"*"`, the result collapses to just `["*"]`. This prevents silly results like `["*", "https://specific.com"]` -- a wildcard already covers everything.

The `applyPermitDefaultValues()` method fills in gaps with permissive defaults (origins: `["*"]`, methods: `["GET", "HEAD", "POST"]`, headers: `["*"]`, maxAge: 1800). It only applies when a value is `null`, never overriding explicit settings. This is called after processing `@CrossOrigin` annotations.

---

## 19.3 CorsUtils & CorsProcessor -- Detection and Enforcement

### CorsUtils: Classifying Requests

`CorsUtils` has two static methods that classify HTTP requests:

**New file:** `src/main/java/com/simplespringmvc/cors/CorsUtils.java`

```java
public static boolean isCorsRequest(HttpServletRequest request) {
    String origin = request.getHeader("Origin");
    if (origin == null || origin.isEmpty()) {
        return false;
    }
    try {
        URI originUri = new URI(origin);
        String scheme = request.getScheme();
        String host = request.getServerName();
        int port = request.getServerPort();

        boolean sameScheme = scheme.equalsIgnoreCase(originUri.getScheme());
        boolean sameHost = host.equalsIgnoreCase(originUri.getHost());
        boolean samePort = (port == getPort(originUri, originUri.getScheme()));

        return !(sameScheme && sameHost && samePort);
    } catch (Exception e) {
        return true;  // Malformed origin -- treat as cross-origin for safety
    }
}
```

A request is CORS if it has an `Origin` header that differs from the server's own scheme/host/port. If the Origin is malformed, it is treated as cross-origin for safety.

```java
public static boolean isPreFlightRequest(HttpServletRequest request) {
    return "OPTIONS".equalsIgnoreCase(request.getMethod())
            && request.getHeader("Origin") != null
            && request.getHeader("Access-Control-Request-Method") != null;
}
```

A preflight request is an OPTIONS request with both `Origin` and `Access-Control-Request-Method` headers. Browsers send preflight requests before actual cross-origin requests that use non-simple methods (PUT, DELETE) or non-simple headers.

### CorsProcessor & DefaultCorsProcessor: Enforcement

`CorsProcessor` is the strategy interface -- it validates a request against a `CorsConfiguration` and writes the CORS response headers.

**New file:** `src/main/java/com/simplespringmvc/cors/CorsProcessor.java`

```java
public interface CorsProcessor {
    boolean processRequest(CorsConfiguration configuration,
                           HttpServletRequest request,
                           HttpServletResponse response) throws IOException;
}
```

Returns `false` to reject the request (403); `true` to proceed.

**New file:** `src/main/java/com/simplespringmvc/cors/DefaultCorsProcessor.java`

The `DefaultCorsProcessor` follows a strict processing pipeline:

```
1. No CorsConfiguration        --> skip (allow through)
2. Add Vary headers             --> Origin, Access-Control-Request-Method, Access-Control-Request-Headers
3. Not a CORS request?          --> skip
4. Already has Allow-Origin?    --> skip (no double-processing)
5. Validate origin              --> reject with 403 if not allowed
6. Validate method              --> reject with 403 if not allowed
7. Validate headers (preflight) --> reject with 403 if not allowed
8. Write CORS response headers
```

Steps 5-8 happen in `handleInternal()`:

```java
protected boolean handleInternal(CorsConfiguration config,
                                 HttpServletRequest request,
                                 HttpServletResponse response,
                                 boolean isPreFlight) throws IOException {

    String origin = request.getHeader("Origin");

    // Step 5: Validate origin
    String allowedOrigin = config.checkOrigin(origin);
    if (allowedOrigin == null) {
        rejectRequest(response);
        return false;
    }

    // Step 6: Validate HTTP method
    String requestMethod = isPreFlight
            ? request.getHeader("Access-Control-Request-Method")
            : request.getMethod();
    List<String> allowedMethods = config.checkHttpMethod(requestMethod);
    if (allowedMethods == null) {
        rejectRequest(response);
        return false;
    }

    // Step 7: Validate headers (preflight only)
    List<String> requestHeaders = getAccessControlRequestHeaders(request);
    List<String> allowedHeaders = null;
    if (isPreFlight) {
        allowedHeaders = config.checkHeaders(requestHeaders);
        if (requestHeaders != null && !requestHeaders.isEmpty() && allowedHeaders == null) {
            rejectRequest(response);
            return false;
        }
    }

    // Step 8: Write CORS response headers
    response.setHeader("Access-Control-Allow-Origin", allowedOrigin);
    // ... Allow-Methods, Allow-Headers, Expose-Headers, Allow-Credentials, Max-Age
    return true;
}
```

Notice that for **actual** CORS requests, the method is taken from `request.getMethod()`, but for **preflight** requests, it is taken from the `Access-Control-Request-Method` header. This is because the preflight is always OPTIONS -- the method being checked is the one the browser *plans* to use.

The Vary headers (step 2) are critical for HTTP caches. Without them, a cache might serve a CORS-enabled response to a same-origin request or vice versa.

---

## 19.4 @CrossOrigin Annotation + Handler Mapping Extraction

### The @CrossOrigin Annotation

**New file:** `src/main/java/com/simplespringmvc/annotation/CrossOrigin.java`

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CrossOrigin {
    String[] origins() default {};
    String[] methods() default {};
    String[] allowedHeaders() default {};
    String[] exposedHeaders() default {};
    boolean allowCredentials() default false;
    long maxAge() default -1;
}
```

`@CrossOrigin` can be applied at the type level (applies to all handler methods in the class) and at the method level (overrides or adds to type-level configuration). When both are present, their values are combined additively.

### Handler Mapping Extraction

**Modifying:** `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java`

During handler method detection, `@CrossOrigin` annotations are read from both the class and method levels. The `initCorsConfiguration()` method builds a `CorsConfiguration` per handler method:

```java
private CorsConfiguration initCorsConfiguration(CrossOrigin typeAnnotation,
                                                 CrossOrigin methodAnnotation,
                                                 String mappedHttpMethod) {
    if (typeAnnotation == null && methodAnnotation == null) {
        return null;
    }

    CorsConfiguration config = new CorsConfiguration();
    updateCorsConfig(config, typeAnnotation);
    updateCorsConfig(config, methodAnnotation);

    // If no methods explicitly set, default to the mapped HTTP method
    if (config.getAllowedMethods() == null || config.getAllowedMethods().isEmpty()) {
        if (mappedHttpMethod != null && !mappedHttpMethod.isEmpty()) {
            config.addAllowedMethod(mappedHttpMethod);
        }
    }

    return config.applyPermitDefaultValues();
}
```

This maps to `RequestMappingHandlerMapping.initCorsConfiguration()` (line 492). The method-default behavior is important: if `@CrossOrigin` does not explicitly declare `methods`, the allowed method defaults to the handler's mapped HTTP method. For a `@GetMapping` handler, only GET is allowed in CORS requests.

The `updateCorsConfig()` helper *adds* values (not replaces) so type-level and method-level annotations combine:

```java
private void updateCorsConfig(CorsConfiguration config, CrossOrigin annotation) {
    if (annotation == null) {
        return;
    }
    for (String origin : annotation.origins()) {
        config.addAllowedOrigin(origin);
    }
    for (String method : annotation.methods()) {
        config.addAllowedMethod(method);
    }
    // ... headers, exposedHeaders, credentials, maxAge
}
```

The configurations are stored in a `Map<HandlerMethod, CorsConfiguration>` field called `corsConfigurations`, keyed by handler method identity. At lookup time, `getCorsConfiguration(handlerMethod)` returns the stored config.

For global CORS, `SimpleHandlerMapping` also stores a `Map<String, CorsConfiguration>` from `CorsRegistry` and provides `getGlobalCorsConfiguration(requestPath)` with simple prefix matching for `"/**"` patterns.

---

## 19.5 CorsInterceptor + CorsRegistry/CorsRegistration

### CorsInterceptor

**New file:** `src/main/java/com/simplespringmvc/cors/CorsInterceptor.java`

```java
public class CorsInterceptor implements HandlerInterceptor {

    private final CorsConfiguration corsConfiguration;
    private final CorsProcessor corsProcessor;

    public CorsInterceptor(CorsConfiguration corsConfiguration) {
        this(corsConfiguration, new DefaultCorsProcessor());
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             HandlerMethod handler) throws Exception {
        try {
            return corsProcessor.processRequest(corsConfiguration, request, response);
        } catch (IOException e) {
            throw new RuntimeException("CORS processing failed", e);
        }
    }
}
```

The interceptor delegates entirely to the `CorsProcessor`. If the processor rejects the request (returns `false`), `preHandle` returns `false` to stop the chain -- the 403 response has already been written by the processor.

In the real framework, this is a private inner class of `AbstractHandlerMapping` (line 761). We promote it to a top-level class for clarity.

### HandlerExecutionChain Enhancement

**Modifying:** `src/main/java/com/simplespringmvc/interceptor/HandlerExecutionChain.java`

A new method allows inserting an interceptor at position 0:

```java
public void addInterceptorFirst(HandlerInterceptor interceptor) {
    this.interceptorList.add(0, interceptor);
}
```

This ensures the `CorsInterceptor` runs before all other interceptors, matching the real framework's behavior in `AbstractHandlerMapping.getCorsHandlerExecutionChain()` (line 754).

### CorsRegistry & CorsRegistration

**New file:** `src/main/java/com/simplespringmvc/cors/CorsRegistry.java`

A fluent API for registering global CORS configurations by URL pattern:

```java
CorsRegistry registry = new CorsRegistry();
registry.addMapping("/api/**")
    .allowedOrigins("https://example.com")
    .allowedMethods("GET", "POST")
    .maxAge(3600);
```

**New file:** `src/main/java/com/simplespringmvc/cors/CorsRegistration.java`

Each `CorsRegistration` holds a path pattern and a `CorsConfiguration`. The constructor calls `applyPermitDefaultValues()` so the registration starts permissive -- each setter *replaces* the default.

The `SimpleDispatcherServlet` exposes `setCorsRegistry(CorsRegistry)` which passes the configurations to `SimpleHandlerMapping`:

```java
public void setCorsRegistry(CorsRegistry corsRegistry) {
    if (handlerMapping != null) {
        handlerMapping.setCorsConfigurations(corsRegistry.getCorsConfigurations());
    }
}
```

---

## 19.6 Try It Yourself

<details>
<summary>Challenge 1: What CORS headers get written?</summary>

Given this controller:

```java
@Controller
@CrossOrigin(origins = "https://trusted.com")
public class ApiController {
    @GetMapping("/api/data")
    @ResponseBody
    public String getData() { return "data"; }
}
```

A browser on `https://trusted.com` sends:

```
GET /api/data HTTP/1.1
Host: localhost:8080
Origin: https://trusted.com
```

What headers will appear in the response?

**Answer:**
```
Access-Control-Allow-Origin: https://trusted.com
Vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
```

The `Access-Control-Allow-Origin` echoes back the trusted origin (not `"*"` since the config specifies an exact origin). The `Vary` headers are always added by `DefaultCorsProcessor` so HTTP caches know the response varies by origin.

No `Access-Control-Allow-Methods` or `Access-Control-Max-Age` headers are written because this is an **actual** CORS request, not a preflight. Those headers are only written for preflight responses.

</details>

<details>
<summary>Challenge 2: What happens with this preflight request?</summary>

Given this controller:

```java
@Controller
public class ApiController {
    @RequestMapping(path = "/api/items", method = "PUT")
    @CrossOrigin(origins = "https://app.com", methods = {"PUT", "DELETE"})
    @ResponseBody
    public String update() { return "updated"; }
}
```

A browser sends this preflight:

```
OPTIONS /api/items HTTP/1.1
Host: localhost:8080
Origin: https://evil.com
Access-Control-Request-Method: PUT
```

What happens?

**Answer:** The server responds with **403 Forbidden** and the body `"Invalid CORS request"`.

The flow:
1. `getHandler()` finds no OPTIONS handler, detects this is a preflight, and looks up the handler for PUT `/api/items`.
2. The handler has `@CrossOrigin(origins = "https://app.com")`.
3. A `CorsInterceptor` is created with this configuration and attached to a no-op handler.
4. During `applyPreHandle()`, the `CorsInterceptor` calls `DefaultCorsProcessor.processRequest()`.
5. `checkOrigin("https://evil.com")` returns `null` because `"https://evil.com"` is not in `["https://app.com"]`.
6. `rejectRequest()` sets 403 and writes the error message.

</details>

<details>
<summary>Challenge 3: How does global + handler merging work?</summary>

Given:

```java
// Global config:
CorsRegistry registry = new CorsRegistry();
registry.addMapping("/**")
    .allowedOrigins("https://global.com")
    .allowedMethods("GET", "POST");

// Handler config:
@Controller
@CrossOrigin(origins = "https://handler.com")
public class MyController {
    @GetMapping("/data")
    @ResponseBody
    public String data() { return "data"; }
}
```

A request arrives from `https://handler.com` with `GET /data`. What is the merged CORS configuration?

**Answer:** The merged configuration has:
- `allowedOrigins`: `["https://global.com", "https://handler.com"]` -- lists are combined additively
- `allowedMethods`: `["GET", "POST", "GET"]` -- global methods + handler default (GET from `@GetMapping`)
- `allowedHeaders`: `["*"]` -- from the handler's `applyPermitDefaultValues()`
- `maxAge`: `1800` -- default from `applyPermitDefaultValues()` on the handler config

The global config is the base, the handler config is overlaid via `globalConfig.combine(handlerConfig)`. Lists are merged additively, scalars use "other overrides this".

The request from `https://handler.com` succeeds because `"https://handler.com"` is in the merged origins list.

</details>

---

## 19.7 Tests

### Unit Tests

#### CorsConfiguration

**New file:** `src/test/java/com/simplespringmvc/cors/CorsConfigurationTest.java`

```java
@Test
void shouldSetPermissiveDefaults_WhenApplyPermitDefaultValuesCalled() {
    CorsConfiguration config = new CorsConfiguration();
    config.applyPermitDefaultValues();

    assertThat(config.getAllowedOrigins()).containsExactly("*");
    assertThat(config.getAllowedMethods()).containsExactly("GET", "HEAD", "POST");
    assertThat(config.getAllowedHeaders()).containsExactly("*");
    assertThat(config.getMaxAge()).isEqualTo(1800L);
}

@Test
void shouldEchoOrigin_WhenAllOriginsAllowedWithCredentials() {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("*");
    config.setAllowCredentials(true);

    assertThat(config.checkOrigin("https://example.com"))
            .isEqualTo("https://example.com");
}

@Test
void shouldMergeOriginsAdditively_WhenCombining() {
    CorsConfiguration global = new CorsConfiguration();
    global.addAllowedOrigin("https://a.com");

    CorsConfiguration handler = new CorsConfiguration();
    handler.addAllowedOrigin("https://b.com");

    CorsConfiguration merged = global.combine(handler);
    assertThat(merged.getAllowedOrigins())
            .containsExactly("https://a.com", "https://b.com");
}

@Test
void shouldThrow_WhenCredentialsTrueAndOriginWildcard() {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("*");
    config.setAllowCredentials(true);

    assertThatThrownBy(config::validateAllowCredentials)
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("allowedOrigins cannot contain '*'");
}
```

#### DefaultCorsProcessor

**New file:** `src/test/java/com/simplespringmvc/cors/DefaultCorsProcessorTest.java`

```java
@Test
void shouldWriteAllowOrigin_WhenOriginAllowed() throws Exception {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("https://example.com");
    config.addAllowedMethod("GET");
    request.setMethod("GET");

    boolean result = processor.processRequest(config, request, response);

    assertThat(result).isTrue();
    assertThat(response.getHeader("Access-Control-Allow-Origin"))
            .isEqualTo("https://example.com");
}

@Test
void shouldReject403_WhenOriginNotAllowed() throws Exception {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("https://trusted.com");
    config.addAllowedMethod("GET");
    request.setMethod("GET");

    boolean result = processor.processRequest(config, request, response);

    assertThat(result).isFalse();
    assertThat(response.getStatus()).isEqualTo(403);
}

@Test
void shouldHandlePreflight_WhenOptionsWithRequestMethod() throws Exception {
    request.setMethod("OPTIONS");
    request.addHeader("Access-Control-Request-Method", "PUT");

    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("https://example.com");
    config.addAllowedMethod("PUT");
    config.addAllowedHeader("Content-Type");

    boolean result = processor.processRequest(config, request, response);

    assertThat(result).isTrue();
    assertThat(response.getHeader("Access-Control-Allow-Origin"))
            .isEqualTo("https://example.com");
    assertThat(response.getHeader("Access-Control-Allow-Methods"))
            .contains("PUT");
}
```

#### CorsUtils

**New file:** `src/test/java/com/simplespringmvc/cors/CorsUtilsTest.java`

```java
@Test
void shouldReturnTrue_WhenOriginDiffersFromServer() {
    MockHttpServletRequest request = new MockHttpServletRequest();
    request.setScheme("http");
    request.setServerName("localhost");
    request.setServerPort(8080);
    request.addHeader("Origin", "https://example.com");

    assertThat(CorsUtils.isCorsRequest(request)).isTrue();
}

@Test
void shouldReturnFalse_WhenNoOriginHeader() {
    MockHttpServletRequest request = new MockHttpServletRequest();
    request.setScheme("http");
    request.setServerName("localhost");
    request.setServerPort(8080);

    assertThat(CorsUtils.isCorsRequest(request)).isFalse();
}

@Test
void shouldReturnTrue_WhenOptionsWithOriginAndRequestMethod() {
    MockHttpServletRequest request = new MockHttpServletRequest();
    request.setMethod("OPTIONS");
    request.addHeader("Origin", "https://example.com");
    request.addHeader("Access-Control-Request-Method", "PUT");

    assertThat(CorsUtils.isPreFlightRequest(request)).isTrue();
}
```

#### CorsRegistry

**New file:** `src/test/java/com/simplespringmvc/cors/CorsRegistryTest.java`

```java
@Test
void shouldSupportFluentConfiguration_WhenChained() {
    CorsRegistry registry = new CorsRegistry();
    registry.addMapping("/api/**")
            .allowedOrigins("https://example.com")
            .allowedMethods("GET", "POST")
            .allowedHeaders("Content-Type", "Authorization")
            .exposedHeaders("X-Custom")
            .allowCredentials(true)
            .maxAge(3600);

    CorsConfiguration config = registry.getCorsConfigurations().get("/api/**");

    assertThat(config.getAllowedOrigins()).containsExactly("https://example.com");
    assertThat(config.getAllowedMethods()).containsExactly("GET", "POST");
    assertThat(config.getAllowedHeaders()).containsExactly("Content-Type", "Authorization");
    assertThat(config.getExposedHeaders()).containsExactly("X-Custom");
    assertThat(config.getAllowCredentials()).isTrue();
    assertThat(config.getMaxAge()).isEqualTo(3600L);
}
```

### Integration Test

**New file:** `src/test/java/com/simplespringmvc/integration/CorsIntegrationTest.java`

The integration test verifies end-to-end CORS processing through the full dispatch pipeline:

```java
@Controller
@CrossOrigin(origins = "https://trusted.com")
static class ClassLevelCorsController {
    @GetMapping("/class-cors/hello")
    @ResponseBody
    public String hello() { return "Hello from class-cors"; }
}

@Test
void shouldAllowRequest_WhenOriginIsTrusted() throws Exception {
    MockHttpServletRequest request = createCorsRequest(
            "GET", "/class-cors/hello", "https://trusted.com");
    MockHttpServletResponse response = new MockHttpServletResponse();

    servlet.service(request, response);

    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(response.getHeader("Access-Control-Allow-Origin"))
            .isEqualTo("https://trusted.com");
}

@Test
void shouldRejectRequest_WhenOriginNotTrusted() throws Exception {
    MockHttpServletRequest request = createCorsRequest(
            "GET", "/class-cors/hello", "https://evil.com");
    MockHttpServletResponse response = new MockHttpServletResponse();

    servlet.service(request, response);

    assertThat(response.getStatus()).isEqualTo(403);
}
```

The integration test covers class-level `@CrossOrigin`, method-level `@CrossOrigin`, merged class + method annotations, preflight handling, global `CorsRegistry` configuration, merged global + handler configuration, and same-origin requests.

---

## 19.8 Why This Works

> ★ **Insight** -------------------------------------------
> **CORS at the handler mapping level, not a filter, enables per-handler configuration.** A servlet filter processes requests before they reach any Spring MVC machinery -- it cannot know which handler method will handle the request, so it can only apply global CORS rules. By integrating CORS into `getHandler()`, the framework has already resolved the handler and can look up its `@CrossOrigin` annotation. This means you can have different CORS policies for different endpoints: `/api/public/**` allows all origins, while `/api/admin/**` restricts to a single trusted domain. The real framework makes the same architectural choice -- `AbstractHandlerMapping.getHandler()` (line 569) is where CORS processing lives, not in any filter.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The preflight no-op handler solves a chicken-and-egg problem.** A preflight OPTIONS request asks "may I send a PUT to `/api/items`?" -- but there is no OPTIONS handler registered for `/api/items`. The framework needs the CORS configuration from the PUT handler to answer the preflight, but it cannot run the PUT handler. The solution: look up the PUT handler to get its CORS config, create a no-op handler that does nothing, attach the CORS config as an interceptor, and let the interceptor write the response headers. The no-op handler is invoked but returns null. In the real framework, this is `AbstractHandlerMapping.PreFlightHttpRequestHandler` (line 795) -- a private class that implements `HttpRequestHandler` and does nothing.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Additive merging with wildcard collapse makes the combine() method both powerful and safe.** When global config allows `["https://global.com"]` and handler config allows `["https://handler.com"]`, the merge produces `["https://global.com", "https://handler.com"]` -- both origins work. But if either list contains `"*"`, the result collapses to just `["*"]` because a wildcard already covers everything. This prevents confusing states where a specific origin is listed alongside a wildcard. The same rule applies to methods and headers. This matches `CorsConfiguration.combine()` (line 367) in the real framework.
> -----------------------------------------------------------

---

## 19.9 What We Enhanced

| Aspect | Before (ch18) | Current (ch19) | Real Framework |
|--------|---------------|----------------|----------------|
| **Cross-origin requests** | No CORS support -- browsers block cross-origin API calls | Full CORS support: origin/method/header validation, response headers, preflight handling | `DefaultCorsProcessor` at `DefaultCorsProcessor.java:72` |
| **Per-handler CORS** | Not possible | `@CrossOrigin` on classes and methods, combined additively | `RequestMappingHandlerMapping.initCorsConfiguration()` at line 492 |
| **Global CORS** | Not possible | `CorsRegistry` with fluent API, merged with handler-level configs | `CorsRegistry` + `UrlBasedCorsConfigurationSource` |
| **Preflight handling** | OPTIONS requests return 404 | Synthetic no-op handler + CorsInterceptor writes response headers | `AbstractHandlerMapping.PreFlightHttpRequestHandler` at line 795 |
| **Execution chain** | Interceptors added at end only | New `addInterceptorFirst()` inserts CorsInterceptor at position 0 | `getCorsHandlerExecutionChain()` at `AbstractHandlerMapping.java:754` |

---

## 19.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `CorsConfiguration` | `CorsConfiguration` | `CorsConfiguration.java` | Real version supports origin patterns with wildcards (`https://*.example.com`), `allowPrivateNetwork`, resolved methods cache, and `OriginPattern` inner class |
| `CorsConfiguration.checkOrigin()` | `CorsConfiguration.checkOrigin()` | `CorsConfiguration.java:422` | Real version also checks origin patterns via `checkOriginPattern()` |
| `CorsConfiguration.combine()` | `CorsConfiguration.combine()` | `CorsConfiguration.java:367` | Same additive list merge + scalar override strategy |
| `CorsConfiguration.applyPermitDefaultValues()` | `CorsConfiguration.applyPermitDefaultValues()` | `CorsConfiguration.java:321` | Same defaults; real version also handles `DEFAULT_PERMIT_METHODS` list |
| `CorsProcessor` | `CorsProcessor` | `CorsProcessor.java` | Same single-method strategy interface |
| `DefaultCorsProcessor.processRequest()` | `DefaultCorsProcessor.processRequest()` | `DefaultCorsProcessor.java:72` | Real version has `allowPrivateNetwork` support and customizable rejection handler |
| `DefaultCorsProcessor.handleInternal()` | `DefaultCorsProcessor.handleInternal()` | `DefaultCorsProcessor.java:127` | Same validation + header writing flow |
| `CorsUtils.isCorsRequest()` | `CorsUtils.isCorsRequest()` | `CorsUtils.java:43` | Real version uses `UriComponentsBuilder.fromOriginHeader()` with default port normalization |
| `CorsUtils.isPreFlightRequest()` | `CorsUtils.isPreFlightRequest()` | `CorsUtils.java:73` | Same three-condition check: OPTIONS + Origin + Access-Control-Request-Method |
| `@CrossOrigin` | `@CrossOrigin` | `CrossOrigin.java` | Real version has `originPatterns`, `allowPrivateNetwork`, String-typed `allowCredentials`, placeholder resolution |
| `CorsInterceptor` | `AbstractHandlerMapping.CorsInterceptor` | `AbstractHandlerMapping.java:761` | Real version is a private inner class; our version is a top-level class |
| `initCorsConfiguration()` | `RequestMappingHandlerMapping.initCorsConfiguration()` | `RequestMappingHandlerMapping.java:492` | Real version resolves `${...}` placeholders, handles `originPatterns`, uses `RequestCondition` |
| `getHandler()` CORS block | `AbstractHandlerMapping.getHandler()` | `AbstractHandlerMapping.java:569-588` | Same flow: check CORS, merge configs, insert interceptor at position 0 |
| `applyCorsProcessing()` | `AbstractHandlerMapping.getCorsHandlerExecutionChain()` | `AbstractHandlerMapping.java:746` | Same merge strategy: global base + handler overlay |
| `CorsRegistry` | `CorsRegistry` | `CorsRegistry.java` | Same fluent API, same `getCorsConfigurations()` method |
| `CorsRegistration` | `CorsRegistration` | `CorsRegistration.java` | Real version supports `allowedOriginPatterns()` and `allowPrivateNetwork()` |

---

## 19.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/CrossOrigin.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation for permitting cross-origin requests on specific handler classes
 * and/or handler methods. Can be applied at the type level (applies to all
 * handler methods) and at the method level (overrides/merges with type-level).
 *
 * Maps to: {@code org.springframework.web.bind.annotation.CrossOrigin}
 *
 * When present, a {@code CorsConfiguration} is built from the annotation
 * attributes at scan time and stored per handler method. During request
 * dispatch, this configuration is merged with any global CORS configuration
 * and enforced via a {@code CorsInterceptor} inserted at position 0 in
 * the execution chain.
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code originPatterns} — only exact origin matching</li>
 *   <li>No {@code allowPrivateNetwork} attribute</li>
 *   <li>{@code allowCredentials} is a boolean, not a String</li>
 *   <li>No placeholder resolution (${...})</li>
 * </ul>
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CrossOrigin {

    /**
     * List of allowed origins. A single "*" means all origins are allowed.
     * Default is all origins ("*") via {@code applyPermitDefaultValues()}.
     *
     * Maps to: {@code CrossOrigin.origins()} (alias for value())
     */
    String[] origins() default {};

    /**
     * List of HTTP methods to allow. Default is the mapped methods of the
     * handler (e.g., if a handler is mapped to GET, only GET is allowed).
     *
     * Maps to: {@code CrossOrigin.methods()}
     */
    String[] methods() default {};

    /**
     * List of request headers allowed in actual requests. A single "*"
     * means all headers are allowed.
     *
     * Maps to: {@code CrossOrigin.allowedHeaders()}
     */
    String[] allowedHeaders() default {};

    /**
     * List of response headers the browser is allowed to expose to the client.
     *
     * Maps to: {@code CrossOrigin.exposedHeaders()}
     */
    String[] exposedHeaders() default {};

    /**
     * Whether user credentials (cookies, HTTP authentication) are supported.
     * When true, "*" cannot be used for origins.
     *
     * Maps to: {@code CrossOrigin.allowCredentials()}
     */
    boolean allowCredentials() default false;

    /**
     * Max age (in seconds) for the preflight response cache.
     * A value of -1 means undefined (the default will be applied).
     *
     * Maps to: {@code CrossOrigin.maxAge()}
     */
    long maxAge() default -1;
}
```

#### File: `src/main/java/com/simplespringmvc/cors/CorsConfiguration.java` [NEW]

```java
package com.simplespringmvc.cors;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * A container for CORS configuration that holds the allowed origins, methods,
 * headers, and other settings. Provides check methods to validate a request
 * against the configuration and a combine method to merge configurations
 * from multiple levels (handler-level + global).
 *
 * Maps to: {@code org.springframework.web.cors.CorsConfiguration}
 *
 * The real class also supports:
 * <ul>
 *   <li>Origin patterns with wildcards (e.g., "https://*.example.com")</li>
 *   <li>Private network access control</li>
 *   <li>Resolved methods cache</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No origin pattern matching — only exact origins or "*"</li>
 *   <li>No allowPrivateNetwork</li>
 *   <li>No OriginPattern inner class</li>
 *   <li>No resolved methods caching</li>
 * </ul>
 */
public class CorsConfiguration {

    /** Wildcard representing all origins, methods, or headers. */
    public static final String ALL = "*";

    /** Default max age for preflight cache: 30 minutes. */
    public static final long DEFAULT_MAX_AGE = 1800;

    private List<String> allowedOrigins;
    private List<String> allowedMethods;
    private List<String> allowedHeaders;
    private List<String> exposedHeaders;
    private Boolean allowCredentials;
    private Long maxAge;

    public CorsConfiguration() {
    }

    // ─── Setters and adders ──────────────────────────────────

    public void setAllowedOrigins(List<String> origins) {
        this.allowedOrigins = origins != null ? new ArrayList<>(origins) : null;
    }

    public void addAllowedOrigin(String origin) {
        if (this.allowedOrigins == null) {
            this.allowedOrigins = new ArrayList<>();
        }
        this.allowedOrigins.add(origin);
    }

    public void setAllowedMethods(List<String> methods) {
        this.allowedMethods = methods != null ? new ArrayList<>(methods) : null;
    }

    public void addAllowedMethod(String method) {
        if (this.allowedMethods == null) {
            this.allowedMethods = new ArrayList<>();
        }
        this.allowedMethods.add(method.toUpperCase());
    }

    public void setAllowedHeaders(List<String> headers) {
        this.allowedHeaders = headers != null ? new ArrayList<>(headers) : null;
    }

    public void addAllowedHeader(String header) {
        if (this.allowedHeaders == null) {
            this.allowedHeaders = new ArrayList<>();
        }
        this.allowedHeaders.add(header);
    }

    public void setExposedHeaders(List<String> headers) {
        this.exposedHeaders = headers != null ? new ArrayList<>(headers) : null;
    }

    public void addExposedHeader(String header) {
        if (this.exposedHeaders == null) {
            this.exposedHeaders = new ArrayList<>();
        }
        this.exposedHeaders.add(header);
    }

    public void setAllowCredentials(Boolean allowCredentials) {
        this.allowCredentials = allowCredentials;
    }

    public void setMaxAge(Long maxAge) {
        this.maxAge = maxAge;
    }

    // ─── Getters ─────────────────────────────────────────────

    public List<String> getAllowedOrigins() {
        return allowedOrigins;
    }

    public List<String> getAllowedMethods() {
        return allowedMethods;
    }

    public List<String> getAllowedHeaders() {
        return allowedHeaders;
    }

    public List<String> getExposedHeaders() {
        return exposedHeaders;
    }

    public Boolean getAllowCredentials() {
        return allowCredentials;
    }

    public Long getMaxAge() {
        return maxAge;
    }

    // ─── Apply permit default values ─────────────────────────

    /**
     * Apply permissive defaults for any setting that is not explicitly configured.
     * Called after processing a {@code @CrossOrigin} annotation.
     *
     * Maps to: {@code CorsConfiguration.applyPermitDefaultValues()} (line 321)
     *
     * Defaults:
     * <ul>
     *   <li>Origins: ["*"] (all origins allowed)</li>
     *   <li>Methods: ["GET", "HEAD", "POST"]</li>
     *   <li>Headers: ["*"] (all headers allowed)</li>
     *   <li>Max age: 1800 seconds (30 minutes)</li>
     * </ul>
     *
     * @return this configuration (for chaining)
     */
    public CorsConfiguration applyPermitDefaultValues() {
        if (this.allowedOrigins == null) {
            this.allowedOrigins = new ArrayList<>(List.of(ALL));
        }
        if (this.allowedMethods == null) {
            this.allowedMethods = new ArrayList<>(List.of("GET", "HEAD", "POST"));
        }
        if (this.allowedHeaders == null) {
            this.allowedHeaders = new ArrayList<>(List.of(ALL));
        }
        if (this.maxAge == null) {
            this.maxAge = DEFAULT_MAX_AGE;
        }
        return this;
    }

    // ─── Combine (merge two configurations) ─────────────────

    /**
     * Merge another configuration into this one. List values are combined
     * additively; scalar values use "other overrides this".
     *
     * Maps to: {@code CorsConfiguration.combine(CorsConfiguration)} (line 367)
     *
     * This is the key method for composing handler-level and global CORS
     * configurations. In the real framework, global config is the base
     * and handler config is overlaid:
     * {@code globalConfig.combine(handlerConfig)}
     *
     * @param other the configuration to merge (may be null)
     * @return a new merged CorsConfiguration
     */
    public CorsConfiguration combine(CorsConfiguration other) {
        if (other == null) {
            return this;
        }
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(combineLists(this.allowedOrigins, other.allowedOrigins));
        config.setAllowedMethods(combineLists(this.allowedMethods, other.allowedMethods));
        config.setAllowedHeaders(combineLists(this.allowedHeaders, other.allowedHeaders));
        config.setExposedHeaders(combineLists(this.exposedHeaders, other.exposedHeaders));
        config.setAllowCredentials(other.allowCredentials != null
                ? other.allowCredentials : this.allowCredentials);
        config.setMaxAge(other.maxAge != null ? other.maxAge : this.maxAge);
        return config;
    }

    private List<String> combineLists(List<String> source, List<String> other) {
        if (source == null && other == null) {
            return null;
        }
        if (source == null) {
            return new ArrayList<>(other);
        }
        if (other == null) {
            return new ArrayList<>(source);
        }
        // If either list contains ALL, the result is ALL
        if (source.contains(ALL) || other.contains(ALL)) {
            return new ArrayList<>(List.of(ALL));
        }
        List<String> combined = new ArrayList<>(source);
        for (String value : other) {
            if (!combined.contains(value)) {
                combined.add(value);
            }
        }
        return combined;
    }

    // ─── Check methods (validate request values) ─────────────

    /**
     * Check the origin of the request against the allowed origins.
     *
     * Maps to: {@code CorsConfiguration.checkOrigin(String)} (line 422)
     *
     * @param origin the Origin header value
     * @return the origin to use in the Access-Control-Allow-Origin header,
     *         or null if the origin is not allowed
     */
    public String checkOrigin(String origin) {
        if (origin == null) {
            return null;
        }
        if (allowedOrigins == null) {
            return null;
        }
        if (allowedOrigins.contains(ALL)) {
            // When credentials are enabled, can't use "*" — echo back the origin
            if (Boolean.TRUE.equals(allowCredentials)) {
                return origin;
            }
            return ALL;
        }
        for (String allowed : allowedOrigins) {
            if (origin.equalsIgnoreCase(allowed)) {
                return origin;
            }
        }
        return null;
    }

    /**
     * Check the HTTP method against the allowed methods.
     *
     * Maps to: {@code CorsConfiguration.checkHttpMethod(HttpMethod)} (line 460)
     *
     * @param method the HTTP method (e.g., "GET", "POST")
     * @return the list of allowed methods if the method is allowed, or null to reject
     */
    public List<String> checkHttpMethod(String method) {
        if (method == null) {
            return null;
        }
        if (allowedMethods == null) {
            return null;
        }
        if (allowedMethods.contains(ALL)) {
            return List.of(method);
        }
        for (String allowed : allowedMethods) {
            if (allowed.equalsIgnoreCase(method)) {
                return Collections.unmodifiableList(allowedMethods);
            }
        }
        return null;
    }

    /**
     * Check the request headers against the allowed headers.
     *
     * Maps to: {@code CorsConfiguration.checkHeaders(List)} (line 491)
     *
     * @param headers the Access-Control-Request-Headers values
     * @return the list of allowed headers, or null to reject
     */
    public List<String> checkHeaders(List<String> headers) {
        if (headers == null || headers.isEmpty()) {
            return Collections.emptyList();
        }
        if (allowedHeaders == null) {
            return null;
        }
        if (allowedHeaders.contains(ALL)) {
            return headers;
        }
        List<String> result = new ArrayList<>();
        for (String header : headers) {
            boolean found = false;
            for (String allowed : allowedHeaders) {
                if (header.equalsIgnoreCase(allowed)) {
                    found = true;
                    result.add(header);
                    break;
                }
            }
            if (!found) {
                return null;
            }
        }
        return result;
    }

    /**
     * Validate that when allowCredentials is true, allowedOrigins doesn't
     * contain "*".
     *
     * Maps to: {@code CorsConfiguration.validateAllowCredentials()} (line 280)
     *
     * @throws IllegalArgumentException if the configuration is invalid
     */
    public void validateAllowCredentials() {
        if (Boolean.TRUE.equals(allowCredentials)
                && allowedOrigins != null && allowedOrigins.contains(ALL)) {
            throw new IllegalArgumentException(
                    "When allowCredentials is true, allowedOrigins cannot contain '*'. "
                            + "Use specific origins instead.");
        }
    }

    @Override
    public String toString() {
        return "CorsConfiguration{" +
                "allowedOrigins=" + allowedOrigins +
                ", allowedMethods=" + allowedMethods +
                ", allowedHeaders=" + allowedHeaders +
                ", exposedHeaders=" + exposedHeaders +
                ", allowCredentials=" + allowCredentials +
                ", maxAge=" + maxAge +
                '}';
    }
}
```

#### File: `src/main/java/com/simplespringmvc/cors/CorsProcessor.java` [NEW]

```java
package com.simplespringmvc.cors;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

/**
 * Strategy interface for processing CORS requests — validates the request
 * against a {@link CorsConfiguration} and writes the appropriate CORS
 * response headers.
 *
 * Maps to: {@code org.springframework.web.cors.CorsProcessor}
 *
 * This is the enforcement side of CORS — it is separated from the
 * configuration side ({@code CorsConfiguration}) so that custom
 * implementations can override validation logic.
 *
 * Simplifications:
 * <ul>
 *   <li>Same interface — no simplification needed, it's already minimal</li>
 * </ul>
 */
public interface CorsProcessor {

    /**
     * Process a request given a {@link CorsConfiguration}.
     *
     * Maps to: {@code CorsProcessor.processRequest()} (line 47)
     *
     * @param configuration the CORS configuration for the matched handler
     *                       (may be null, meaning no CORS config applies)
     * @param request       the current HTTP request
     * @param response      the current HTTP response
     * @return {@code false} if the request is rejected (403); {@code true} to proceed
     * @throws IOException in case of I/O errors
     */
    boolean processRequest(CorsConfiguration configuration,
                           HttpServletRequest request,
                           HttpServletResponse response) throws IOException;
}
```

#### File: `src/main/java/com/simplespringmvc/cors/DefaultCorsProcessor.java` [NEW]

```java
package com.simplespringmvc.cors;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

/**
 * Default implementation of {@link CorsProcessor} that applies the W3C CORS
 * specification logic: validates origin/method/headers against a
 * {@link CorsConfiguration} and writes the CORS response headers.
 *
 * Maps to: {@code org.springframework.web.cors.DefaultCorsProcessor}
 *
 * <h3>Processing flow:</h3>
 * <pre>
 *   1. If no CorsConfiguration → skip (allow the request)
 *   2. Add Vary headers (Origin, Access-Control-Request-Method, Access-Control-Request-Headers)
 *   3. If not a CORS request (no Origin header, or same-origin) → skip
 *   4. If response already has Access-Control-Allow-Origin → skip (no double-processing)
 *   5. Validate origin → reject with 403 if not allowed
 *   6. Validate method → reject with 403 if not allowed
 *   7. Validate headers (preflight only) → reject with 403 if not allowed
 *   8. Write CORS response headers
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No allowPrivateNetwork support</li>
 *   <li>No custom rejection handler — always writes 403 + "Invalid CORS request"</li>
 * </ul>
 */
public class DefaultCorsProcessor implements CorsProcessor {

    /**
     * Process a CORS request.
     *
     * Maps to: {@code DefaultCorsProcessor.processRequest()} (line 72)
     */
    @Override
    public boolean processRequest(CorsConfiguration config,
                                  HttpServletRequest request,
                                  HttpServletResponse response) throws IOException {

        // Step 1: No config → skip CORS processing (allow through)
        if (config == null) {
            return true;
        }

        // Step 2: Always add Vary headers so caches know the response
        // varies by these request headers.
        // Maps to: DefaultCorsProcessor.processRequest() (line 85)
        addVaryHeaders(response);

        // Step 3: If not a CORS request, nothing to do
        if (!CorsUtils.isCorsRequest(request)) {
            return true;
        }

        // Step 4: If response already has Access-Control-Allow-Origin, skip
        if (response.getHeader("Access-Control-Allow-Origin") != null) {
            return true;
        }

        // Step 5-8: Handle the CORS validation and header writing
        boolean isPreFlight = CorsUtils.isPreFlightRequest(request);
        return handleInternal(config, request, response, isPreFlight);
    }

    /**
     * Internal CORS handling — validate and write headers.
     *
     * Maps to: {@code DefaultCorsProcessor.handleInternal()} (line 127)
     */
    protected boolean handleInternal(CorsConfiguration config,
                                     HttpServletRequest request,
                                     HttpServletResponse response,
                                     boolean isPreFlight) throws IOException {

        String origin = request.getHeader("Origin");

        // Step 5: Validate origin
        String allowedOrigin = config.checkOrigin(origin);
        if (allowedOrigin == null) {
            rejectRequest(response);
            return false;
        }

        // Step 6: Validate HTTP method
        String requestMethod = isPreFlight
                ? request.getHeader("Access-Control-Request-Method")
                : request.getMethod();
        List<String> allowedMethods = config.checkHttpMethod(requestMethod);
        if (allowedMethods == null) {
            rejectRequest(response);
            return false;
        }

        // Step 7: Validate headers (preflight only)
        List<String> requestHeaders = getAccessControlRequestHeaders(request);
        List<String> allowedHeaders = null;
        if (isPreFlight) {
            allowedHeaders = config.checkHeaders(requestHeaders);
            if (requestHeaders != null && !requestHeaders.isEmpty() && allowedHeaders == null) {
                rejectRequest(response);
                return false;
            }
        }

        // Step 8: Write CORS response headers
        response.setHeader("Access-Control-Allow-Origin", allowedOrigin);

        if (isPreFlight) {
            response.setHeader("Access-Control-Allow-Methods",
                    String.join(", ", allowedMethods));
        }

        if (isPreFlight && allowedHeaders != null && !allowedHeaders.isEmpty()) {
            response.setHeader("Access-Control-Allow-Headers",
                    String.join(", ", allowedHeaders));
        }

        if (config.getExposedHeaders() != null && !config.getExposedHeaders().isEmpty()) {
            response.setHeader("Access-Control-Expose-Headers",
                    String.join(", ", config.getExposedHeaders()));
        }

        if (Boolean.TRUE.equals(config.getAllowCredentials())) {
            response.setHeader("Access-Control-Allow-Credentials", "true");
        }

        if (isPreFlight && config.getMaxAge() != null) {
            response.setHeader("Access-Control-Max-Age",
                    config.getMaxAge().toString());
        }

        return true;
    }

    /**
     * Reject a CORS request — set 403 status and write an error message.
     *
     * Maps to: {@code DefaultCorsProcessor.rejectRequest()} (line 113)
     */
    protected void rejectRequest(HttpServletResponse response) throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.getWriter().write("Invalid CORS request");
        response.getWriter().flush();
    }

    /**
     * Add Vary headers so HTTP caches know the response may vary based
     * on these request headers.
     */
    private void addVaryHeaders(HttpServletResponse response) {
        addVaryHeader(response, "Origin");
        addVaryHeader(response, "Access-Control-Request-Method");
        addVaryHeader(response, "Access-Control-Request-Headers");
    }

    private void addVaryHeader(HttpServletResponse response, String headerName) {
        String existing = response.getHeader("Vary");
        if (existing == null || existing.isEmpty()) {
            response.setHeader("Vary", headerName);
        } else if (!existing.contains(headerName)) {
            response.setHeader("Vary", existing + ", " + headerName);
        }
    }

    /**
     * Parse the Access-Control-Request-Headers header into a list.
     */
    private List<String> getAccessControlRequestHeaders(HttpServletRequest request) {
        String headerValue = request.getHeader("Access-Control-Request-Headers");
        if (headerValue == null || headerValue.isEmpty()) {
            return List.of();
        }
        return Arrays.stream(headerValue.split(","))
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .toList();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/cors/CorsUtils.java` [NEW]

```java
package com.simplespringmvc.cors;

import jakarta.servlet.http.HttpServletRequest;

import java.net.URI;

/**
 * Utility class with static methods for classifying HTTP requests as
 * CORS-related. Used by the handler mapping to decide whether to activate
 * CORS processing.
 *
 * Maps to: {@code org.springframework.web.cors.CorsUtils}
 *
 * The real class uses {@code UriComponentsBuilder} for parsing the Origin
 * header and handles default ports (80 for http/ws, 443 for https/wss).
 * We use {@code java.net.URI} for simplicity.
 *
 * Simplifications:
 * <ul>
 *   <li>No default port normalization (80/443)</li>
 *   <li>No WebSocket scheme handling (ws/wss)</li>
 *   <li>Simpler same-origin comparison</li>
 * </ul>
 */
public abstract class CorsUtils {

    private CorsUtils() {
    }

    /**
     * Check if the request is a CORS request — i.e., it has an Origin header
     * that differs from the request's own scheme/host/port.
     *
     * Maps to: {@code CorsUtils.isCorsRequest(HttpServletRequest)} (line 43)
     *
     * The real implementation uses {@code UriComponentsBuilder.fromOriginHeader()}
     * and compares scheme, host, and port (with default port normalization).
     *
     * @param request the current HTTP request
     * @return true if this is a cross-origin request
     */
    public static boolean isCorsRequest(HttpServletRequest request) {
        String origin = request.getHeader("Origin");
        if (origin == null || origin.isEmpty()) {
            return false;
        }
        try {
            URI originUri = new URI(origin);
            String scheme = request.getScheme();
            String host = request.getServerName();
            int port = request.getServerPort();

            // Compare the origin to the request's own address
            boolean sameScheme = scheme.equalsIgnoreCase(originUri.getScheme());
            boolean sameHost = host.equalsIgnoreCase(originUri.getHost());
            boolean samePort = (port == getPort(originUri, originUri.getScheme()));

            return !(sameScheme && sameHost && samePort);
        } catch (Exception e) {
            // Malformed origin — treat as cross-origin for safety
            return true;
        }
    }

    /**
     * Check if the request is a CORS pre-flight request — an OPTIONS request
     * with both Origin and Access-Control-Request-Method headers.
     *
     * Maps to: {@code CorsUtils.isPreFlightRequest(HttpServletRequest)} (line 73)
     *
     * Preflight requests are sent by browsers before actual cross-origin
     * requests that use non-simple methods or headers. The server must
     * respond with the appropriate CORS headers to allow the actual request.
     *
     * @param request the current HTTP request
     * @return true if this is a preflight OPTIONS request
     */
    public static boolean isPreFlightRequest(HttpServletRequest request) {
        return "OPTIONS".equalsIgnoreCase(request.getMethod())
                && request.getHeader("Origin") != null
                && request.getHeader("Access-Control-Request-Method") != null;
    }

    private static int getPort(URI uri, String scheme) {
        int port = uri.getPort();
        if (port == -1) {
            if ("https".equalsIgnoreCase(scheme) || "wss".equalsIgnoreCase(scheme)) {
                return 443;
            }
            return 80;
        }
        return port;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/cors/CorsInterceptor.java` [NEW]

```java
package com.simplespringmvc.cors;

import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

/**
 * A {@link HandlerInterceptor} that delegates CORS processing to a
 * {@link CorsProcessor}. Inserted at position 0 in the execution chain
 * for CORS requests so that CORS validation happens before the handler.
 *
 * Maps to: {@code AbstractHandlerMapping.CorsInterceptor} (inner class, line 761)
 *
 * In the real framework this is a private inner class of AbstractHandlerMapping.
 * We promote it to a top-level class for clarity.
 *
 * For preflight requests, this interceptor is attached to a no-op handler
 * (a simple HandlerMethod that does nothing). The interceptor writes all
 * necessary CORS headers, and the no-op handler is invoked but does nothing.
 */
public class CorsInterceptor implements HandlerInterceptor {

    private final CorsConfiguration corsConfiguration;
    private final CorsProcessor corsProcessor;

    public CorsInterceptor(CorsConfiguration corsConfiguration) {
        this(corsConfiguration, new DefaultCorsProcessor());
    }

    public CorsInterceptor(CorsConfiguration corsConfiguration, CorsProcessor corsProcessor) {
        this.corsConfiguration = corsConfiguration;
        this.corsProcessor = corsProcessor;
    }

    /**
     * Apply CORS validation before the handler executes.
     *
     * Maps to: {@code CorsInterceptor.preHandle()} (line 774)
     *
     * If the CORS processor rejects the request (returns false), this
     * interceptor returns false to stop the chain — the 403 response
     * has already been written by the processor.
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             HandlerMethod handler) throws Exception {
        try {
            return corsProcessor.processRequest(corsConfiguration, request, response);
        } catch (IOException e) {
            throw new RuntimeException("CORS processing failed", e);
        }
    }

    public CorsConfiguration getCorsConfiguration() {
        return corsConfiguration;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/cors/CorsRegistry.java` [NEW]

```java
package com.simplespringmvc.cors;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * A fluent API for registering global CORS configuration mappings by URL
 * pattern. Used programmatically or by a configuration callback.
 *
 * Maps to: {@code org.springframework.web.servlet.config.annotation.CorsRegistry}
 *
 * In the real framework, this is used via {@code WebMvcConfigurer.addCorsMappings(CorsRegistry)}:
 * <pre>
 *   registry.addMapping("/api/**")
 *       .allowedOrigins("https://example.com")
 *       .allowedMethods("GET", "POST")
 *       .maxAge(3600);
 * </pre>
 *
 * The resulting configurations are fed to
 * {@code AbstractHandlerMapping.setCorsConfigurations()}, which wraps them
 * in a {@code UrlBasedCorsConfigurationSource}.
 *
 * Simplifications:
 * <ul>
 *   <li>No URL pattern matching — path is stored as-is for exact or simple
 *       prefix matching in the handler mapping</li>
 * </ul>
 */
public class CorsRegistry {

    private final List<CorsRegistration> registrations = new ArrayList<>();

    /**
     * Enable cross-origin request handling for the specified path pattern.
     *
     * Maps to: {@code CorsRegistry.addMapping(String)} (line 50)
     *
     * @param pathPattern the path pattern (e.g., "/api/**", "/**")
     * @return the CorsRegistration for fluent configuration
     */
    public CorsRegistration addMapping(String pathPattern) {
        CorsRegistration registration = new CorsRegistration(pathPattern);
        this.registrations.add(registration);
        return registration;
    }

    /**
     * Return the registered CORS configurations keyed by path pattern.
     *
     * Maps to: {@code CorsRegistry.getCorsConfigurations()} (line 67)
     *
     * @return Map of path pattern → CorsConfiguration
     */
    public Map<String, CorsConfiguration> getCorsConfigurations() {
        Map<String, CorsConfiguration> configs = new LinkedHashMap<>();
        for (CorsRegistration registration : registrations) {
            configs.put(registration.getPathPattern(), registration.getCorsConfiguration());
        }
        return configs;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/cors/CorsRegistration.java` [NEW]

```java
package com.simplespringmvc.cors;

import java.util.Arrays;

/**
 * A fluent builder that creates a {@link CorsConfiguration} for a single URL
 * path pattern. Initialized with {@code applyPermitDefaultValues()} so it
 * starts permissive — each setter overrides the defaults.
 *
 * Maps to: {@code org.springframework.web.servlet.config.annotation.CorsRegistration}
 *
 * Usage:
 * <pre>
 *   registry.addMapping("/api/**")
 *       .allowedOrigins("https://example.com")
 *       .allowedMethods("GET", "POST")
 *       .allowCredentials(true)
 *       .maxAge(3600);
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No originPatterns support</li>
 *   <li>No allowPrivateNetwork</li>
 * </ul>
 */
public class CorsRegistration {

    private final String pathPattern;
    private final CorsConfiguration config;

    public CorsRegistration(String pathPattern) {
        this.pathPattern = pathPattern;
        this.config = new CorsConfiguration().applyPermitDefaultValues();
    }

    /**
     * Set the allowed origins. Replaces the defaults.
     *
     * @param origins one or more origins (e.g., "https://example.com") or "*" for all
     * @return this registration for chaining
     */
    public CorsRegistration allowedOrigins(String... origins) {
        this.config.setAllowedOrigins(Arrays.asList(origins));
        return this;
    }

    /**
     * Set the allowed HTTP methods. Replaces the defaults.
     *
     * @param methods HTTP methods (e.g., "GET", "POST", "PUT")
     * @return this registration for chaining
     */
    public CorsRegistration allowedMethods(String... methods) {
        this.config.setAllowedMethods(Arrays.asList(methods));
        return this;
    }

    /**
     * Set the allowed request headers. Replaces the defaults.
     *
     * @param headers header names or "*" for all
     * @return this registration for chaining
     */
    public CorsRegistration allowedHeaders(String... headers) {
        this.config.setAllowedHeaders(Arrays.asList(headers));
        return this;
    }

    /**
     * Set the response headers exposed to the client.
     *
     * @param headers header names to expose
     * @return this registration for chaining
     */
    public CorsRegistration exposedHeaders(String... headers) {
        this.config.setExposedHeaders(Arrays.asList(headers));
        return this;
    }

    /**
     * Whether user credentials (cookies) are supported.
     * When true, allowedOrigins cannot contain "*".
     *
     * @param allowCredentials true to allow credentials
     * @return this registration for chaining
     */
    public CorsRegistration allowCredentials(boolean allowCredentials) {
        this.config.setAllowCredentials(allowCredentials);
        return this;
    }

    /**
     * Set the max age (in seconds) for the preflight response cache.
     *
     * @param maxAge max age in seconds
     * @return this registration for chaining
     */
    public CorsRegistration maxAge(long maxAge) {
        this.config.setMaxAge(maxAge);
        return this;
    }

    /**
     * Return the path pattern for this registration.
     */
    public String getPathPattern() {
        return pathPattern;
    }

    /**
     * Return the built CorsConfiguration.
     */
    public CorsConfiguration getCorsConfiguration() {
        return this.config;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java` [MODIFIED]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.CrossOrigin;
import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.cors.CorsConfiguration;
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
 * <h3>ch19 Enhancement:</h3>
 * During handler method detection, {@code @CrossOrigin} annotations are read from
 * both the class and method levels. A {@link CorsConfiguration} is built per handler
 * method and stored for later retrieval. Global CORS configurations (from a
 * {@link com.simplespringmvc.cors.CorsRegistry}) can also be set and are merged
 * with handler-level configs at lookup time.
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
     * ch19: Per-handler CORS configurations extracted from @CrossOrigin annotations.
     * Keyed by HandlerMethod identity (bean + method).
     *
     * Maps to: {@code AbstractHandlerMethodMapping.corsLookup} (line 93)
     * The real field is a {@code Map<HandlerMethod, CorsConfiguration>}.
     */
    private final Map<HandlerMethod, CorsConfiguration> corsConfigurations = new LinkedHashMap<>();

    /**
     * ch19: Global CORS configurations keyed by path pattern.
     * Set via {@link #setCorsConfigurations(Map)} from a CorsRegistry.
     *
     * Maps to: {@code AbstractHandlerMapping.corsConfigurationSource} (line 101)
     */
    private Map<String, CorsConfiguration> globalCorsConfigurations;

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
     *
     * <h3>ch19 Enhancement:</h3>
     * After registering a handler method, checks for {@code @CrossOrigin} on the
     * class and method, builds a {@link CorsConfiguration}, and stores it keyed
     * by the handler method. This mirrors {@code RequestMappingHandlerMapping.initCorsConfiguration()}.
     */
    private void detectHandlerMethods(Object bean) {
        Class<?> beanType = bean.getClass();

        // Check for type-level @RequestMapping (base path prefix)
        String basePath = "";
        RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
        if (typeMapping != null) {
            basePath = typeMapping.path();
        }

        // ch19: Read type-level @CrossOrigin (applies to all handler methods in this class)
        CrossOrigin typeCrossOrigin = beanType.getAnnotation(CrossOrigin.class);

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

            // ch19: Extract @CrossOrigin and store CORS config for this handler
            CorsConfiguration corsConfig = initCorsConfiguration(
                    typeCrossOrigin, method.getAnnotation(CrossOrigin.class), httpMethod);
            if (corsConfig != null) {
                corsConfigurations.put(handlerMethod, corsConfig);
            }
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

    // ─── CORS support (ch19) ────────────────────────────────────

    /**
     * Build a {@link CorsConfiguration} from type-level and method-level
     * {@code @CrossOrigin} annotations.
     *
     * Maps to: {@code RequestMappingHandlerMapping.initCorsConfiguration()} (line 492)
     *
     * The type-level annotation values are applied first, then method-level
     * values are added additively. After merging, {@code applyPermitDefaultValues()}
     * fills in any remaining gaps. If no methods are explicitly declared in
     * the annotation, the handler's mapped HTTP method is used as the default.
     *
     * @param typeAnnotation   the @CrossOrigin on the class (may be null)
     * @param methodAnnotation the @CrossOrigin on the method (may be null)
     * @param mappedHttpMethod the HTTP method this handler is mapped to
     * @return a CorsConfiguration, or null if neither annotation is present
     */
    private CorsConfiguration initCorsConfiguration(CrossOrigin typeAnnotation,
                                                     CrossOrigin methodAnnotation,
                                                     String mappedHttpMethod) {
        if (typeAnnotation == null && methodAnnotation == null) {
            return null;
        }

        CorsConfiguration config = new CorsConfiguration();
        updateCorsConfig(config, typeAnnotation);
        updateCorsConfig(config, methodAnnotation);

        // If no methods explicitly set, default to the mapped HTTP method
        if (config.getAllowedMethods() == null || config.getAllowedMethods().isEmpty()) {
            if (mappedHttpMethod != null && !mappedHttpMethod.isEmpty()) {
                config.addAllowedMethod(mappedHttpMethod);
            }
        }

        return config.applyPermitDefaultValues();
    }

    /**
     * Apply values from a {@code @CrossOrigin} annotation to a CorsConfiguration.
     *
     * Maps to: {@code RequestMappingHandlerMapping.updateCorsConfig()} (line 514)
     *
     * Values are added (not replaced) so type-level and method-level annotations
     * combine additively.
     */
    private void updateCorsConfig(CorsConfiguration config, CrossOrigin annotation) {
        if (annotation == null) {
            return;
        }
        for (String origin : annotation.origins()) {
            config.addAllowedOrigin(origin);
        }
        for (String method : annotation.methods()) {
            config.addAllowedMethod(method);
        }
        for (String header : annotation.allowedHeaders()) {
            config.addAllowedHeader(header);
        }
        for (String header : annotation.exposedHeaders()) {
            config.addExposedHeader(header);
        }
        if (annotation.allowCredentials()) {
            config.setAllowCredentials(true);
        }
        if (annotation.maxAge() >= 0) {
            config.setMaxAge(annotation.maxAge());
        }
    }

    /**
     * Get the CORS configuration for a specific handler method.
     * Used by the dispatcher to decide whether to add a CorsInterceptor.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.getCorsConfiguration()} (line 478)
     *
     * @param handlerMethod the handler method
     * @return the CorsConfiguration, or null if no CORS config for this handler
     */
    public CorsConfiguration getCorsConfiguration(HandlerMethod handlerMethod) {
        return corsConfigurations.get(handlerMethod);
    }

    /**
     * Check if any CORS configuration exists for a handler method
     * (either from @CrossOrigin or from global configuration).
     *
     * Maps to: {@code AbstractHandlerMapping.hasCorsConfigurationSource()} (line 548)
     */
    public boolean hasCorsConfiguration(HandlerMethod handlerMethod) {
        return corsConfigurations.containsKey(handlerMethod)
                || (globalCorsConfigurations != null && !globalCorsConfigurations.isEmpty());
    }

    /**
     * Set global CORS configurations keyed by path pattern.
     * These are merged with handler-level configs at request time.
     *
     * Maps to: {@code AbstractHandlerMapping.setCorsConfigurations(Map)} (line 227)
     *
     * @param corsConfigurations map of path pattern → CorsConfiguration
     */
    public void setCorsConfigurations(Map<String, CorsConfiguration> corsConfigurations) {
        this.globalCorsConfigurations = corsConfigurations;
    }

    /**
     * Get the global CORS configuration that matches the given request path.
     * Uses simple prefix matching for "/**" patterns.
     *
     * Maps to: {@code UrlBasedCorsConfigurationSource.getCorsConfiguration()} (line 109)
     *
     * @param requestPath the request URI path
     * @return the matching global CorsConfiguration, or null
     */
    public CorsConfiguration getGlobalCorsConfiguration(String requestPath) {
        if (globalCorsConfigurations == null) {
            return null;
        }
        for (Map.Entry<String, CorsConfiguration> entry : globalCorsConfigurations.entrySet()) {
            String pattern = entry.getKey();
            if (pathMatchesPattern(requestPath, pattern)) {
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Simple path pattern matching for CORS global configurations.
     * Supports "/**" (match all) and "/prefix/**" (prefix match).
     */
    private boolean pathMatchesPattern(String path, String pattern) {
        if ("/**".equals(pattern)) {
            return true;
        }
        if (pattern.endsWith("/**")) {
            String prefix = pattern.substring(0, pattern.length() - 3);
            return path.startsWith(prefix);
        }
        return path.equals(pattern);
    }

    // ─── Utilities ──────────────────────────────────────────────

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
import com.simplespringmvc.adapter.RedirectAttributesArgumentResolver;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.cors.CorsConfiguration;
import com.simplespringmvc.cors.CorsInterceptor;
import com.simplespringmvc.cors.CorsProcessor;
import com.simplespringmvc.cors.CorsRegistry;
import com.simplespringmvc.cors.CorsUtils;
import com.simplespringmvc.cors.DefaultCorsProcessor;
import com.simplespringmvc.exception.ExceptionHandlerExceptionResolver;
import com.simplespringmvc.exception.HandlerExceptionResolver;
import com.simplespringmvc.flash.FlashMap;
import com.simplespringmvc.flash.FlashMapManager;
import com.simplespringmvc.flash.SimpleRedirectAttributes;
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
import java.util.Collections;
import java.util.List;
import java.util.Map;

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
 *   <li>ch18: flash map lifecycle and redirect handling ✓</li>
 *   <li>ch19: CORS processing in getHandler() ✓</li>
 * </ul>
 *
 * <h3>ch18 Enhancement:</h3>
 * The dispatch pipeline now includes flash map management and redirect support:
 * <pre>
 *   0. Flash map lifecycle: retrieve input flash map, create output flash map
 *   1. getHandler(request) → HandlerExecutionChain
 *   2. chain.applyPreHandle()
 *   3. ha.handle() → returns ModelAndView (null if response handled directly)
 *   4. chain.applyPostHandle()
 *   5. chain.triggerAfterCompletion() (always)
 *   6. If exception → processHandlerException()
 *   7. If ModelAndView non-null:
 *      7a. If view name starts with "redirect:" → handle redirect + save flash map
 *      7b. Otherwise → resolve and render view (merge input flash attrs into model)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No locale/theme resolution (added in ch20)</li>
 *   <li>Falls back to text/plain when view can't be resolved (real framework throws)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    /**
     * Request attribute under which the input FlashMap is stored.
     *
     * Maps to: {@code DispatcherServlet.INPUT_FLASH_MAP_ATTRIBUTE} (line 221)
     */
    public static final String INPUT_FLASH_MAP_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".INPUT_FLASH_MAP";

    /**
     * Request attribute under which the output FlashMap is stored.
     *
     * Maps to: {@code DispatcherServlet.OUTPUT_FLASH_MAP_ATTRIBUTE} (line 228)
     */
    public static final String OUTPUT_FLASH_MAP_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".OUTPUT_FLASH_MAP";

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

    /**
     * ch18: FlashMapManager for managing flash attributes across redirects.
     *
     * Maps to: {@code DispatcherServlet.flashMapManager} (line 200)
     * The real version is initialized from a FlashMapManager bean in the context,
     * defaulting to SessionFlashMapManager.
     */
    private FlashMapManager flashMapManager;

    /**
     * ch19: CORS processor for validating and writing CORS response headers.
     * Defaults to {@link DefaultCorsProcessor}.
     *
     * Maps to: {@code AbstractHandlerMapping.corsProcessor} (line 99)
     */
    private CorsProcessor corsProcessor = new DefaultCorsProcessor();

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
        // ch18: FlashMapManager is set programmatically via setFlashMapManager()
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
     * <h3>ch18 Enhancement:</h3>
     * The dispatch flow now includes flash map management:
     * <pre>
     *   0. Flash map lifecycle:
     *      - Retrieve input flash map (from previous redirect)
     *      - Create output flash map (for possible redirect in this request)
     *   1. mappedHandler = getHandler(request)
     *   2. mappedHandler.applyPreHandle()
     *   3. mv = ha.handle(request, response, handler) ← now manages session attrs
     *   4. mappedHandler.applyPostHandle()
     *   5. mappedHandler.triggerAfterCompletion() (always)
     *   6. processHandlerException() if exception
     *   7. If view name starts with "redirect:" → handleRedirect() + save flash map
     *      Otherwise → render view (merge input flash attrs into model)
     * </pre>
     *
     * The flash map lifecycle mirrors the real DispatcherServlet.doService()
     * (lines 850-857) which sets up INPUT/OUTPUT flash maps as request attributes
     * before doDispatch() is called.
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerExecutionChain mappedHandler = null;
        ModelAndView mv = null;
        Exception dispatchException = null;

        // ch18: Step 0 — Flash map lifecycle: retrieve input, create output
        FlashMap inputFlashMap = null;
        if (flashMapManager != null) {
            inputFlashMap = flashMapManager.retrieveAndUpdate(request, response);
            if (inputFlashMap != null) {
                request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE,
                        Collections.unmodifiableMap(inputFlashMap));
            }
            request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        }

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
            // ch18: handle() now manages session attributes lifecycle internally.
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
            mv = adapter.handle(request, response, mappedHandler.getHandler());

            // Step 4: Apply interceptor postHandle — reverse order
            mappedHandler.applyPostHandle(request, response);

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

        // Step 7 (ch13/ch18): View resolution, rendering, or redirect.
        if (mv != null && mv.getViewName() != null) {
            String viewName = mv.getViewName();

            // ch18: Handle "redirect:" prefix
            if (viewName.startsWith("redirect:")) {
                String redirectUrl = viewName.substring("redirect:".length());
                handleRedirect(redirectUrl, request, response);
            } else {
                // ch18: Merge input flash attributes into model for view access
                if (inputFlashMap != null && !inputFlashMap.isEmpty()) {
                    mv.getModel().putAll(inputFlashMap);
                }
                render(mv, request, response);
            }
        }
    }

    /**
     * Handle a redirect by saving flash attributes and sending the redirect response.
     *
     * <h3>ch18 Enhancement:</h3>
     * Maps to the redirect handling in {@code DispatcherServlet.processDispatchResult()}
     * and {@code RequestContextUtils.saveOutputFlashMap()} (line 218).
     *
     * The flow:
     * <ol>
     *   <li>Get the output FlashMap (created at request start)</li>
     *   <li>Copy flash attributes from RedirectAttributes (if a handler used it)</li>
     *   <li>Set the target request path on the FlashMap</li>
     *   <li>Save the FlashMap via FlashMapManager</li>
     *   <li>Send the HTTP redirect response</li>
     * </ol>
     *
     * @param redirectUrl the target URL to redirect to
     * @param request     current HTTP request
     * @param response    current HTTP response
     */
    private void handleRedirect(String redirectUrl, HttpServletRequest request,
                                 HttpServletResponse response) throws IOException {
        if (flashMapManager != null) {
            FlashMap outputFlashMap =
                    (FlashMap) request.getAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE);

            // Copy flash attributes from RedirectAttributes if the handler used it
            SimpleRedirectAttributes redirectAttrs = (SimpleRedirectAttributes)
                    request.getAttribute(RedirectAttributesArgumentResolver.REDIRECT_ATTRIBUTES_ATTRIBUTE);
            if (redirectAttrs != null && !redirectAttrs.getFlashAttributes().isEmpty()) {
                outputFlashMap.putAll(redirectAttrs.getFlashAttributes());
            }

            // Set target path so the FlashMap matches the redirect target
            outputFlashMap.setTargetRequestPath(redirectUrl);

            // Save to session for the next request to retrieve
            flashMapManager.saveOutputFlashMap(outputFlashMap, request, response);
        }

        response.sendRedirect(redirectUrl);
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

    /**
     * Look up the handler for this request and build the execution chain.
     *
     * <h3>ch19 Enhancement:</h3>
     * After resolving the handler and building the initial chain, checks for
     * CORS configuration. If this is a CORS request (or preflight), a
     * {@link CorsInterceptor} is inserted at position 0 in the chain.
     *
     * For preflight requests where no handler matches (because there's no
     * OPTIONS mapping), we create a no-op handler and rely on the CORS
     * interceptor to write the response headers.
     *
     * Maps to: {@code AbstractHandlerMapping.getHandler()} (line 542)
     * — specifically the CORS block at lines 569-588.
     */
    private HandlerExecutionChain getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        HandlerMethod handler = handlerMapping.lookupHandler(request);

        // ch19: Handle preflight requests that have no explicit OPTIONS handler
        if (handler == null && CorsUtils.isPreFlightRequest(request)) {
            // Look up the handler for the *actual* method in the preflight
            // Access-Control-Request-Method header to get its CORS config
            HandlerMethod actualHandler = findPreflightHandler(request);
            if (actualHandler != null) {
                // Create a no-op handler for the preflight — the CorsInterceptor
                // will write the CORS headers, and this handler does nothing.
                HandlerMethod noOpHandler = createNoOpHandler();
                HandlerExecutionChain chain = new HandlerExecutionChain(noOpHandler, interceptors);
                applyCorsProcessing(chain, actualHandler, request);
                return chain;
            }
            // No CORS config found — fall through to return null (404)
            return null;
        }

        if (handler == null) {
            return null;
        }

        HandlerExecutionChain chain = new HandlerExecutionChain(handler, interceptors);

        // ch19: Apply CORS processing for actual CORS requests
        if (CorsUtils.isCorsRequest(request)
                || handlerMapping.hasCorsConfiguration(handler)) {
            applyCorsProcessing(chain, handler, request);
        }

        return chain;
    }

    /**
     * For a preflight request, find the handler that would handle the
     * actual request method (from Access-Control-Request-Method header).
     *
     * Maps to: The real framework does this implicitly because handler
     * mappings register CORS configs keyed by HandlerMethod, and the
     * handler is found by the requested method, not OPTIONS.
     */
    private HandlerMethod findPreflightHandler(HttpServletRequest request) {
        String actualMethod = request.getHeader("Access-Control-Request-Method");
        if (actualMethod == null) {
            return null;
        }
        return handlerMapping.lookupHandler(request.getRequestURI(), actualMethod);
    }

    /**
     * Create a no-op handler for preflight requests. The handler method
     * does nothing — all CORS work is done by the CorsInterceptor.
     *
     * Maps to: {@code AbstractHandlerMapping.PreFlightHttpRequestHandler} (line 795)
     */
    private HandlerMethod createNoOpHandler() {
        try {
            // Use a method that exists on Object — toString() as a no-op target
            return new HandlerMethod(this, getClass().getMethod("noOpPreflightHandler"));
        } catch (NoSuchMethodException e) {
            throw new IllegalStateException("Cannot create no-op preflight handler", e);
        }
    }

    /**
     * No-op method used as the handler for preflight requests.
     * Must be public so reflection can invoke it.
     */
    public String noOpPreflightHandler() {
        return null; // intentionally does nothing
    }

    /**
     * Merge handler-level and global CORS configs, then insert a CorsInterceptor
     * at position 0 in the execution chain.
     *
     * Maps to: {@code AbstractHandlerMapping.getCorsHandlerExecutionChain()} (line 746)
     *
     * The merging strategy: global config is the base, handler config is overlaid:
     * {@code globalConfig.combine(handlerConfig)}
     */
    private void applyCorsProcessing(HandlerExecutionChain chain,
                                      HandlerMethod handler,
                                      HttpServletRequest request) {
        // Get handler-level config (from @CrossOrigin)
        CorsConfiguration handlerConfig = handlerMapping.getCorsConfiguration(handler);

        // Get global config (from CorsRegistry)
        CorsConfiguration globalConfig = handlerMapping.getGlobalCorsConfiguration(
                request.getRequestURI());

        // Merge: global is base, handler overlays
        CorsConfiguration mergedConfig;
        if (globalConfig != null && handlerConfig != null) {
            mergedConfig = globalConfig.combine(handlerConfig);
        } else if (globalConfig != null) {
            mergedConfig = globalConfig;
        } else if (handlerConfig != null) {
            mergedConfig = handlerConfig;
        } else {
            return; // no CORS config at all
        }

        // Validate
        mergedConfig.validateAllowCredentials();

        // Insert CorsInterceptor at position 0
        chain.addInterceptorFirst(new CorsInterceptor(mergedConfig, corsProcessor));
    }

    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
    }

    // ─── FlashMapManager registration ────────────────────────────────

    /**
     * Set the FlashMapManager for managing flash attributes across redirects.
     *
     * Maps to: In the real framework, the FlashMapManager is initialized from
     * a FlashMapManager bean in the ApplicationContext (defaulting to
     * SessionFlashMapManager). We use programmatic registration for simplicity.
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support flash attributes and the Post/Redirect/Get pattern.
     *
     * @param flashMapManager the manager to use
     */
    public void setFlashMapManager(FlashMapManager flashMapManager) {
        this.flashMapManager = flashMapManager;
    }

    /**
     * Returns the configured FlashMapManager (for testing/inspection).
     */
    public FlashMapManager getFlashMapManager() {
        return flashMapManager;
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

    // ─── CORS registration (ch19) ─────────────────────────────────

    /**
     * Apply global CORS configuration from a {@link CorsRegistry}.
     *
     * Maps to: In the real framework, this is done via
     * {@code WebMvcConfigurer.addCorsMappings(CorsRegistry)}, which eventually
     * calls {@code AbstractHandlerMapping.setCorsConfigurations()}.
     *
     * @param corsRegistry the registry with global CORS mappings
     */
    public void setCorsRegistry(CorsRegistry corsRegistry) {
        if (handlerMapping != null) {
            handlerMapping.setCorsConfigurations(corsRegistry.getCorsConfigurations());
        }
    }

    /**
     * Set a custom CORS processor. Defaults to {@link DefaultCorsProcessor}.
     *
     * Maps to: {@code AbstractHandlerMapping.setCorsProcessor(CorsProcessor)} (line 237)
     *
     * @param corsProcessor the processor to use
     */
    public void setCorsProcessor(CorsProcessor corsProcessor) {
        this.corsProcessor = corsProcessor;
    }

    /**
     * Returns the configured CorsProcessor (for testing/inspection).
     */
    public CorsProcessor getCorsProcessor() {
        return corsProcessor;
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

#### File: `src/main/java/com/simplespringmvc/interceptor/HandlerExecutionChain.java` [MODIFIED]

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.ArrayList;
import java.util.List;

/**
 * Pairs a {@link HandlerMethod} with a list of {@link HandlerInterceptor}s and
 * manages the forward/reverse iteration lifecycle for pre-handle, post-handle,
 * and after-completion callbacks.
 *
 * Maps to: {@code org.springframework.web.servlet.HandlerExecutionChain}
 *
 * <h3>The interceptorIndex bookkeeping:</h3>
 * The critical field is {@code interceptorIndex}, which starts at {@code -1}.
 * It advances only when a {@code preHandle} returns {@code true}. When
 * {@code afterCompletion} runs, it counts DOWN from {@code interceptorIndex},
 * ensuring only interceptors whose {@code preHandle} succeeded get cleanup.
 *
 * <pre>
 *   Interceptors: [A, B, C]
 *
 *   Normal flow:
 *     A.preHandle → true  (interceptorIndex = 0)
 *     B.preHandle → true  (interceptorIndex = 1)
 *     C.preHandle → true  (interceptorIndex = 2)
 *     handler invoked
 *     C.postHandle, B.postHandle, A.postHandle  (reverse of all)
 *     C.afterCompletion, B.afterCompletion, A.afterCompletion  (reverse from index 2)
 *
 *   B vetoes:
 *     A.preHandle → true  (interceptorIndex = 0)
 *     B.preHandle → false (interceptorIndex stays 0)
 *     → handler NOT invoked, C.preHandle NOT called
 *     A.afterCompletion  (reverse from index 0 — only A's preHandle returned true)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No constructor unwrapping of nested HandlerExecutionChains</li>
 *   <li>No async handling ({@code applyAfterConcurrentHandlingStarted})</li>
 *   <li>No ModelAndView parameter on postHandle</li>
 * </ul>
 */
public class HandlerExecutionChain {

    private final HandlerMethod handler;
    private final List<HandlerInterceptor> interceptorList;

    /**
     * Tracks the index of the last interceptor whose {@code preHandle} returned {@code true}.
     * Starts at -1 (no interceptor has run). Used by {@link #triggerAfterCompletion}
     * to ensure cleanup runs only for interceptors that completed pre-processing.
     *
     * Maps to: {@code HandlerExecutionChain.interceptorIndex} (line 55)
     */
    private int interceptorIndex = -1;

    public HandlerExecutionChain(HandlerMethod handler) {
        this(handler, new ArrayList<>());
    }

    public HandlerExecutionChain(HandlerMethod handler, List<HandlerInterceptor> interceptors) {
        this.handler = handler;
        this.interceptorList = new ArrayList<>(interceptors);
    }

    /**
     * Add an interceptor to the end of the chain.
     */
    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptorList.add(interceptor);
    }

    /**
     * Add an interceptor at position 0 — before all other interceptors.
     *
     * <h3>ch19 Enhancement:</h3>
     * Used to insert the {@code CorsInterceptor} at the front of the chain
     * so CORS validation happens before any other interceptors.
     *
     * Maps to: The real framework inserts the CorsInterceptor at index 0
     * in {@code AbstractHandlerMapping.getCorsHandlerExecutionChain()} (line 754).
     *
     * @param interceptor the interceptor to add at position 0
     */
    public void addInterceptorFirst(HandlerInterceptor interceptor) {
        this.interceptorList.add(0, interceptor);
    }

    /**
     * Apply {@code preHandle} on each interceptor in FORWARD order.
     *
     * Maps to: {@code HandlerExecutionChain.applyPreHandle()} (line 142)
     *
     * If any interceptor returns {@code false}, immediately triggers
     * {@code afterCompletion} for all previously successful interceptors
     * and returns {@code false} to signal that dispatch should stop.
     *
     * The key line is: {@code this.interceptorIndex = i} — this records which
     * interceptors need cleanup. The interceptor that returned {@code false}
     * does NOT get its afterCompletion called (its preHandle "didn't complete").
     *
     * @return {@code true} if all interceptors approved; {@code false} if any vetoed
     */
    public boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        for (int i = 0; i < this.interceptorList.size(); i++) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            if (!interceptor.preHandle(request, response, this.handler)) {
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
        return true;
    }

    /**
     * Apply {@code postHandle} on each interceptor in REVERSE order.
     *
     * Maps to: {@code HandlerExecutionChain.applyPostHandle()} (line 157)
     *
     * Only called on the success path (no exception during handler invocation).
     * Iterates the full list in reverse — by this point all preHandle calls
     * succeeded, so all interceptors deserve their postHandle callback.
     */
    public void applyPostHandle(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            interceptor.postHandle(request, response, this.handler);
        }
    }

    /**
     * Trigger {@code afterCompletion} in REVERSE order from {@code interceptorIndex}.
     *
     * Maps to: {@code HandlerExecutionChain.triggerAfterCompletion()} (line 171)
     *
     * This is the safety-net cleanup — analogous to a finally block. Key behaviors:
     * <ul>
     *   <li>Only iterates from {@code interceptorIndex} down to 0 (not the full list)</li>
     *   <li>Each call is wrapped in try/catch — one interceptor's error must NOT prevent
     *       cleanup of earlier interceptors</li>
     *   <li>The exception parameter is the exception from handler invocation (or null)</li>
     * </ul>
     *
     * @param ex the exception from handler invocation, or {@code null} if the request
     *           completed normally
     */
    public void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response,
                                       Exception ex) {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            } catch (Throwable ex2) {
                // Log but don't propagate — must not prevent other interceptors' cleanup.
                // The real framework logs via commons-logging here.
                System.err.println("HandlerInterceptor.afterCompletion threw exception: " + ex2);
            }
        }
    }

    /**
     * Returns the handler this chain wraps.
     */
    public HandlerMethod getHandler() {
        return handler;
    }

    /**
     * Returns an unmodifiable view of the interceptor list for testing/debugging.
     */
    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptorList);
    }

    @Override
    public String toString() {
        return "HandlerExecutionChain with [" + handler + "] and " +
                interceptorList.size() + " interceptors " + interceptorList;
    }
}
```

#### File: `build.gradle` [MODIFIED]

```groovy
plugins {
    id 'java'
}

group = 'com.simplespringmvc'
version = '1.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

// Preserve parameter names in bytecode so @PathVariable (and @RequestParam)
// can fall back to the method parameter name when no explicit value is given.
// This is what Spring Boot enables by default via its Gradle/Maven plugin.
tasks.withType(JavaCompile).configureEach {
    options.compilerArgs.add('-parameters')
}

repositories {
    mavenCentral()
}

dependencies {
    // Embedded Tomcat (Servlet container + WebSocket support)
    implementation 'org.apache.tomcat.embed:tomcat-embed-core:11.0.6'
    implementation 'org.apache.tomcat.embed:tomcat-embed-websocket:11.0.6'

    // Jakarta Servlet API
    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'

    // Jackson for JSON processing
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.18.3'

    // Jakarta Bean Validation + Hibernate Validator (reference implementation)
    implementation 'org.hibernate.validator:hibernate-validator:8.0.2.Final'
    // Expression Language implementation — required by Hibernate Validator
    // for message interpolation (e.g., "${validatedValue}" in constraint messages)
    runtimeOnly 'org.glassfish.expressly:expressly:5.0.0'

    // Testing
    testImplementation 'org.junit.jupiter:junit-jupiter:5.11.4'
    testImplementation 'org.assertj:assertj-core:3.27.3'
    testImplementation 'org.mockito:mockito-core:5.14.2'
    testImplementation 'org.springframework:spring-test:6.2.4'
    testImplementation 'org.springframework:spring-web:6.2.4'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform()
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/cors/CorsConfigurationTest.java` [NEW]

```java
package com.simplespringmvc.cors;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link CorsConfiguration} — verifying configuration defaults,
 * origin/method/header checking, combining, and validation.
 */
class CorsConfigurationTest {

    // ─── applyPermitDefaultValues ────────────────────────────────

    @Test
    void shouldSetPermissiveDefaults_WhenApplyPermitDefaultValuesCalled() {
        CorsConfiguration config = new CorsConfiguration();
        config.applyPermitDefaultValues();

        assertThat(config.getAllowedOrigins()).containsExactly("*");
        assertThat(config.getAllowedMethods()).containsExactly("GET", "HEAD", "POST");
        assertThat(config.getAllowedHeaders()).containsExactly("*");
        assertThat(config.getMaxAge()).isEqualTo(1800L);
    }

    @Test
    void shouldNotOverrideExplicitValues_WhenApplyPermitDefaultValuesCalled() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://example.com"));
        config.setAllowedMethods(List.of("GET"));
        config.applyPermitDefaultValues();

        assertThat(config.getAllowedOrigins()).containsExactly("https://example.com");
        assertThat(config.getAllowedMethods()).containsExactly("GET");
    }

    // ─── checkOrigin ─────────────────────────────────────────────

    @Test
    void shouldReturnWildcard_WhenAllOriginsAllowedAndNoCredentials() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("*");

        assertThat(config.checkOrigin("https://example.com")).isEqualTo("*");
    }

    @Test
    void shouldEchoOrigin_WhenAllOriginsAllowedWithCredentials() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("*");
        config.setAllowCredentials(true);

        assertThat(config.checkOrigin("https://example.com"))
                .isEqualTo("https://example.com");
    }

    @Test
    void shouldReturnOrigin_WhenOriginMatchesExactly() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");

        assertThat(config.checkOrigin("https://example.com"))
                .isEqualTo("https://example.com");
    }

    @Test
    void shouldReturnNull_WhenOriginNotAllowed() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");

        assertThat(config.checkOrigin("https://evil.com")).isNull();
    }

    @Test
    void shouldReturnNull_WhenOriginIsNull() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");

        assertThat(config.checkOrigin(null)).isNull();
    }

    @Test
    void shouldMatchCaseInsensitive_WhenCheckingOrigin() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");

        assertThat(config.checkOrigin("HTTPS://EXAMPLE.COM"))
                .isEqualTo("HTTPS://EXAMPLE.COM");
    }

    // ─── checkHttpMethod ─────────────────────────────────────────

    @Test
    void shouldReturnMethods_WhenMethodIsAllowed() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("GET");
        config.addAllowedMethod("POST");

        assertThat(config.checkHttpMethod("GET")).isNotNull();
    }

    @Test
    void shouldReturnNull_WhenMethodNotAllowed() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("GET");

        assertThat(config.checkHttpMethod("DELETE")).isNull();
    }

    @Test
    void shouldAllowAnyMethod_WhenWildcardUsed() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");

        assertThat(config.checkHttpMethod("PUT")).containsExactly("PUT");
    }

    // ─── checkHeaders ────────────────────────────────────────────

    @Test
    void shouldReturnHeaders_WhenAllowedHeadersMatch() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedHeader("Content-Type");
        config.addAllowedHeader("Authorization");

        List<String> result = config.checkHeaders(List.of("Content-Type"));
        assertThat(result).containsExactly("Content-Type");
    }

    @Test
    void shouldReturnNull_WhenHeaderNotAllowed() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedHeader("Content-Type");

        assertThat(config.checkHeaders(List.of("X-Custom"))).isNull();
    }

    @Test
    void shouldAllowAllHeaders_WhenWildcardUsed() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedHeader("*");

        List<String> result = config.checkHeaders(List.of("X-Custom", "Authorization"));
        assertThat(result).containsExactly("X-Custom", "Authorization");
    }

    @Test
    void shouldReturnEmptyList_WhenNoHeadersRequested() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedHeader("Content-Type");

        assertThat(config.checkHeaders(List.of())).isEmpty();
    }

    // ─── combine ─────────────────────────────────────────────────

    @Test
    void shouldMergeOriginsAdditively_WhenCombining() {
        CorsConfiguration global = new CorsConfiguration();
        global.addAllowedOrigin("https://a.com");

        CorsConfiguration handler = new CorsConfiguration();
        handler.addAllowedOrigin("https://b.com");

        CorsConfiguration merged = global.combine(handler);
        assertThat(merged.getAllowedOrigins())
                .containsExactly("https://a.com", "https://b.com");
    }

    @Test
    void shouldUseOtherScalars_WhenCombining() {
        CorsConfiguration global = new CorsConfiguration();
        global.setMaxAge(1800L);
        global.setAllowCredentials(false);

        CorsConfiguration handler = new CorsConfiguration();
        handler.setMaxAge(3600L);
        handler.setAllowCredentials(true);

        CorsConfiguration merged = global.combine(handler);
        assertThat(merged.getMaxAge()).isEqualTo(3600L);
        assertThat(merged.getAllowCredentials()).isTrue();
    }

    @Test
    void shouldReturnThis_WhenOtherIsNull() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");

        assertThat(config.combine(null)).isSameAs(config);
    }

    @Test
    void shouldCollapseToWildcard_WhenEitherListHasAll() {
        CorsConfiguration global = new CorsConfiguration();
        global.addAllowedOrigin("*");

        CorsConfiguration handler = new CorsConfiguration();
        handler.addAllowedOrigin("https://specific.com");

        CorsConfiguration merged = global.combine(handler);
        assertThat(merged.getAllowedOrigins()).containsExactly("*");
    }

    // ─── validateAllowCredentials ────────────────────────────────

    @Test
    void shouldThrow_WhenCredentialsTrueAndOriginWildcard() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("*");
        config.setAllowCredentials(true);

        assertThatThrownBy(config::validateAllowCredentials)
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("allowedOrigins cannot contain '*'");
    }

    @Test
    void shouldNotThrow_WhenCredentialsTrueAndSpecificOrigin() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.setAllowCredentials(true);

        assertThatCode(config::validateAllowCredentials).doesNotThrowAnyException();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/cors/DefaultCorsProcessorTest.java` [NEW]

```java
package com.simplespringmvc.cors;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link DefaultCorsProcessor} — verifying CORS header writing,
 * request validation, and rejection behavior.
 */
class DefaultCorsProcessorTest {

    private DefaultCorsProcessor processor;
    private MockHttpServletRequest request;
    private MockHttpServletResponse response;

    @BeforeEach
    void setUp() {
        processor = new DefaultCorsProcessor();
        request = new MockHttpServletRequest();
        response = new MockHttpServletResponse();
        // Set up as a cross-origin request
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);
        request.addHeader("Origin", "https://example.com");
    }

    // ─── Allow through (no CORS) ────────────────────────────────

    @Test
    void shouldAllowThrough_WhenConfigIsNull() throws Exception {
        boolean result = processor.processRequest(null, request, response);

        assertThat(result).isTrue();
        assertThat(response.getHeader("Access-Control-Allow-Origin")).isNull();
    }

    @Test
    void shouldAllowThrough_WhenNotCorsRequest() throws Exception {
        // Same-origin request
        request = new MockHttpServletRequest();
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);
        // No Origin header = not CORS

        CorsConfiguration config = new CorsConfiguration();
        config.applyPermitDefaultValues();

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isTrue();
    }

    // ─── Add Vary headers ────────────────────────────────────────

    @Test
    void shouldAddVaryHeaders_WhenProcessingCorsRequest() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.applyPermitDefaultValues();

        processor.processRequest(config, request, response);

        String vary = response.getHeader("Vary");
        assertThat(vary).contains("Origin");
        assertThat(vary).contains("Access-Control-Request-Method");
        assertThat(vary).contains("Access-Control-Request-Headers");
    }

    // ─── Actual CORS request (not preflight) ────────────────────

    @Test
    void shouldWriteAllowOrigin_WhenOriginAllowed() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("GET");
        request.setMethod("GET");

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isTrue();
        assertThat(response.getHeader("Access-Control-Allow-Origin"))
                .isEqualTo("https://example.com");
    }

    @Test
    void shouldWriteWildcardOrigin_WhenAllOriginsAllowed() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("*");
        config.addAllowedMethod("GET");
        request.setMethod("GET");

        processor.processRequest(config, request, response);

        assertThat(response.getHeader("Access-Control-Allow-Origin")).isEqualTo("*");
    }

    @Test
    void shouldReject403_WhenOriginNotAllowed() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://trusted.com");
        config.addAllowedMethod("GET");
        request.setMethod("GET");

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isFalse();
        assertThat(response.getStatus()).isEqualTo(403);
    }

    @Test
    void shouldReject403_WhenMethodNotAllowed() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("GET");
        request.setMethod("DELETE");

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isFalse();
        assertThat(response.getStatus()).isEqualTo(403);
    }

    @Test
    void shouldWriteCredentialsHeader_WhenAllowCredentialsTrue() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("GET");
        config.setAllowCredentials(true);
        request.setMethod("GET");

        processor.processRequest(config, request, response);

        assertThat(response.getHeader("Access-Control-Allow-Credentials"))
                .isEqualTo("true");
    }

    @Test
    void shouldWriteExposedHeaders_WhenConfigured() throws Exception {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("GET");
        config.addExposedHeader("X-Custom-Header");
        request.setMethod("GET");

        processor.processRequest(config, request, response);

        assertThat(response.getHeader("Access-Control-Expose-Headers"))
                .isEqualTo("X-Custom-Header");
    }

    // ─── Preflight request ───────────────────────────────────────

    @Test
    void shouldHandlePreflight_WhenOptionsWithRequestMethod() throws Exception {
        request.setMethod("OPTIONS");
        request.addHeader("Access-Control-Request-Method", "PUT");

        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("PUT");
        config.addAllowedHeader("Content-Type");

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isTrue();
        assertThat(response.getHeader("Access-Control-Allow-Origin"))
                .isEqualTo("https://example.com");
        assertThat(response.getHeader("Access-Control-Allow-Methods"))
                .contains("PUT");
    }

    @Test
    void shouldWriteMaxAge_WhenPreflightWithMaxAge() throws Exception {
        request.setMethod("OPTIONS");
        request.addHeader("Access-Control-Request-Method", "PUT");

        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("PUT");
        config.setMaxAge(3600L);

        processor.processRequest(config, request, response);

        assertThat(response.getHeader("Access-Control-Max-Age")).isEqualTo("3600");
    }

    @Test
    void shouldWriteAllowHeaders_WhenPreflightWithHeaders() throws Exception {
        request.setMethod("OPTIONS");
        request.addHeader("Access-Control-Request-Method", "POST");
        request.addHeader("Access-Control-Request-Headers", "Content-Type, Authorization");

        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("POST");
        config.addAllowedHeader("*");

        processor.processRequest(config, request, response);

        assertThat(response.getHeader("Access-Control-Allow-Headers"))
                .contains("Content-Type")
                .contains("Authorization");
    }

    @Test
    void shouldReject403_WhenPreflightHeaderNotAllowed() throws Exception {
        request.setMethod("OPTIONS");
        request.addHeader("Access-Control-Request-Method", "POST");
        request.addHeader("Access-Control-Request-Headers", "X-Evil-Header");

        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");
        config.addAllowedMethod("POST");
        config.addAllowedHeader("Content-Type"); // only Content-Type allowed

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isFalse();
        assertThat(response.getStatus()).isEqualTo(403);
    }

    // ─── Skip double-processing ──────────────────────────────────

    @Test
    void shouldSkip_WhenAllowOriginAlreadySet() throws Exception {
        response.setHeader("Access-Control-Allow-Origin", "https://already-set.com");

        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOrigin("https://example.com");

        boolean result = processor.processRequest(config, request, response);

        assertThat(result).isTrue();
        // Should NOT overwrite the existing header
        assertThat(response.getHeader("Access-Control-Allow-Origin"))
                .isEqualTo("https://already-set.com");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/cors/CorsUtilsTest.java` [NEW]

```java
package com.simplespringmvc.cors;

import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link CorsUtils} — verifying CORS request and
 * preflight request detection.
 */
class CorsUtilsTest {

    // ─── isCorsRequest ───────────────────────────────────────────

    @Test
    void shouldReturnTrue_WhenOriginDiffersFromServer() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);
        request.addHeader("Origin", "https://example.com");

        assertThat(CorsUtils.isCorsRequest(request)).isTrue();
    }

    @Test
    void shouldReturnFalse_WhenNoOriginHeader() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);

        assertThat(CorsUtils.isCorsRequest(request)).isFalse();
    }

    @Test
    void shouldReturnFalse_WhenOriginMatchesServer() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);
        request.addHeader("Origin", "http://localhost:8080");

        assertThat(CorsUtils.isCorsRequest(request)).isFalse();
    }

    @Test
    void shouldReturnTrue_WhenDifferentPort() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);
        request.addHeader("Origin", "http://localhost:3000");

        assertThat(CorsUtils.isCorsRequest(request)).isTrue();
    }

    @Test
    void shouldReturnTrue_WhenDifferentScheme() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(80);
        request.addHeader("Origin", "https://localhost");

        assertThat(CorsUtils.isCorsRequest(request)).isTrue();
    }

    // ─── isPreFlightRequest ──────────────────────────────────────

    @Test
    void shouldReturnTrue_WhenOptionsWithOriginAndRequestMethod() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setMethod("OPTIONS");
        request.addHeader("Origin", "https://example.com");
        request.addHeader("Access-Control-Request-Method", "PUT");

        assertThat(CorsUtils.isPreFlightRequest(request)).isTrue();
    }

    @Test
    void shouldReturnFalse_WhenNotOptions() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setMethod("GET");
        request.addHeader("Origin", "https://example.com");
        request.addHeader("Access-Control-Request-Method", "PUT");

        assertThat(CorsUtils.isPreFlightRequest(request)).isFalse();
    }

    @Test
    void shouldReturnFalse_WhenOptionsMissingOrigin() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setMethod("OPTIONS");
        request.addHeader("Access-Control-Request-Method", "PUT");

        assertThat(CorsUtils.isPreFlightRequest(request)).isFalse();
    }

    @Test
    void shouldReturnFalse_WhenOptionsMissingRequestMethod() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setMethod("OPTIONS");
        request.addHeader("Origin", "https://example.com");

        assertThat(CorsUtils.isPreFlightRequest(request)).isFalse();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/cors/CorsRegistryTest.java` [NEW]

```java
package com.simplespringmvc.cors;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link CorsRegistry} and {@link CorsRegistration} —
 * verifying the fluent API for global CORS configuration.
 */
class CorsRegistryTest {

    @Test
    void shouldCreateRegistration_WhenMappingAdded() {
        CorsRegistry registry = new CorsRegistry();
        registry.addMapping("/api/**");

        Map<String, CorsConfiguration> configs = registry.getCorsConfigurations();

        assertThat(configs).containsKey("/api/**");
        assertThat(configs.get("/api/**").getAllowedOrigins()).containsExactly("*");
    }

    @Test
    void shouldSupportFluentConfiguration_WhenChained() {
        CorsRegistry registry = new CorsRegistry();
        registry.addMapping("/api/**")
                .allowedOrigins("https://example.com")
                .allowedMethods("GET", "POST")
                .allowedHeaders("Content-Type", "Authorization")
                .exposedHeaders("X-Custom")
                .allowCredentials(true)
                .maxAge(3600);

        CorsConfiguration config = registry.getCorsConfigurations().get("/api/**");

        assertThat(config.getAllowedOrigins()).containsExactly("https://example.com");
        assertThat(config.getAllowedMethods()).containsExactly("GET", "POST");
        assertThat(config.getAllowedHeaders()).containsExactly("Content-Type", "Authorization");
        assertThat(config.getExposedHeaders()).containsExactly("X-Custom");
        assertThat(config.getAllowCredentials()).isTrue();
        assertThat(config.getMaxAge()).isEqualTo(3600L);
    }

    @Test
    void shouldSupportMultipleMappings_WhenMultipleAdded() {
        CorsRegistry registry = new CorsRegistry();
        registry.addMapping("/api/**")
                .allowedOrigins("https://api.example.com");
        registry.addMapping("/admin/**")
                .allowedOrigins("https://admin.example.com");

        Map<String, CorsConfiguration> configs = registry.getCorsConfigurations();

        assertThat(configs).hasSize(2);
        assertThat(configs.get("/api/**").getAllowedOrigins())
                .containsExactly("https://api.example.com");
        assertThat(configs.get("/admin/**").getAllowedOrigins())
                .containsExactly("https://admin.example.com");
    }

    @Test
    void shouldApplyPermitDefaults_WhenNoExplicitConfig() {
        CorsRegistry registry = new CorsRegistry();
        registry.addMapping("/**");

        CorsConfiguration config = registry.getCorsConfigurations().get("/**");

        // Defaults from applyPermitDefaultValues()
        assertThat(config.getAllowedOrigins()).containsExactly("*");
        assertThat(config.getAllowedMethods()).containsExactly("GET", "HEAD", "POST");
        assertThat(config.getAllowedHeaders()).containsExactly("*");
        assertThat(config.getMaxAge()).isEqualTo(1800L);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/CorsIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.CrossOrigin;
import com.simplespringmvc.annotation.GetMapping;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.cors.CorsRegistry;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import jakarta.servlet.ServletException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for CORS Configuration (Feature 19) — verifying
 * end-to-end CORS processing through the full dispatch pipeline.
 */
class CorsIntegrationTest {

    private SimpleDispatcherServlet servlet;
    private SimpleBeanContainer container;

    @BeforeEach
    void setUp() throws ServletException {
        container = new SimpleBeanContainer();
    }

    private void initServlet() throws ServletException {
        servlet = new SimpleDispatcherServlet(container);
        servlet.init();
    }

    // ─── Test controllers ────────────────────────────────────────

    @Controller
    @CrossOrigin(origins = "https://trusted.com")
    static class ClassLevelCorsController {
        @GetMapping("/class-cors/hello")
        @ResponseBody
        public String hello() {
            return "Hello from class-cors";
        }

        @GetMapping("/class-cors/other")
        @ResponseBody
        public String other() {
            return "Other from class-cors";
        }
    }

    @Controller
    static class MethodLevelCorsController {
        @GetMapping("/method-cors/hello")
        @CrossOrigin(origins = "https://method-trusted.com", maxAge = 600)
        @ResponseBody
        public String hello() {
            return "Hello from method-cors";
        }

        @GetMapping("/method-cors/no-cors")
        @ResponseBody
        public String noCors() {
            return "No CORS config";
        }
    }

    @Controller
    @CrossOrigin(origins = "https://class.com")
    static class MergedCorsController {
        @GetMapping("/merged/hello")
        @CrossOrigin(origins = "https://method.com")
        @ResponseBody
        public String hello() {
            return "Hello from merged";
        }
    }

    @Controller
    static class NoCorsController {
        @GetMapping("/no-cors/hello")
        @ResponseBody
        public String hello() {
            return "No CORS at all";
        }
    }

    @Controller
    static class CorsWithPutController {
        @RequestMapping(path = "/cors-put", method = "PUT")
        @CrossOrigin(origins = "https://trusted.com", methods = {"PUT", "DELETE"})
        @ResponseBody
        public String update() {
            return "Updated";
        }
    }

    // ─── Helpers ─────────────────────────────────────────────────

    private MockHttpServletRequest createCorsRequest(String method, String path, String origin) {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setMethod(method);
        request.setRequestURI(path);
        request.setScheme("http");
        request.setServerName("localhost");
        request.setServerPort(8080);
        request.addHeader("Origin", origin);
        return request;
    }

    private MockHttpServletRequest createPreflightRequest(String path, String origin,
                                                          String requestMethod) {
        MockHttpServletRequest request = createCorsRequest("OPTIONS", path, origin);
        request.addHeader("Access-Control-Request-Method", requestMethod);
        return request;
    }

    // ─── Class-level @CrossOrigin ────────────────────────────────

    @Nested
    class ClassLevelCors {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("classLevelCorsController",
                    new ClassLevelCorsController());
            initServlet();
        }

        @Test
        void shouldAllowRequest_WhenOriginIsTrusted() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/class-cors/hello", "https://trusted.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://trusted.com");
        }

        @Test
        void shouldRejectRequest_WhenOriginNotTrusted() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/class-cors/hello", "https://evil.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(403);
        }

        @Test
        void shouldApplyToAllMethods_WhenClassLevelAnnotation() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/class-cors/other", "https://trusted.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://trusted.com");
        }
    }

    // ─── Method-level @CrossOrigin ───────────────────────────────

    @Nested
    class MethodLevelCors {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("methodLevelCorsController",
                    new MethodLevelCorsController());
            initServlet();
        }

        @Test
        void shouldAllowRequest_WhenMethodAnnotationPresent() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/method-cors/hello", "https://method-trusted.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://method-trusted.com");
        }

        @Test
        void shouldNotSetCorsHeaders_WhenNoAnnotation() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/method-cors/no-cors", "https://any-origin.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            // No CORS config → no CORS headers but request goes through
            assertThat(response.getHeader("Access-Control-Allow-Origin")).isNull();
        }
    }

    // ─── Merged class + method @CrossOrigin ──────────────────────

    @Nested
    class MergedCors {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("mergedCorsController",
                    new MergedCorsController());
            initServlet();
        }

        @Test
        void shouldAllowClassOrigin_WhenMerged() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/merged/hello", "https://class.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://class.com");
        }

        @Test
        void shouldAllowMethodOrigin_WhenMerged() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/merged/hello", "https://method.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://method.com");
        }
    }

    // ─── Preflight handling ──────────────────────────────────────

    @Nested
    class Preflight {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("corsWithPutController",
                    new CorsWithPutController());
            initServlet();
        }

        @Test
        void shouldHandlePreflight_WhenOptionsWithRequestMethod() throws Exception {
            MockHttpServletRequest request = createPreflightRequest(
                    "/cors-put", "https://trusted.com", "PUT");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://trusted.com");
            assertThat(response.getHeader("Access-Control-Allow-Methods"))
                    .contains("PUT");
        }

        @Test
        void shouldRejectPreflight_WhenOriginNotAllowed() throws Exception {
            MockHttpServletRequest request = createPreflightRequest(
                    "/cors-put", "https://evil.com", "PUT");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(403);
        }
    }

    // ─── Global CORS configuration via CorsRegistry ──────────────

    @Nested
    class GlobalCors {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("noCorsController",
                    new NoCorsController());
            initServlet();

            CorsRegistry registry = new CorsRegistry();
            registry.addMapping("/**")
                    .allowedOrigins("https://global.com")
                    .allowedMethods("GET", "POST");
            servlet.setCorsRegistry(registry);
        }

        @Test
        void shouldApplyGlobalCors_WhenNoAnnotation() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/no-cors/hello", "https://global.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://global.com");
        }

        @Test
        void shouldReject_WhenGlobalCorsOriginNotMatched() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/no-cors/hello", "https://not-global.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(403);
        }
    }

    // ─── Global + handler-level CORS merging ─────────────────────

    @Nested
    class MergedGlobalAndHandler {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("classLevelCorsController",
                    new ClassLevelCorsController());
            initServlet();

            CorsRegistry registry = new CorsRegistry();
            registry.addMapping("/**")
                    .allowedOrigins("https://global.com")
                    .allowedMethods("GET", "POST");
            servlet.setCorsRegistry(registry);
        }

        @Test
        void shouldAllowGlobalOrigin_WhenGlobalAndHandlerMerged() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/class-cors/hello", "https://global.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://global.com");
        }

        @Test
        void shouldAllowHandlerOrigin_WhenGlobalAndHandlerMerged() throws Exception {
            MockHttpServletRequest request = createCorsRequest(
                    "GET", "/class-cors/hello", "https://trusted.com");
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin"))
                    .isEqualTo("https://trusted.com");
        }
    }

    // ─── Same-origin (not CORS) ──────────────────────────────────

    @Nested
    class SameOrigin {

        @BeforeEach
        void setup() throws ServletException {
            container.registerBean("classLevelCorsController",
                    new ClassLevelCorsController());
            initServlet();
        }

        @Test
        void shouldNotAddCorsHeaders_WhenSameOriginRequest() throws Exception {
            MockHttpServletRequest request = new MockHttpServletRequest();
            request.setMethod("GET");
            request.setRequestURI("/class-cors/hello");
            // No Origin header = same origin
            MockHttpServletResponse response = new MockHttpServletResponse();

            servlet.service(request, response);

            assertThat(response.getStatus()).isEqualTo(200);
            assertThat(response.getHeader("Access-Control-Allow-Origin")).isNull();
        }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`CorsConfiguration`** | Central data structure holding allowed origins, methods, headers, exposed headers, credentials, and max age -- provides `checkOrigin()`, `checkHttpMethod()`, `checkHeaders()` for validation and `combine()` for merging |
| **`CorsUtils`** | Static utility that classifies requests: `isCorsRequest()` detects cross-origin by comparing Origin header to server address; `isPreFlightRequest()` detects OPTIONS + Origin + Access-Control-Request-Method |
| **`CorsProcessor` / `DefaultCorsProcessor`** | Strategy interface and its implementation that enforces the W3C CORS spec: validate origin/method/headers, write CORS response headers, or reject with 403 |
| **`@CrossOrigin`** | Annotation for type-level and method-level CORS configuration -- origins, methods, headers are read during handler detection and stored per handler method |
| **`CorsInterceptor`** | Interceptor inserted at position 0 in the execution chain -- delegates to `CorsProcessor` in `preHandle()`, stopping the chain with 403 if rejected |
| **`CorsRegistry` / `CorsRegistration`** | Fluent API for global CORS configuration by URL pattern -- configurations are merged with handler-level configs via `combine()` |
| **`addInterceptorFirst()`** | New method on `HandlerExecutionChain` that inserts the `CorsInterceptor` before all other interceptors, ensuring CORS runs first |
| **No-op preflight handler** | For preflight OPTIONS requests with no explicit handler, a synthetic handler is created; the `CorsInterceptor` writes CORS headers and the handler does nothing |
| **Global + handler merging** | Global config (from `CorsRegistry`) is the base, handler config (from `@CrossOrigin`) is overlaid -- lists merge additively, scalars use "other wins" |

**Next: Chapter 20 -- Locale Resolution** -- Support internationalization by resolving the user's locale from request headers, cookies, or session, and making it available to view rendering and message interpolation
