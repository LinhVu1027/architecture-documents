# Chapter 7: Path Variables & Pattern Matching

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Handler mapping uses exact string equality — `/users` matches `/users` | Cannot handle RESTful URLs like `/users/42` or `/users/1/posts/99` — every distinct ID would need its own handler method | Build a pattern-matching system that parses URI templates like `/users/{id}`, matches incoming request paths, extracts variable values, and passes them as controller method arguments via `@PathVariable` |

---

## 7.1 The Integration Point

The integration point for path variables spans two subsystems: **handler mapping** (where URLs are matched) and **argument resolution** (where method parameters are populated). These two subsystems are decoupled — the handler mapping runs first during `getHandler()`, the argument resolver runs later during `handle()`. They need a way to communicate the extracted variable values.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Pass the full `HttpServletRequest` to handler mapping instead of just `requestPath` and `requestMethod`

```java
// BEFORE (ch03-ch06):
private HandlerMethod getHandler(HttpServletRequest request) {
    if (handlerMapping == null) {
        return null;
    }
    return handlerMapping.lookupHandler(request.getRequestURI(), request.getMethod());
}

// AFTER (ch07):
private HandlerMethod getHandler(HttpServletRequest request) {
    if (handlerMapping == null) {
        return null;
    }
    return handlerMapping.lookupHandler(request);  // pass full request
}
```

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Register `PathVariableArgumentResolver` before `RequestParamArgumentResolver`

```java
// BEFORE (ch05-ch06):
argumentResolvers.addResolver(new RequestParamArgumentResolver());

// AFTER (ch07):
argumentResolvers.addResolver(new PathVariableArgumentResolver());
argumentResolvers.addResolver(new RequestParamArgumentResolver());
```

Two key decisions here:

1. **Request attributes as the bridge:** Rather than returning path variables from `lookupHandler()` (which would change the method signature), we store them as a request attribute. This is exactly what the real framework does — `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE`. It keeps the handler mapping and argument resolver decoupled.
2. **Passing the full request object:** The handler mapping needs the request object to set the attribute. This is a natural evolution — the real `DispatcherServlet.getHandler()` always passes the full request.

This connects **pattern matching** (in handler mapping) to **argument resolution** (in the adapter). To make it work, we need to build:
- `PathPattern` — parses URI templates and matches/extracts variables
- `@PathVariable` — annotation marking parameters to bind
- `PathVariableArgumentResolver` — reads extracted variables from the request attribute
- Enhanced `RouteKey` — delegates to `PathPattern` instead of exact string equality
- Enhanced `SimpleHandlerMapping` — stores extracted variables on the request

## 7.2 PathPattern — The URL Pattern Parser and Matcher

This is the core engine. It parses a URI template like `/users/{id}/posts/{postId}` into segments (literal or variable capture), then matches incoming request paths against those segments.

**New file:** `src/main/java/com/simplespringmvc/mapping/PathPattern.java`

```java
package com.simplespringmvc.mapping;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class PathPattern {

    private final String patternString;
    private final List<Segment> segments;
    private final boolean hasVariables;

    public PathPattern(String pattern) {
        this.patternString = pattern;
        this.segments = parse(pattern);
        this.hasVariables = segments.stream().anyMatch(Segment::isVariable);
    }

    public boolean matches(String requestPath) {
        String[] requestSegments = splitPath(requestPath);
        return doMatch(requestSegments) != null;
    }

    public Map<String, String> matchAndExtract(String requestPath) {
        String[] requestSegments = splitPath(requestPath);
        return doMatch(requestSegments);
    }

    public boolean hasVariables() {
        return hasVariables;
    }

    private Map<String, String> doMatch(String[] requestSegments) {
        if (segments.size() != requestSegments.length) {
            return null;
        }
        Map<String, String> variables = new LinkedHashMap<>();
        for (int i = 0; i < segments.size(); i++) {
            Segment segment = segments.get(i);
            String requestValue = requestSegments[i];
            if (segment.isVariable()) {
                if (requestValue.isEmpty()) return null;
                variables.put(segment.variableName(), requestValue);
            } else {
                if (!segment.text().equals(requestValue)) return null;
            }
        }
        return Collections.unmodifiableMap(variables);
    }

    private static List<Segment> parse(String pattern) {
        String[] parts = splitPath(pattern);
        List<Segment> segments = new ArrayList<>(parts.length);
        for (String part : parts) {
            if (part.startsWith("{") && part.endsWith("}")) {
                String varName = part.substring(1, part.length() - 1);
                if (varName.isEmpty()) {
                    throw new IllegalArgumentException("Empty variable name in pattern: " + pattern);
                }
                segments.add(new Segment(null, varName));
            } else {
                segments.add(new Segment(part, null));
            }
        }
        return List.copyOf(segments);
    }

    private static String[] splitPath(String path) {
        if (path == null || path.isEmpty() || path.equals("/")) return new String[0];
        String trimmed = path;
        if (trimmed.startsWith("/")) trimmed = trimmed.substring(1);
        if (trimmed.endsWith("/")) trimmed = trimmed.substring(0, trimmed.length() - 1);
        if (trimmed.isEmpty()) return new String[0];
        return trimmed.split("/");
    }

    record Segment(String text, String variableName) {
        boolean isVariable() { return variableName != null; }
    }
}
```

The real framework's `PathPattern` uses a linked list of `PathElement` subclasses (`LiteralPathElement`, `CaptureVariablePathElement`, `WildcardPathElement`, etc.) that are built by the `InternalPathPatternParser` character-by-character. Our simplified version splits on `/` and classifies each segment — same concept, much less code.

## 7.3 @PathVariable Annotation

**New file:** `src/main/java/com/simplespringmvc/annotation/PathVariable.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface PathVariable {
    String value() default "";
}
```

Same design as `@RequestParam` — an explicit `value()` for the variable name with fallback to the Java parameter name (requires `-parameters` compiler flag).

## 7.4 RouteKey Enhancement — Pattern-Based Matching

**Modifying:** `src/main/java/com/simplespringmvc/mapping/RouteKey.java`
**Change:** Replace exact `String.equals()` with `PathPattern` matching; add `matchAndExtract()` method

```java
// NEW field:
private final PathPattern pathPattern;

// Constructor now creates a PathPattern:
public RouteKey(String path, String httpMethod) {
    this.path = normalizePath(path);
    this.httpMethod = httpMethod != null ? httpMethod.toUpperCase() : "";
    this.pathPattern = new PathPattern(this.path);
}

// matches() now delegates to PathPattern:
public boolean matches(String requestPath, String requestMethod) {
    String normalizedRequestPath = normalizePath(requestPath);
    if (!pathPattern.matches(normalizedRequestPath)) {
        return false;
    }
    if (this.httpMethod.isEmpty()) return true;
    return this.httpMethod.equalsIgnoreCase(requestMethod);
}

// NEW method for extracting variables:
public Map<String, String> matchAndExtract(String requestPath, String requestMethod) {
    String normalizedRequestPath = normalizePath(requestPath);
    if (!this.httpMethod.isEmpty() && !this.httpMethod.equalsIgnoreCase(requestMethod)) {
        return null;
    }
    return pathPattern.matchAndExtract(normalizedRequestPath);
}
```

The `matches()` method still works for existing code (returns `boolean`). The new `matchAndExtract()` returns the captured variable map, or `null` if no match.

## 7.5 SimpleHandlerMapping Enhancement — Store Path Variables on Request

**Modifying:** `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java`
**Change:** Add `PATH_VARIABLES_ATTRIBUTE` constant and new `lookupHandler(HttpServletRequest)` overload

```java
public static final String PATH_VARIABLES_ATTRIBUTE =
        "com.simplespringmvc.mapping.SimpleHandlerMapping.pathVariables";

// NEW overload that stores path variables on the request:
public HandlerMethod lookupHandler(HttpServletRequest request) {
    String requestPath = request.getRequestURI();
    String requestMethod = request.getMethod();

    for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
        RouteKey key = entry.getKey();
        Map<String, String> pathVariables = key.matchAndExtract(requestPath, requestMethod);
        if (pathVariables != null) {
            request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);
            return entry.getValue();
        }
    }
    return null;
}
```

The original `lookupHandler(String, String)` overload is preserved for backward compatibility with existing tests.

## 7.6 PathVariableArgumentResolver — Reading Extracted Variables

**New file:** `src/main/java/com/simplespringmvc/adapter/PathVariableArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.http.HttpServletRequest;
import java.util.Map;

public class PathVariableArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(PathVariable.class);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String variableName = getVariableName(parameter);

        Map<String, String> pathVariables = (Map<String, String>) request.getAttribute(
                SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE);

        if (pathVariables == null) {
            throw new IllegalStateException(
                    "No path variables found in request.");
        }

        String value = pathVariables.get(variableName);
        if (value == null) {
            throw new IllegalStateException(
                    "Missing path variable '" + variableName + "'");
        }
        return value;
    }

    private String getVariableName(MethodParameter parameter) {
        PathVariable annotation = parameter.getParameterAnnotation(PathVariable.class);
        return (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
    }
}
```

## 7.7 Try It Yourself

<details>
<summary>Challenge 1: Implement a PathPattern that handles /users/{id}</summary>

Given the `Segment` record, implement `doMatch()` that walks pattern segments and request segments in lockstep:

```java
private Map<String, String> doMatch(String[] requestSegments) {
    if (segments.size() != requestSegments.length) {
        return null;
    }
    Map<String, String> variables = new LinkedHashMap<>();
    for (int i = 0; i < segments.size(); i++) {
        Segment segment = segments.get(i);
        String requestValue = requestSegments[i];
        if (segment.isVariable()) {
            if (requestValue.isEmpty()) return null;
            variables.put(segment.variableName(), requestValue);
        } else {
            if (!segment.text().equals(requestValue)) return null;
        }
    }
    return Collections.unmodifiableMap(variables);
}
```

</details>

<details>
<summary>Challenge 2: Bridge handler mapping and argument resolution using request attributes</summary>

The key insight: handler mapping extracts variables, argument resolvers consume them. How do you pass data between these decoupled subsystems?

Answer: Use `HttpServletRequest.setAttribute()` — the same approach the real Spring Framework uses:

```java
// In SimpleHandlerMapping.lookupHandler(HttpServletRequest):
request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);

// In PathVariableArgumentResolver.resolveArgument():
Map<String, String> pathVariables = (Map<String, String>) request.getAttribute(
        SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE);
```

</details>

## 7.8 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/mapping/PathPatternTest.java`

```java
class PathPatternTest {

    @Test
    void shouldMatchExactLiteralPath() {
        PathPattern pattern = new PathPattern("/users");
        assertThat(pattern.matches("/users")).isTrue();
    }

    @Test
    void shouldExtractSingleVariable() {
        PathPattern pattern = new PathPattern("/users/{id}");
        Map<String, String> variables = pattern.matchAndExtract("/users/42");
        assertThat(variables).containsEntry("id", "42");
    }

    @Test
    void shouldExtractMultipleVariables() {
        PathPattern pattern = new PathPattern("/users/{userId}/posts/{postId}");
        Map<String, String> variables = pattern.matchAndExtract("/users/1/posts/99");
        assertThat(variables)
                .containsEntry("userId", "1")
                .containsEntry("postId", "99");
    }

    @Test
    void shouldReturnNull_WhenNoMatch() {
        PathPattern pattern = new PathPattern("/users/{id}");
        assertThat(pattern.matchAndExtract("/orders/42")).isNull();
    }
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/PathVariableArgumentResolverTest.java`

```java
class PathVariableArgumentResolverTest {

    @Test
    void shouldResolveByExplicitName() throws Exception {
        when(request.getAttribute(SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE))
                .thenReturn(Map.of("id", "42"));
        MethodParameter param = parameterFor("handlerWithPathVariable");
        assertThat(resolver.resolveArgument(param, request)).isEqualTo("42");
    }

    @Test
    void shouldResolveByImplicitName() throws Exception {
        when(request.getAttribute(SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE))
                .thenReturn(Map.of("userId", "7"));
        MethodParameter param = parameterFor("handlerWithImplicitName");
        assertThat(resolver.resolveArgument(param, request)).isEqualTo("7");
    }
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/PathVariableIntegrationTest.java`

```java
class PathVariableIntegrationTest {

    @Test
    void shouldResolveSinglePathVariable() throws Exception {
        // GET /users/42 → "User 42"
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/users/42")).GET().build(),
                HttpResponse.BodyHandlers.ofString());
        assertThat(response.body()).isEqualTo("User 42");
    }

    @Test
    void shouldResolveMixedAnnotations() throws Exception {
        // GET /search/electronics?q=laptop → "Searching electronics for 'laptop'"
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/search/electronics?q=laptop")).GET().build(),
                HttpResponse.BodyHandlers.ofString());
        assertThat(response.body()).isEqualTo("Searching electronics for 'laptop'");
    }
}
```

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 7.9 Why This Works

> ★ **Insight** -------------------------------------------
> **Request attributes as a communication channel** — The handler mapping and argument resolver are independent components that run at different phases of request processing. Rather than coupling them through return values or shared state objects, the real Spring Framework uses the `HttpServletRequest` itself as a per-request data bus. This is a classic Servlet-era pattern: `request.setAttribute()` acts like a scoped Map that lives for exactly one request. The key `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE` is a well-known constant that both sides agree on — a simple protocol between decoupled components.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Segment-based matching vs. character-based matching** — The real `PathPattern` uses a linked list of `PathElement` nodes built by parsing the pattern character by character. This enables features like regex constraints (`{id:[0-9]+}`), wildcards (`*`), catch-all (`{*rest}`), and specificity ranking. Our segment-based approach (split on `/`, compare per-segment) handles the 90% case with 10% of the complexity. The trade-off: we can't support regex constraints or wildcards, but those are rarely needed in typical REST APIs.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Match-and-extract in one pass** — Rather than first calling `matches()` and then separately extracting variables, our `matchAndExtract()` does both in a single pass. The real framework uses this same optimization: `PathPattern.matchAndExtract()` creates a `MatchingContext` with `extractVariables=true`, so variable values are captured during the same walk that determines whether the pattern matches. If you need only a boolean match (e.g., for pre-filtering), `matches()` avoids the Map allocation overhead.
> -----------------------------------------------------------

## 7.10 What We Enhanced

| Aspect | Before (ch03-ch06) | Current (ch07) | Real Framework |
|--------|---------------------|----------------|----------------|
| **Path matching** | Exact string equality — `/users` matches only `/users` | Pattern matching with `{variable}` segments — `/users/{id}` matches `/users/42` | `PathPattern` with linked PathElement chain, regex constraints, wildcards, catch-all, specificity ranking (`PathPattern.java:198`) |
| **Variable extraction** | Not supported | `matchAndExtract()` returns `Map<String, String>` via segment-by-segment walk | `PathPattern.matchAndExtract()` returns `PathMatchInfo` with uriVariables + matrixVariables (`PathPattern.java:220`) |
| **Handler mapping → resolver bridge** | Not needed — no variables to pass | Request attribute `PATH_VARIABLES_ATTRIBUTE` set during lookup | `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE` set in `AbstractHandlerMethodMapping.handleMatch()` |
| **Argument resolvers** | `@RequestParam` only | `@PathVariable` + `@RequestParam` — both annotation-driven, same resolver pattern | ~30 resolvers including `PathVariableMethodArgumentResolver` extending `AbstractNamedValueMethodArgumentResolver` with type conversion |
| **DispatcherServlet → mapping** | Passes `(String path, String method)` | Passes full `HttpServletRequest` | Always passes `HttpServletRequest` — `DispatcherServlet.getHandler()` (line 1103) |

## 7.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `PathPattern` | `PathPattern` | `PathPattern.java:198` | Real version uses linked list of `PathElement` subclasses with recursive matching, regex constraints, wildcards, catch-all, matrix variables, and specificity comparator |
| `PathPattern.Segment` | `CaptureVariablePathElement`, `LiteralPathElement` | `CaptureVariablePathElement.java:59` | Real version supports regex constraints via `{var:pattern}`, compiled Java regex per variable |
| `PathPattern.parse()` | `InternalPathPatternParser.parse()` | `InternalPathPatternParser.java:108` | Real version is a character-by-character state machine building a doubly-linked list |
| `PathPattern.matchAndExtract()` | `PathPattern.matchAndExtract()` | `PathPattern.java:220` | Real version returns `PathMatchInfo` with both `uriVariables` and `matrixVariables` maps |
| `@PathVariable` | `@PathVariable` | `PathVariable.java:49` | Real version has `required` attribute and `name`/`value` alias via `@AliasFor` |
| `PathVariableArgumentResolver` | `PathVariableMethodArgumentResolver` | `PathVariableMethodArgumentResolver.java:73` | Real version extends `AbstractNamedValueMethodArgumentResolver` with type conversion via `WebDataBinder`, `required` flag, `View.PATH_VARIABLES` storage |
| `SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE` | `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE` | `HandlerMapping.java:120` | Same concept — a well-known request attribute key |
| `RouteKey.matchAndExtract()` | `RequestMappingInfo.getMatchingCondition()` | `RequestMappingInfo.java:259` | Real version composes 8 condition matchers (patterns, methods, params, headers, consumes, produces, custom) |

## 7.12 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/PathVariable.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation indicating that a method parameter should be bound to a URI template
 * variable extracted from the request path.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.PathVariable}
 *
 * Example:
 * <pre>
 * {@literal @}RequestMapping(path = "/users/{id}", method = "GET")
 * public String getUser({@literal @}PathVariable("id") String id) {
 *     return "User " + id;
 * }
 * </pre>
 *
 * When no value is specified, the parameter name from the method signature is
 * used (requires {@code -parameters} compiler flag).
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No {@code required} attribute — all path variables are required</li>
 *   <li>No support for Map parameters (all path vars as Map)</li>
 *   <li>No {@code name} alias attribute</li>
 * </ul>
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface PathVariable {

    /**
     * The name of the URI template variable to bind to.
     * If empty, the method parameter name is used as a fallback
     * (requires the {@code -parameters} compiler flag).
     */
    String value() default "";
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/PathPattern.java` [NEW]

```java
package com.simplespringmvc.mapping;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * A compiled URL pattern that can match request paths and extract variable values.
 *
 * Parses URI templates like {@code /users/{id}/posts/{postId}} into a chain of
 * segments (literal or variable capture), then matches incoming paths against
 * that chain, extracting captured variable values into a Map.
 *
 * Maps to: {@code org.springframework.web.util.pattern.PathPattern}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. PathPatternParser creates InternalPathPatternParser (per-call, not thread-safe)
 *   2. InternalPathPatternParser walks the pattern char-by-char, building a linked
 *      list of PathElement subclasses (LiteralPathElement, CaptureVariablePathElement,
 *      WildcardPathElement, RegexPathElement, etc.)
 *   3. PathPattern.matches() delegates to the head PathElement which recurses through the chain
 *   4. PathPattern.matchAndExtract() does the same with variable capture enabled
 *   5. PathMatchInfo holds extracted uriVariables + matrixVariables
 * </pre>
 *
 * We simplify to:
 * <pre>
 *   1. Constructor parses the pattern string by splitting on "/"
 *   2. Each segment is either a literal string or a {variableName} capture
 *   3. matches() splits the request path and compares segment-by-segment
 *   4. matchAndExtract() does the same, collecting variable values
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No linked-list PathElement chain — just a List of segments</li>
 *   <li>No regex constraints ({id:[0-9]+}) — variables match any non-empty segment</li>
 *   <li>No wildcard (*) or catch-all (**) patterns</li>
 *   <li>No matrix variables</li>
 *   <li>No specificity comparator — patterns are matched in registration order</li>
 * </ul>
 */
public class PathPattern {

    private final String patternString;
    private final List<Segment> segments;
    private final boolean hasVariables;

    /**
     * Parse a URI template into a PathPattern.
     *
     * Maps to: {@code PathPatternParser.parse()} which delegates to
     * {@code InternalPathPatternParser.parse()} (line 108)
     *
     * @param pattern a URI template like "/users/{id}" or "/users/{id}/posts/{postId}"
     */
    public PathPattern(String pattern) {
        this.patternString = pattern;
        this.segments = parse(pattern);
        this.hasVariables = segments.stream().anyMatch(Segment::isVariable);
    }

    /**
     * Test if this pattern matches the given request path.
     *
     * Maps to: {@code PathPattern.matches(PathContainer)} (line 198)
     * Real version creates a MatchingContext with extractVariables=false.
     */
    public boolean matches(String requestPath) {
        String[] requestSegments = splitPath(requestPath);
        return doMatch(requestSegments) != null;
    }

    /**
     * Match the given request path and extract URI template variable values.
     *
     * Maps to: {@code PathPattern.matchAndExtract(PathContainer)} (line 220)
     * Real version creates a MatchingContext with extractVariables=true,
     * returns a PathMatchInfo containing uriVariables and matrixVariables.
     *
     * @return an unmodifiable map of variable names to values, or null if no match
     */
    public Map<String, String> matchAndExtract(String requestPath) {
        String[] requestSegments = splitPath(requestPath);
        return doMatch(requestSegments);
    }

    /**
     * Whether this pattern contains any {variable} captures.
     * Patterns without variables are "exact" — they can use fast HashMap lookup.
     */
    public boolean hasVariables() {
        return hasVariables;
    }

    public String getPatternString() {
        return patternString;
    }

    /**
     * Core matching algorithm: walk both the pattern segments and request segments
     * in lockstep. Literal segments must match exactly; variable segments match
     * any non-empty value and capture it.
     *
     * Maps to the recursive PathElement.matches() chain in the real framework.
     * We use a simple iterative approach instead.
     *
     * @return extracted variables map, or null if no match
     */
    private Map<String, String> doMatch(String[] requestSegments) {
        if (segments.size() != requestSegments.length) {
            return null;
        }

        Map<String, String> variables = new LinkedHashMap<>();

        for (int i = 0; i < segments.size(); i++) {
            Segment segment = segments.get(i);
            String requestValue = requestSegments[i];

            if (segment.isVariable()) {
                // Variable segment: capture the value.
                // Maps to: CaptureVariablePathElement.matches() which calls
                // matchingContext.set(variableName, candidateCapture, parameters)
                if (requestValue.isEmpty()) {
                    return null; // Variable must match a non-empty segment
                }
                variables.put(segment.variableName(), requestValue);
            } else {
                // Literal segment: must match exactly.
                // Maps to: LiteralPathElement.matches() which compares characters
                if (!segment.text().equals(requestValue)) {
                    return null;
                }
            }
        }

        return Collections.unmodifiableMap(variables);
    }

    /**
     * Parse the pattern string into a list of segments.
     *
     * Maps to: {@code InternalPathPatternParser} character loop (line 108-182)
     * which builds a linked list of PathElement nodes. We split on "/" and
     * classify each segment as literal or variable capture.
     */
    private static List<Segment> parse(String pattern) {
        String[] parts = splitPath(pattern);
        List<Segment> segments = new ArrayList<>(parts.length);

        for (String part : parts) {
            if (part.startsWith("{") && part.endsWith("}")) {
                // Variable capture segment: {variableName}
                String varName = part.substring(1, part.length() - 1);
                if (varName.isEmpty()) {
                    throw new IllegalArgumentException(
                            "Empty variable name in pattern: " + pattern);
                }
                segments.add(new Segment(null, varName));
            } else {
                // Literal segment
                segments.add(new Segment(part, null));
            }
        }

        return List.copyOf(segments);
    }

    /**
     * Split a path into non-empty segments.
     * "/users/{id}/posts" → ["users", "{id}", "posts"]
     */
    private static String[] splitPath(String path) {
        if (path == null || path.isEmpty() || path.equals("/")) {
            return new String[0];
        }

        // Strip leading and trailing slashes, then split
        String trimmed = path;
        if (trimmed.startsWith("/")) {
            trimmed = trimmed.substring(1);
        }
        if (trimmed.endsWith("/")) {
            trimmed = trimmed.substring(0, trimmed.length() - 1);
        }
        if (trimmed.isEmpty()) {
            return new String[0];
        }

        return trimmed.split("/");
    }

    @Override
    public String toString() {
        return patternString;
    }

    /**
     * A single segment in the URL pattern — either a literal text segment
     * or a variable capture segment.
     *
     * Maps to PathElement subclasses in the real framework:
     * - LiteralPathElement for literal text
     * - CaptureVariablePathElement for {variable} captures
     */
    record Segment(String text, String variableName) {

        boolean isVariable() {
            return variableName != null;
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/RouteKey.java` [MODIFIED]

```java
package com.simplespringmvc.mapping;

import java.util.Collections;
import java.util.Map;
import java.util.Objects;

/**
 * A composite key combining a URL pattern and HTTP method for handler lookup.
 *
 * This has no direct equivalent in real Spring — the real framework uses
 * {@code RequestMappingInfo} (a composite of 8 request conditions) as the
 * registry key. We simplify to just path pattern + method, which is enough for
 * pattern-based routing.
 *
 * The path is normalized to always start with "/" and never end with "/".
 * The HTTP method is stored in uppercase for case-insensitive comparison.
 *
 * <h3>ch07 Enhancement:</h3>
 * RouteKey now uses {@link PathPattern} instead of raw String comparison.
 * Patterns containing {@code {variable}} segments match dynamically, with
 * variable values extractable via {@link #matchAndExtract(String, String)}.
 * Patterns without variables still use fast exact-match semantics.
 *
 * RouteKey is immutable and suitable for use as a HashMap key. Note: equals/hashCode
 * still use the raw pattern string, so two RouteKeys with the same pattern text
 * and HTTP method are considered equal regardless of whether the pattern contains variables.
 */
public final class RouteKey {

    private final String path;
    private final String httpMethod;
    private final PathPattern pathPattern;

    public RouteKey(String path, String httpMethod) {
        this.path = normalizePath(path);
        this.httpMethod = httpMethod != null ? httpMethod.toUpperCase() : "";
        this.pathPattern = new PathPattern(this.path);
    }

    public String getPath() {
        return path;
    }

    public String getHttpMethod() {
        return httpMethod;
    }

    /**
     * Checks if this RouteKey matches a request path and HTTP method.
     * A RouteKey with an empty httpMethod matches any HTTP method.
     *
     * <h3>ch07 Enhancement:</h3>
     * Uses {@link PathPattern#matches(String)} for pattern-based matching instead
     * of exact String equality. Patterns like {@code /users/{id}} now match
     * request paths like {@code /users/42}.
     */
    public boolean matches(String requestPath, String requestMethod) {
        String normalizedRequestPath = normalizePath(requestPath);
        if (!pathPattern.matches(normalizedRequestPath)) {
            return false;
        }
        // Empty httpMethod means "match any"
        if (this.httpMethod.isEmpty()) {
            return true;
        }
        return this.httpMethod.equalsIgnoreCase(requestMethod);
    }

    /**
     * Match the request and extract path variable values.
     *
     * Maps to: {@code PathPattern.matchAndExtract(PathContainer)} in the real
     * framework (PathPattern.java:220), which returns a PathMatchInfo containing
     * the extracted URI template variables.
     *
     * @return extracted path variables, or null if no match
     */
    public Map<String, String> matchAndExtract(String requestPath, String requestMethod) {
        String normalizedRequestPath = normalizePath(requestPath);

        // Check HTTP method first (fast rejection)
        if (!this.httpMethod.isEmpty() && !this.httpMethod.equalsIgnoreCase(requestMethod)) {
            return null;
        }

        Map<String, String> variables = pathPattern.matchAndExtract(normalizedRequestPath);
        if (variables == null) {
            return null;
        }

        return variables;
    }

    private static String normalizePath(String path) {
        if (path == null || path.isEmpty()) {
            return "/";
        }
        // Ensure starts with /
        if (!path.startsWith("/")) {
            path = "/" + path;
        }
        // Remove trailing slash (except for root "/")
        if (path.length() > 1 && path.endsWith("/")) {
            path = path.substring(0, path.length() - 1);
        }
        return path;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof RouteKey that)) return false;
        return path.equals(that.path) && httpMethod.equals(that.httpMethod);
    }

    @Override
    public int hashCode() {
        return Objects.hash(path, httpMethod);
    }

    @Override
    public String toString() {
        if (httpMethod.isEmpty()) {
            return "ALL " + path;
        }
        return httpMethod + " " + path;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java` [MODIFIED]

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;
import jakarta.servlet.http.HttpServletRequest;

import java.lang.reflect.Method;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Scans all beans in the container for @Controller classes, discovers methods
 * annotated with @RequestMapping, and builds a registry that maps
 * (URL pattern + HTTP method) → HandlerMethod.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping}
 * which extends {@code AbstractHandlerMethodMapping<RequestMappingInfo>}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. afterPropertiesSet() → initHandlerMethods()
 *   2. initHandlerMethods() iterates all beans, calls isHandler() on each
 *   3. isHandler() checks for @Controller annotation (via MergedAnnotations)
 *   4. detectHandlerMethods() introspects each handler class for @RequestMapping methods
 *   5. getMappingForMethod() builds a RequestMappingInfo from the annotation
 *   6. registerHandlerMethod() stores (RequestMappingInfo → HandlerMethod) in MappingRegistry
 *   7. At request time, lookupHandlerMethod() matches the request against all registrations
 * </pre>
 *
 * We collapse steps 1-7 into a simpler flow:
 * <pre>
 *   1. init(BeanContainer) iterates all beans
 *   2. Checks each bean class for @Controller
 *   3. For each @Controller, scans methods for @RequestMapping
 *   4. Combines type-level + method-level paths
 *   5. Stores (RouteKey → HandlerMethod) in a LinkedHashMap
 *   6. lookupHandler() matches request path + method against RouteKeys
 * </pre>
 *
 * <h3>ch07 Enhancement:</h3>
 * Handler mapping now supports URI templates with path variables (e.g., {@code /users/{id}}).
 * When a pattern match succeeds, extracted path variables are stored as a request
 * attribute under {@link #PATH_VARIABLES_ATTRIBUTE}, making them available to the
 * {@code PathVariableArgumentResolver} during method invocation.
 *
 * Simplifications:
 * <ul>
 *   <li>No MappingRegistry inner class with ReadWriteLock</li>
 *   <li>No RequestMappingInfo composite — just RouteKey (path + method)</li>
 *   <li>No direct-path optimization (we scan all entries on every request)</li>
 *   <li>No HandlerExecutionChain wrapping (added in ch11)</li>
 *   <li>No meta-annotation detection (added in ch10)</li>
 * </ul>
 */
public class SimpleHandlerMapping {

    /**
     * Request attribute key for extracted path variables.
     *
     * Maps to: {@code HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE} which is
     * {@code "org.springframework.web.servlet.HandlerMapping.uriTemplateVariables"}
     *
     * The real framework sets this in AbstractHandlerMethodMapping.handleMatch()
     * after a successful pattern match. PathVariableMethodArgumentResolver.resolveName()
     * reads it back to resolve @PathVariable arguments.
     */
    public static final String PATH_VARIABLES_ATTRIBUTE =
            "com.simplespringmvc.mapping.SimpleHandlerMapping.pathVariables";

    private final Map<RouteKey, HandlerMethod> registry = new LinkedHashMap<>();

    /**
     * Scan all beans in the container and register handler methods.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.initHandlerMethods()} (line 217)
     * which is called from afterPropertiesSet().
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
     * Check if a bean type is a handler (i.e., annotated with @Controller).
     *
     * Maps to: {@code RequestMappingHandlerMapping.isHandler()} (line 181)
     * Real version uses: AnnotatedElementUtils.hasAnnotation(beanType, Controller.class)
     * which walks the meta-annotation hierarchy. We do a direct check for now.
     */
    private boolean isHandler(Class<?> beanType) {
        return beanType.isAnnotationPresent(Controller.class);
    }

    /**
     * Discover all @RequestMapping methods on a handler bean and register them.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.detectHandlerMethods()} (line 270)
     * Real version uses MethodIntrospector.selectMethods() to scan the class,
     * then calls getMappingForMethod() for each method.
     */
    private void detectHandlerMethods(Object bean) {
        Class<?> beanType = bean.getClass();

        // Check for type-level @RequestMapping (base path prefix)
        String basePath = "";
        RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
        if (typeMapping != null) {
            basePath = typeMapping.path();
        }

        // Scan all declared methods for @RequestMapping
        for (Method method : beanType.getDeclaredMethods()) {
            RequestMapping methodMapping = method.getAnnotation(RequestMapping.class);
            if (methodMapping == null) {
                continue;
            }

            // Combine type-level base path + method-level path
            String fullPath = combinePaths(basePath, methodMapping.path());
            String httpMethod = methodMapping.method();

            // Ensure the method is accessible via reflection.
            // Maps to: ReflectionUtils.makeAccessible() used in Spring's
            // InvocableHandlerMethod. Needed when the declaring class
            // is package-private (common in tests, inner classes, etc.)
            method.setAccessible(true);

            RouteKey routeKey = new RouteKey(fullPath, httpMethod);
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
     *
     * Maps to: {@code AbstractHandlerMethodMapping.lookupHandlerMethod()} (line 393)
     *
     * The real version first tries a fast direct-path lookup, then falls back
     * to scanning all mappings. We always scan (simple but O(n)).
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
     * Look up the handler method for a given HTTP request, storing extracted
     * path variables as a request attribute.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.getHandlerInternal()} (line 337)
     * which calls lookupHandlerMethod(), then handleMatch() which stores the
     * extracted URI template variables as a request attribute under
     * {@code HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE}.
     *
     * <h3>ch07 Enhancement:</h3>
     * This overload takes the full HttpServletRequest so it can:
     * <ol>
     *   <li>Match the request path against pattern-based RouteKeys</li>
     *   <li>Extract path variable values (e.g., {@code {id}} → "42")</li>
     *   <li>Store them as a request attribute for the PathVariableArgumentResolver</li>
     * </ol>
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
                // Store extracted path variables on the request for argument resolvers.
                // Maps to: AbstractHandlerMethodMapping.handleMatch() which calls
                // request.setAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, uriVariables)
                request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Returns an unmodifiable view of all registered mappings.
     * Useful for debugging and testing.
     */
    public Map<RouteKey, HandlerMethod> getRegisteredMappings() {
        return Map.copyOf(registry);
    }

    /**
     * Combine a base path and method path.
     * Handles cases like "/" + "/users" → "/users", "/api" + "/users" → "/api/users".
     */
    private String combinePaths(String basePath, String methodPath) {
        if (basePath == null || basePath.isEmpty()) {
            return methodPath;
        }
        if (methodPath == null || methodPath.isEmpty()) {
            return basePath;
        }
        // Remove trailing slash from base, ensure method starts with /
        String base = basePath.endsWith("/") ? basePath.substring(0, basePath.length() - 1) : basePath;
        String method = methodPath.startsWith("/") ? methodPath : "/" + methodPath;
        return base + method;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/PathVariableArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.http.HttpServletRequest;

import java.util.Map;

/**
 * Resolves method arguments annotated with {@link PathVariable} by reading
 * extracted URI template variable values from a request attribute.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.PathVariableMethodArgumentResolver}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@link SimpleHandlerMapping#lookupHandler(HttpServletRequest)} matches the request
 *       path against a pattern like {@code /users/{id}} and stores the extracted variables
 *       ({@code {"id" → "42"}}) as a request attribute</li>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @PathVariable}</li>
 *   <li>{@code resolveArgument()} reads the extracted variables map from the request
 *       attribute and looks up the value by variable name</li>
 * </ol>
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   PathVariableMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver
 *   1. supportsParameter() checks for @PathVariable annotation
 *   2. createNamedValueInfo() extracts name, required, defaultValue from the annotation
 *   3. resolveName() reads the Map from request attribute HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE
 *   4. Base class handles type conversion via WebDataBinder.convertIfNecessary()
 *   5. handleResolvedValue() stores the type-converted value in View.PATH_VARIABLES attribute
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Doesn't extend AbstractNamedValueMethodArgumentResolver — standalone resolver</li>
 *   <li>Only handles String parameters — no type conversion (ch09 adds ConversionService)</li>
 *   <li>No {@code required} flag handling</li>
 *   <li>No {@code View.PATH_VARIABLES} attribute storage</li>
 *   <li>No support for Map parameters (all path vars as Map)</li>
 * </ul>
 */
public class PathVariableArgumentResolver implements HandlerMethodArgumentResolver {

    /**
     * Supports parameters annotated with {@code @PathVariable}.
     *
     * Maps to: {@code PathVariableMethodArgumentResolver.supportsParameter()} (line 73)
     * The real version also checks if the parameter type is Map (without a specific
     * variable name). We only handle the simple case.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(PathVariable.class);
    }

    /**
     * Extract the path variable value from the request attributes.
     *
     * Maps to: {@code PathVariableMethodArgumentResolver.resolveName()} (line 93)
     * which reads from {@code HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE}
     * and returns {@code uriTemplateVars.get(name)}.
     *
     * @throws IllegalStateException if the path variable is not found
     */
    @Override
    @SuppressWarnings("unchecked")
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String variableName = getVariableName(parameter);

        // Read the path variables map stored by SimpleHandlerMapping during lookup.
        // Maps to: PathVariableMethodArgumentResolver.resolveName() (line 93):
        //   Map<String, String> uriTemplateVars = (Map<String, String>) request.getAttribute(
        //       HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
        Map<String, String> pathVariables = (Map<String, String>) request.getAttribute(
                SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE);

        if (pathVariables == null) {
            throw new IllegalStateException(
                    "No path variables found in request. Was the handler resolved by SimpleHandlerMapping?");
        }

        String value = pathVariables.get(variableName);
        if (value == null) {
            // Maps to: PathVariableMethodArgumentResolver.handleMissingValue() (line 100)
            // which throws MissingPathVariableException
            throw new IllegalStateException(
                    "Missing path variable '" + variableName + "' for method parameter type "
                            + parameter.getParameterType().getSimpleName()
                            + ". Available path variables: " + pathVariables.keySet());
        }

        return value;
    }

    /**
     * Determine the path variable name: use the annotation's value() if
     * specified, otherwise fall back to the method parameter name.
     *
     * Maps to: {@code PathVariableMethodArgumentResolver.createNamedValueInfo()} (line 85)
     */
    private String getVariableName(MethodParameter parameter) {
        PathVariable annotation = parameter.getParameterAnnotation(PathVariable.class);
        String name = (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
        return name;
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
        // Real version registers ~30 resolvers in a specific order.
        // ORDER MATTERS: resolvers are checked in registration order. @PathVariable
        // comes before @RequestParam to match the real framework's ordering.
        argumentResolvers.addResolver(new PathVariableArgumentResolver());
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
        argumentResolvers.addResolver(new PathVariableArgumentResolver());
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

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

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
 *   <li>ch13: view rendering</li>
 * </ul>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No locale/theme resolution (added in ch20)</li>
 *   <li>No flash map management (added in ch18)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Called by Tomcat once the Servlet is registered. Initializes strategy
     * components (handler mappings, adapters, etc.) from the bean container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * which is called from {@code onRefresh()}, which is triggered by
     * {@code FrameworkServlet.initWebApplicationContext()}.
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
        // Future features will initialize:
        // - ExceptionResolvers (ch12)
        // - ViewResolvers (ch13)
    }

    /**
     * Create the handler mapping and scan all beans for @Controller methods.
     *
     * Maps to: {@code DispatcherServlet.initHandlerMappings()} (line 505)
     * Real version looks for HandlerMapping beans in the context. We create
     * one directly and initialize it with our bean container.
     */
    private void initHandlerMapping() {
        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(beanContainer);
    }

    /**
     * Create the handler adapter that knows how to invoke handler methods.
     *
     * Maps to: {@code DispatcherServlet.initHandlerAdapters()} (line 552)
     * Real version finds all HandlerAdapter beans in the ApplicationContext
     * (or falls back to defaults from DispatcherServlet.properties).
     * The default adapters include:
     * <ul>
     *   <li>HttpRequestHandlerAdapter</li>
     *   <li>SimpleControllerHandlerAdapter</li>
     *   <li>RequestMappingHandlerAdapter</li>
     * </ul>
     *
     * We create just one adapter directly — the one that handles our
     * HandlerMethod type via reflection.
     */
    private void initHandlerAdapter() {
        handlerAdapter = new SimpleHandlerAdapter();
    }

    /**
     * Override service() to route ALL HTTP methods through doDispatch().
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870) which overrides
     * HttpServlet.service() and routes GET/POST/PUT/DELETE/PATCH/OPTIONS/TRACE
     * all through {@code processRequest()}, which calls {@code doService()},
     * which calls {@code doDispatch()}.
     *
     * We collapse that chain: service() → doDispatch() directly.
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
     * The real doDispatch() flow:
     * <pre>
     *   1. checkMultipart(request)
     *   2. mappedHandler = getHandler(request)         ← ch03
     *   3. mappedHandler.applyPreHandle()              ← ch11
     *   4. ha = getHandlerAdapter(handler)             ← ch04 ✓
     *   5. ha.handle(request, response, handler)       ← ch04 ✓
     *   6. mappedHandler.applyPostHandle()             ← ch11
     *   7. processDispatchResult(mv, exception)        ← ch12, ch13
     * </pre>
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        // Step 1: Look up the handler for this request (ch03)
        HandlerMethod handler = getHandler(request);

        if (handler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    "No handler found for " + request.getMethod() + " " + request.getRequestURI());
            return;
        }

        // Step 2: Find a HandlerAdapter that supports this handler (ch04)
        HandlerAdapter adapter = getHandlerAdapter(handler);

        // Step 3: Delegate invocation + response writing to the adapter (ch04)
        adapter.handle(request, response, handler);
    }

    /**
     * Find the HandlerMethod that matches this request.
     *
     * Maps to: {@code DispatcherServlet.getHandler()} (line 1103)
     * Real version iterates a list of HandlerMapping beans and returns the
     * first non-null HandlerExecutionChain. We have just one mapping.
     *
     * <h3>ch07 Enhancement:</h3>
     * Now passes the full HttpServletRequest to the handler mapping so that
     * extracted path variables (from URI templates like {@code /users/{id}})
     * can be stored as request attributes for argument resolvers to read.
     */
    private HandlerMethod getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        return handlerMapping.lookupHandler(request);
    }

    /**
     * Find a HandlerAdapter that can handle the given handler.
     *
     * Maps to: {@code DispatcherServlet.getHandlerAdapter()} (line 1185)
     * Real version iterates a list of HandlerAdapters and returns the first
     * one where {@code adapter.supports(handler)} is true.
     *
     * We have just one adapter, so we check it directly. The real framework
     * throws ServletException if no adapter supports the handler — we do the same.
     */
    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
    }

    /**
     * Provides access to the bean container for strategy initialization.
     * Future features use this to look up HandlerMappings, HandlerAdapters, etc.
     */
    public BeanContainer getBeanContainer() {
        return beanContainer;
    }

    /**
     * Provides access to the handler mapping for testing and inspection.
     */
    public SimpleHandlerMapping getHandlerMapping() {
        return handlerMapping;
    }

    /**
     * Provides access to the handler adapter for testing and inspection.
     */
    public HandlerAdapter getHandlerAdapter() {
        return handlerAdapter;
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/mapping/PathPatternTest.java` [NEW]

```java
package com.simplespringmvc.mapping;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link PathPattern}.
 */
class PathPatternTest {

    // ─── Literal matching ────────────────────────────────────────────

    @Nested
    @DisplayName("Literal pattern matching")
    class LiteralMatching {

        @Test
        @DisplayName("should match exact literal path")
        void shouldMatchExactLiteralPath() {
            PathPattern pattern = new PathPattern("/users");
            assertThat(pattern.matches("/users")).isTrue();
        }

        @Test
        @DisplayName("should not match different literal path")
        void shouldNotMatchDifferentLiteralPath() {
            PathPattern pattern = new PathPattern("/users");
            assertThat(pattern.matches("/orders")).isFalse();
        }

        @Test
        @DisplayName("should match root path")
        void shouldMatchRootPath() {
            PathPattern pattern = new PathPattern("/");
            assertThat(pattern.matches("/")).isTrue();
        }

        @Test
        @DisplayName("should match multi-segment literal path")
        void shouldMatchMultiSegmentLiteralPath() {
            PathPattern pattern = new PathPattern("/api/v1/users");
            assertThat(pattern.matches("/api/v1/users")).isTrue();
        }

        @Test
        @DisplayName("should not match when segment count differs")
        void shouldNotMatchDifferentSegmentCount() {
            PathPattern pattern = new PathPattern("/users");
            assertThat(pattern.matches("/users/123")).isFalse();
        }

        @Test
        @DisplayName("should extract empty map for literal match")
        void shouldExtractEmptyMap_WhenLiteralMatch() {
            PathPattern pattern = new PathPattern("/users");
            Map<String, String> variables = pattern.matchAndExtract("/users");
            assertThat(variables).isNotNull().isEmpty();
        }
    }

    // ─── Variable matching ───────────────────────────────────────────

    @Nested
    @DisplayName("Variable pattern matching")
    class VariableMatching {

        @Test
        @DisplayName("should match path with single variable")
        void shouldMatchSingleVariable() {
            PathPattern pattern = new PathPattern("/users/{id}");
            assertThat(pattern.matches("/users/42")).isTrue();
        }

        @Test
        @DisplayName("should extract single variable value")
        void shouldExtractSingleVariable() {
            PathPattern pattern = new PathPattern("/users/{id}");
            Map<String, String> variables = pattern.matchAndExtract("/users/42");
            assertThat(variables).containsEntry("id", "42");
        }

        @Test
        @DisplayName("should match path with multiple variables")
        void shouldMatchMultipleVariables() {
            PathPattern pattern = new PathPattern("/users/{userId}/posts/{postId}");
            assertThat(pattern.matches("/users/1/posts/99")).isTrue();
        }

        @Test
        @DisplayName("should extract multiple variable values")
        void shouldExtractMultipleVariables() {
            PathPattern pattern = new PathPattern("/users/{userId}/posts/{postId}");
            Map<String, String> variables = pattern.matchAndExtract("/users/1/posts/99");
            assertThat(variables)
                    .containsEntry("userId", "1")
                    .containsEntry("postId", "99");
        }

        @Test
        @DisplayName("should not match when variable segment is empty")
        void shouldNotMatchEmptyVariableSegment() {
            // URL "/users//posts" would have an empty segment for {id}
            PathPattern pattern = new PathPattern("/users/{id}");
            assertThat(pattern.matches("/users/")).isFalse();
        }

        @Test
        @DisplayName("should match variable with non-numeric value")
        void shouldMatchNonNumericVariable() {
            PathPattern pattern = new PathPattern("/users/{username}");
            Map<String, String> variables = pattern.matchAndExtract("/users/alice");
            assertThat(variables).containsEntry("username", "alice");
        }

        @Test
        @DisplayName("should not match when trailing segments are missing")
        void shouldNotMatchMissingTrailingSegments() {
            PathPattern pattern = new PathPattern("/users/{id}/posts");
            assertThat(pattern.matches("/users/42")).isFalse();
        }

        @Test
        @DisplayName("should return null from matchAndExtract when no match")
        void shouldReturnNull_WhenNoMatch() {
            PathPattern pattern = new PathPattern("/users/{id}");
            Map<String, String> variables = pattern.matchAndExtract("/orders/42");
            assertThat(variables).isNull();
        }
    }

    // ─── Mixed literal and variable ──────────────────────────────────

    @Nested
    @DisplayName("Mixed literal and variable segments")
    class MixedMatching {

        @Test
        @DisplayName("should match when literal segments match and variable captures")
        void shouldMatchMixed() {
            PathPattern pattern = new PathPattern("/api/users/{id}/profile");
            Map<String, String> variables = pattern.matchAndExtract("/api/users/42/profile");
            assertThat(variables).containsEntry("id", "42");
        }

        @Test
        @DisplayName("should not match when literal segment differs")
        void shouldNotMatchWhenLiteralDiffers() {
            PathPattern pattern = new PathPattern("/api/users/{id}/profile");
            assertThat(pattern.matches("/api/users/42/settings")).isFalse();
        }
    }

    // ─── hasVariables ────────────────────────────────────────────────

    @Nested
    @DisplayName("hasVariables()")
    class HasVariables {

        @Test
        @DisplayName("should return false for literal pattern")
        void shouldReturnFalse_WhenLiteralPattern() {
            PathPattern pattern = new PathPattern("/users");
            assertThat(pattern.hasVariables()).isFalse();
        }

        @Test
        @DisplayName("should return true for pattern with variable")
        void shouldReturnTrue_WhenPatternHasVariable() {
            PathPattern pattern = new PathPattern("/users/{id}");
            assertThat(pattern.hasVariables()).isTrue();
        }
    }

    // ─── Path normalization ──────────────────────────────────────────

    @Nested
    @DisplayName("Path normalization in matching")
    class PathNormalization {

        @Test
        @DisplayName("should match with trailing slash on request")
        void shouldMatchWithTrailingSlash() {
            PathPattern pattern = new PathPattern("/users/{id}");
            assertThat(pattern.matches("/users/42/")).isTrue();
        }

        @Test
        @DisplayName("should match pattern without leading slash")
        void shouldMatchPatternWithoutLeadingSlash() {
            PathPattern pattern = new PathPattern("users/{id}");
            assertThat(pattern.matches("/users/42")).isTrue();
        }
    }

    // ─── Error handling ──────────────────────────────────────────────

    @Nested
    @DisplayName("Error handling")
    class ErrorHandling {

        @Test
        @DisplayName("should throw on empty variable name")
        void shouldThrowOnEmptyVariableName() {
            assertThatThrownBy(() -> new PathPattern("/users/{}"))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("Empty variable name");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/PathVariableArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import jakarta.servlet.http.HttpServletRequest;
import java.lang.reflect.Method;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.when;

/**
 * Unit tests for {@link PathVariableArgumentResolver}.
 */
class PathVariableArgumentResolverTest {

    private PathVariableArgumentResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        resolver = new PathVariableArgumentResolver();
        request = Mockito.mock(HttpServletRequest.class);
    }

    // ─── Test helper methods ─────────────────────────────────────

    @SuppressWarnings("unused")
    static void handlerWithPathVariable(@PathVariable("id") String id) {}

    @SuppressWarnings("unused")
    static void handlerWithImplicitName(@PathVariable String userId) {}

    @SuppressWarnings("unused")
    static void handlerWithRequestParam(@RequestParam("q") String query) {}

    @SuppressWarnings("unused")
    static void handlerWithNoAnnotation(String plain) {}

    private MethodParameter parameterFor(String methodName) throws NoSuchMethodException {
        for (Method m : PathVariableArgumentResolverTest.class.getDeclaredMethods()) {
            if (m.getName().equals(methodName)) {
                return new MethodParameter(m, 0);
            }
        }
        throw new NoSuchMethodException(methodName);
    }

    // ─── supportsParameter ───────────────────────────────────────

    @Nested
    @DisplayName("supportsParameter")
    class SupportsParameter {

        @Test
        @DisplayName("should support parameter annotated with @PathVariable")
        void shouldSupportPathVariableParameter() throws Exception {
            MethodParameter param = parameterFor("handlerWithPathVariable");
            assertThat(resolver.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should support parameter with implicit @PathVariable name")
        void shouldSupportImplicitNameParameter() throws Exception {
            MethodParameter param = parameterFor("handlerWithImplicitName");
            assertThat(resolver.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should not support parameter with @RequestParam")
        void shouldNotSupportRequestParamParameter() throws Exception {
            MethodParameter param = parameterFor("handlerWithRequestParam");
            assertThat(resolver.supportsParameter(param)).isFalse();
        }

        @Test
        @DisplayName("should not support unannotated parameter")
        void shouldNotSupportUnannotatedParameter() throws Exception {
            MethodParameter param = parameterFor("handlerWithNoAnnotation");
            assertThat(resolver.supportsParameter(param)).isFalse();
        }
    }

    // ─── resolveArgument ─────────────────────────────────────────

    @Nested
    @DisplayName("resolveArgument")
    class ResolveArgument {

        @Test
        @DisplayName("should resolve path variable by explicit name")
        void shouldResolveByExplicitName() throws Exception {
            when(request.getAttribute(SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE))
                    .thenReturn(Map.of("id", "42"));

            MethodParameter param = parameterFor("handlerWithPathVariable");
            Object result = resolver.resolveArgument(param, request);

            assertThat(result).isEqualTo("42");
        }

        @Test
        @DisplayName("should resolve path variable by implicit parameter name")
        void shouldResolveByImplicitName() throws Exception {
            when(request.getAttribute(SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE))
                    .thenReturn(Map.of("userId", "7"));

            MethodParameter param = parameterFor("handlerWithImplicitName");
            Object result = resolver.resolveArgument(param, request);

            assertThat(result).isEqualTo("7");
        }

        @Test
        @DisplayName("should throw when no path variables on request")
        void shouldThrow_WhenNoPathVariablesAttribute() throws Exception {
            when(request.getAttribute(SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE))
                    .thenReturn(null);

            MethodParameter param = parameterFor("handlerWithPathVariable");

            assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("No path variables found");
        }

        @Test
        @DisplayName("should throw when specific path variable is missing")
        void shouldThrow_WhenVariableMissing() throws Exception {
            when(request.getAttribute(SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE))
                    .thenReturn(Map.of("otherId", "99"));

            MethodParameter param = parameterFor("handlerWithPathVariable");

            assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("Missing path variable 'id'");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/PathVariableIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for Path Variables & Pattern Matching (ch07).
 *
 * Verifies the full end-to-end flow: HTTP request → pattern matching →
 * path variable extraction → argument resolution → controller method invocation.
 */
class PathVariableIntegrationTest {

    // ─── Test controllers ────────────────────────────────────────

    @Controller
    static class UserController {

        @RequestMapping(path = "/users/{id}", method = "GET")
        public String getUser(@PathVariable("id") String id) {
            return "User " + id;
        }

        @RequestMapping(path = "/users/{userId}/posts/{postId}", method = "GET")
        public String getUserPost(
                @PathVariable("userId") String userId,
                @PathVariable("postId") String postId) {
            return "User " + userId + " Post " + postId;
        }
    }

    @Controller
    static class ImplicitNameController {

        @RequestMapping(path = "/products/{productId}", method = "GET")
        public String getProduct(@PathVariable String productId) {
            return "Product " + productId;
        }
    }

    @Controller
    static class MixedController {

        @RequestMapping(path = "/search/{category}", method = "GET")
        public String search(
                @PathVariable("category") String category,
                @RequestParam("q") String query) {
            return "Searching " + category + " for '" + query + "'";
        }
    }

    @Controller
    static class JsonController {

        @RequestMapping(path = "/api/items/{id}", method = "GET")
        @ResponseBody
        public Item getItem(@PathVariable("id") String id) {
            return new Item(id, "Item " + id);
        }

        record Item(String id, String name) {}
    }

    @Controller
    static class StaticAndPatternController {

        @RequestMapping(path = "/info", method = "GET")
        public String info() {
            return "static info";
        }

        @RequestMapping(path = "/info/{topic}", method = "GET")
        public String infoTopic(@PathVariable("topic") String topic) {
            return "info about " + topic;
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new UserController());
        container.registerBean(new ImplicitNameController());
        container.registerBean(new MixedController());
        container.registerBean(new JsonController());
        container.registerBean(new StaticAndPatternController());

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        baseUrl = "http://localhost:" + tomcat.getPort();
        httpClient = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        tomcat.stop();
    }

    // ─── Single path variable ────────────────────────────────────

    @Test
    @DisplayName("should resolve single @PathVariable from URL")
    void shouldResolveSinglePathVariable() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/users/42")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("User 42");
    }

    @Test
    @DisplayName("should resolve path variable with string value")
    void shouldResolveStringPathVariable() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/users/alice")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("User alice");
    }

    // ─── Multiple path variables ─────────────────────────────────

    @Test
    @DisplayName("should resolve multiple @PathVariable parameters")
    void shouldResolveMultiplePathVariables() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/users/1/posts/99")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("User 1 Post 99");
    }

    // ─── Implicit parameter name ─────────────────────────────────

    @Test
    @DisplayName("should resolve @PathVariable using method parameter name as fallback")
    void shouldResolveByImplicitName() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/products/ABC-123")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Product ABC-123");
    }

    // ─── Mixed @PathVariable and @RequestParam ───────────────────

    @Test
    @DisplayName("should resolve both @PathVariable and @RequestParam in same method")
    void shouldResolveMixedAnnotations() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/search/electronics?q=laptop")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Searching electronics for 'laptop'");
    }

    // ─── @PathVariable with @ResponseBody (JSON) ─────────────────

    @Test
    @DisplayName("should resolve @PathVariable with @ResponseBody JSON response")
    void shouldResolvePathVariableWithJsonResponse() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/api/items/7")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("\"id\":\"7\"");
        assertThat(response.body()).contains("\"name\":\"Item 7\"");
    }

    // ─── Static and pattern routes coexist ───────────────────────

    @Test
    @DisplayName("should match static route when it comes first")
    void shouldMatchStaticRoute() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/info")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("static info");
    }

    @Test
    @DisplayName("should match pattern route for dynamic path")
    void shouldMatchPatternRoute() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/info/spring")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("info about spring");
    }

    // ─── 404 for unmatched patterns ──────────────────────────────

    @Test
    @DisplayName("should return 404 when path doesn't match any pattern")
    void shouldReturn404_WhenNoPatternMatches() throws Exception {
        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/nonexistent/path/here")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(404);
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **URI Template** | A path pattern with `{variable}` placeholders, e.g., `/users/{id}` |
| **PathPattern** | A compiled representation of a URI template that can match request paths and extract variable values |
| **@PathVariable** | Annotation binding a method parameter to a URI template variable |
| **Request Attribute Bridge** | Using `request.setAttribute()` to pass data between handler mapping (extraction) and argument resolver (consumption) |
| **Segment-based matching** | Walking pattern and request path segments in lockstep — literals must match exactly, variables capture any non-empty value |

**Next: Chapter 8 — @RequestBody Deserialization** — Deserialize JSON request bodies into Java objects, completing the request side of the `HttpMessageConverter` pattern (ch06 handled the response side with `@ResponseBody`).
