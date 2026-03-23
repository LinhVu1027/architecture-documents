# Chapter 13: View Resolution & Rendering

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Handler methods either return `@ResponseBody` JSON or write text/plain via `StringReturnValueHandler` — there's no way to render HTML templates with dynamic data | Controllers cannot return HTML pages with model data; every response is either raw JSON or plain text | Build a view resolution pipeline: `ViewResolver` resolves view names to `View` objects, `TemplateView` renders HTML templates with `${placeholder}` substitution, and `ModelAndView` carries the view name + model from the adapter to the dispatcher for rendering |

---

## 13.1 The Integration Point

The integration point is `SimpleDispatcherServlet.doDispatch()` and `HandlerAdapter.handle()`. Currently, `handle()` returns `void` — the adapter writes to the response directly. For view resolution, we need `handle()` to return a `ModelAndView` that tells the dispatcher "resolve this view name and render it." When the return is null, it means "response already handled" (e.g., @ResponseBody wrote JSON).

**Modifying:** `src/main/java/com/simplespringmvc/adapter/HandlerAdapter.java`
**Change:** Return type from `void` to `ModelAndView`

```java
public interface HandlerAdapter {

    boolean supports(Object handler);

    // ch13: Changed from void to ModelAndView
    // null = response already written (@ResponseBody)
    // non-null = resolve and render this view
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                        Object handler) throws Exception;
}
```

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Capture the ModelAndView from `handle()`, add `render()` step after dispatch

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;              // ← NEW: capture the ModelAndView
    Exception dispatchException = null;

    try {
        mappedHandler = getHandler(request);
        // ... interceptor preHandle ...

        HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
        mv = adapter.handle(request, response, mappedHandler.getHandler());  // ← returns ModelAndView

        // ... interceptor postHandle ...
    } catch (Exception ex) {
        dispatchException = ex;
    }

    // ... afterCompletion, exception handling ...

    // NEW: View resolution and rendering
    if (mv != null && mv.getViewName() != null) {
        render(mv, request, response);
    }
}
```

Two key decisions here:

1. **Why return `ModelAndView` from `handle()` instead of having the adapter render directly?** Separation of concerns — the adapter knows how to invoke handlers and process return values, but the dispatcher owns the ViewResolver chain. This split mirrors the real framework exactly (`DispatcherServlet.doDispatch()` line 963 captures `mv = ha.handle(...)`, then line 980 calls `processDispatchResult()`).

2. **Why null signals "response handled"?** In the real framework, this is done via `ModelAndViewContainer.requestHandled = true`. Returning null is our simplified equivalent — it avoids introducing a mutable container object.

This connects the **return value handler chain** to the **view resolution pipeline**. To make it work, we need to build:
- `ModelAndView` — the carrier between adapter and dispatcher
- `View` and `ViewResolver` — the resolution and rendering interfaces
- `TemplateView` and `TemplateViewResolver` — concrete implementations
- `ViewNameReturnValueHandler` — interprets String returns as view names
- `ModelAndViewReturnValueHandler` — passes through direct ModelAndView returns
- Updates to the return value handler interface (return ModelAndView instead of void)

## 13.2 ModelAndView — The Carrier

The `ModelAndView` carries both the logical view name and the model data from the adapter back to the dispatcher.

**New file:** `src/main/java/com/simplespringmvc/view/ModelAndView.java`

```java
package com.simplespringmvc.view;

import java.util.LinkedHashMap;
import java.util.Map;

public class ModelAndView {

    private String viewName;
    private final Map<String, Object> model = new LinkedHashMap<>();

    public ModelAndView() {}

    public ModelAndView(String viewName) {
        this.viewName = viewName;
    }

    public ModelAndView(String viewName, Map<String, ?> model) {
        this.viewName = viewName;
        if (model != null) {
            this.model.putAll(model);
        }
    }

    public ModelAndView addObject(String name, Object value) {
        model.put(name, value);
        return this;  // fluent chaining
    }

    public String getViewName() { return viewName; }
    public void setViewName(String viewName) { this.viewName = viewName; }
    public Map<String, Object> getModel() { return model; }
}
```

## 13.3 View and ViewResolver — The Contracts

**New file:** `src/main/java/com/simplespringmvc/view/View.java`

```java
package com.simplespringmvc.view;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.Map;

public interface View {

    default String getContentType() { return null; }

    void render(Map<String, ?> model, HttpServletRequest request,
                HttpServletResponse response) throws Exception;
}
```

**New file:** `src/main/java/com/simplespringmvc/view/ViewResolver.java`

```java
package com.simplespringmvc.view;

public interface ViewResolver {

    // Returns null if this resolver can't resolve the name (allows chaining)
    View resolveViewName(String viewName) throws Exception;
}
```

## 13.4 TemplateView — HTML Rendering

**New file:** `src/main/java/com/simplespringmvc/view/TemplateView.java`

```java
package com.simplespringmvc.view;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class TemplateView implements View {

    private static final Pattern PLACEHOLDER_PATTERN = Pattern.compile("\\$\\{([^}]+)}");
    private final String templatePath;

    public TemplateView(String templatePath) {
        this.templatePath = templatePath;
    }

    @Override
    public String getContentType() { return "text/html"; }

    @Override
    public void render(Map<String, ?> model, HttpServletRequest request,
                       HttpServletResponse response) throws Exception {
        String template = loadTemplate();
        String rendered = replacePlaceholders(template, model);

        response.setContentType("text/html");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(rendered);
        response.getWriter().flush();
    }

    private String loadTemplate() throws Exception {
        InputStream is = getClass().getResourceAsStream(templatePath);
        if (is == null) throw new IllegalStateException("Template not found: " + templatePath);
        try (is) { return new String(is.readAllBytes(), StandardCharsets.UTF_8); }
    }

    private String replacePlaceholders(String template, Map<String, ?> model) {
        if (model == null || model.isEmpty()) return template;

        Matcher matcher = PLACEHOLDER_PATTERN.matcher(template);
        StringBuilder result = new StringBuilder();
        while (matcher.find()) {
            String key = matcher.group(1);
            Object value = model.get(key);
            String replacement = (value != null) ? value.toString() : matcher.group(0);
            matcher.appendReplacement(result, Matcher.quoteReplacement(replacement));
        }
        matcher.appendTail(result);
        return result.toString();
    }
}
```

## 13.5 TemplateViewResolver — Name to View

**New file:** `src/main/java/com/simplespringmvc/view/TemplateViewResolver.java`

```java
package com.simplespringmvc.view;

import java.io.InputStream;

public class TemplateViewResolver implements ViewResolver {

    private String prefix = "";
    private String suffix = "";

    public TemplateViewResolver() {}

    public TemplateViewResolver(String prefix, String suffix) {
        this.prefix = prefix;
        this.suffix = suffix;
    }

    @Override
    public View resolveViewName(String viewName) {
        String path = prefix + viewName + suffix;
        InputStream resource = getClass().getResourceAsStream(path);
        if (resource != null) {
            try { resource.close(); } catch (Exception ignored) {}
            return new TemplateView(path);
        }
        return null;  // allows chaining — next resolver gets a chance
    }
}
```

## 13.6 Return Value Handler Changes

The `HandlerMethodReturnValueHandler` interface now returns `ModelAndView` instead of void. This reflects the fundamental duality: handlers either write the response directly (returning null) or produce a view name for the dispatcher to render.

**New file:** `src/main/java/com/simplespringmvc/adapter/ViewNameReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public class ViewNameReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return CharSequence.class.isAssignableFrom(handlerMethod.getMethod().getReturnType());
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) {
        if (returnValue instanceof CharSequence viewName) {
            return new ModelAndView(viewName.toString());
        }
        return null;
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/adapter/ModelAndViewReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public class ModelAndViewReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return ModelAndView.class.isAssignableFrom(handlerMethod.getMethod().getReturnType());
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) {
        return (returnValue instanceof ModelAndView mv) ? mv : null;
    }
}
```

**Modified:** `ResponseBodyReturnValueHandler` — now returns `null` (response was written directly)

**Modified:** `StringReturnValueHandler` — now returns `null` (catch-all fallback, writes text/plain)

The adapter's handler chain order becomes:
```
1. ResponseBodyReturnValueHandler  → @ResponseBody → JSON, returns null
2. ModelAndViewReturnValueHandler  → ModelAndView returns → pass through
3. ViewNameReturnValueHandler      → String returns → ModelAndView(viewName)
4. StringReturnValueHandler        → catch-all → text/plain, returns null
```

## 13.7 Try It Yourself

<details>
<summary>Challenge: Build a TemplateView that loads HTML and substitutes ${placeholders}</summary>

Given this template at `/templates/hello.html`:
```html
<html>
<body><h1>Hello, ${name}!</h1></body>
</html>
```

And this model: `Map.of("name", "Alice")`

The rendered output should be:
```html
<html>
<body><h1>Hello, Alice!</h1></body>
</html>
```

Hint: Use `Pattern.compile("\\$\\{([^}]+)}")` to find placeholders, and `Matcher.appendReplacement()` / `appendTail()` to build the result.

```java
private String replacePlaceholders(String template, Map<String, ?> model) {
    if (model == null || model.isEmpty()) return template;

    Matcher matcher = PLACEHOLDER_PATTERN.matcher(template);
    StringBuilder result = new StringBuilder();
    while (matcher.find()) {
        String key = matcher.group(1);
        Object value = model.get(key);
        String replacement = (value != null) ? value.toString() : matcher.group(0);
        matcher.appendReplacement(result, Matcher.quoteReplacement(replacement));
    }
    matcher.appendTail(result);
    return result.toString();
}
```

</details>

<details>
<summary>Challenge: Implement the DispatcherServlet's render() method</summary>

The render method should:
1. Extract the view name from the ModelAndView
2. Iterate ViewResolvers to find a View
3. Call `view.render(model, request, response)`
4. Fall back to text/plain if no resolver matches

```java
private void render(ModelAndView mv, HttpServletRequest request,
                    HttpServletResponse response) throws Exception {
    String viewName = mv.getViewName();
    View view = resolveViewName(viewName);

    if (view != null) {
        view.render(mv.getModel(), request, response);
    } else {
        // Fallback: write view name as text/plain
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(viewName);
        response.getWriter().flush();
    }
}

private View resolveViewName(String viewName) throws Exception {
    for (ViewResolver resolver : viewResolvers) {
        View view = resolver.resolveViewName(viewName);
        if (view != null) return view;
    }
    return null;
}
```

</details>

## 13.8 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/view/ModelAndViewTest.java`

```java
@Test
@DisplayName("should create with view name and model")
void shouldCreateWithViewNameAndModel() {
    ModelAndView mv = new ModelAndView("profile", Map.of("name", "Alice", "age", 30));
    assertThat(mv.getViewName()).isEqualTo("profile");
    assertThat(mv.getModel()).containsEntry("name", "Alice");
}

@Test
@DisplayName("should add objects fluently")
void shouldAddObjectsFluently() {
    ModelAndView mv = new ModelAndView("view")
            .addObject("name", "Bob")
            .addObject("age", 25);
    assertThat(mv.getModel()).hasSize(2);
}
```

**New file:** `src/test/java/com/simplespringmvc/view/TemplateViewTest.java`

```java
@Test
@DisplayName("should replace placeholders with model values")
void shouldReplacePlaceholders_WhenModelHasValues() throws Exception {
    TemplateView view = new TemplateView("/templates/hello.html");
    view.render(Map.of("name", "Alice"), request, response);
    assertThat(responseBody.toString()).contains("Hello, Alice!");
    assertThat(responseBody.toString()).doesNotContain("${name}");
}

@Test
@DisplayName("should leave unmatched placeholders as-is")
void shouldLeaveUnmatchedPlaceholders() throws Exception {
    TemplateView view = new TemplateView("/templates/greeting.html");
    view.render(Map.of("title", "Greetings"), request, response);
    assertThat(responseBody.toString()).contains("${name}");
}
```

**New file:** `src/test/java/com/simplespringmvc/view/TemplateViewResolverTest.java`

```java
@Test
@DisplayName("should resolve view name using prefix and suffix")
void shouldResolveViewName_WhenTemplateExists() throws Exception {
    TemplateViewResolver resolver = new TemplateViewResolver("/templates/", ".html");
    View view = resolver.resolveViewName("hello");
    assertThat(view).isNotNull().isInstanceOf(TemplateView.class);
}

@Test
@DisplayName("should return null when template does not exist")
void shouldReturnNull_WhenTemplateDoesNotExist() throws Exception {
    TemplateViewResolver resolver = new TemplateViewResolver("/templates/", ".html");
    assertThat(resolver.resolveViewName("nonexistent")).isNull();
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/ViewResolutionIntegrationTest.java`

```java
@Test
@DisplayName("should render template with model data from ModelAndView")
void shouldRenderTemplate_WhenModelAndViewReturned() throws Exception {
    HttpResponse<String> response = httpClient.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/hello?name=Alice")).GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).contains("Hello, Alice!");
    assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("text/html");
}

@Test
@DisplayName("should bypass view resolution for @RestController")
void shouldBypassViewResolution_WhenResponseBody() throws Exception {
    HttpResponse<String> response = httpClient.send(
            HttpRequest.newBuilder(URI.create(baseUrl + "/api/status")).GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).contains("\"status\"");
    assertThat(response.headers().firstValue("Content-Type").orElse("")).contains("application/json");
}
```

**Run:** `./gradlew test` — expected: all 387 tests pass (including all prior features' tests)

---

## 13.9 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why separate "view name" from "view rendering"?** The ViewResolver pattern lets you swap template engines (Thymeleaf, FreeMarker, JSP) without changing controllers. Controllers return logical names like "hello", and the ViewResolver maps them to technology-specific templates. This is the **Strategy pattern** applied to rendering — the same pattern Spring uses for MessageConverters, HandlerAdapters, and ArgumentResolvers.
> - **When NOT to use views:** REST APIs should use `@ResponseBody`/`@RestController` exclusively. Views are for server-rendered HTML. Mixing both in one controller is possible but confusing — the real framework's `@RestController` exists specifically to eliminate this ambiguity.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why does returning null from handle() mean "response handled"?** This convention avoids a mutable container object. In the real framework, `ModelAndViewContainer.requestHandled` serves the same purpose — `@ResponseBody` handlers set it to `true`, and the adapter checks it before building a `ModelAndView`. Our `null` return is a simpler expression of the same idea.
> - **The text/plain fallback is a simplification.** The real `DispatcherServlet.render()` throws `ServletException("Could not resolve view with name '...'")` when no ViewResolver matches. Our fallback preserves backward compatibility for controllers that return Strings without templates — they still get text/plain responses. This is a trade-off: easier migration vs. strict correctness.
> -----------------------------------------------------------

## 13.10 What We Enhanced

| Aspect | Before (ch06) | Current (ch13) | Real Framework |
|--------|--------------|----------------|----------------|
| `HandlerAdapter.handle()` return | `void` — always writes response directly | `ModelAndView` — null for direct response, non-null for view rendering | `ModelAndView` returned from `HandlerAdapter.handle()` (`HandlerAdapter.java:76`) |
| String return handling | `StringReturnValueHandler` writes text/plain | `ViewNameReturnValueHandler` creates `ModelAndView(viewName)` for the dispatcher to resolve | `ViewNameMethodReturnValueHandler` sets view name on `ModelAndViewContainer` |
| Return value handler interface | Returns `void` | Returns `ModelAndView` (null if response handled) | Modifies `ModelAndViewContainer` (mutable output parameter) |
| HTML rendering | Not supported | `TemplateView` with `${placeholder}` substitution | Template engine integration (Thymeleaf, FreeMarker) via `View` implementations |

## 13.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ModelAndView` | `ModelAndView` | `ModelAndView.java:48` | Real supports direct View objects, HttpStatusCode, clear/wasCleared |
| `View.render()` | `View.render()` | `View.java:95` | Same contract — real adds `getContentType()` and request attribute constants |
| `ViewResolver.resolveViewName()` | `ViewResolver.resolveViewName()` | `ViewResolver.java:55` | Real takes Locale parameter for i18n (ch20) |
| `TemplateView` | `InternalResourceView` | `InternalResourceView.java:47` | Real forwards to JSP via `RequestDispatcher`; we do placeholder substitution |
| `TemplateViewResolver` | `InternalResourceViewResolver` | `InternalResourceViewResolver.java:50` | Real extends `UrlBasedViewResolver` with caching, JSTL detection, always-include option |
| `ViewNameReturnValueHandler` | `ViewNameMethodReturnValueHandler` | `ViewNameMethodReturnValueHandler.java:43` | Real handles void returns, "redirect:" prefix, sets on `ModelAndViewContainer` |
| `ModelAndViewReturnValueHandler` | `ModelAndViewMethodReturnValueHandler` | `ModelAndViewMethodReturnValueHandler.java:42` | Real handles `ModelAndViewContainer` interaction, "redirect:" prefix |
| `render()` | `DispatcherServlet.render()` | `DispatcherServlet.java:1267` | Real resolves locale, sets response status, throws on unresolved views |

## 13.12 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/view/ModelAndView.java` [NEW]

```java
package com.simplespringmvc.view;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Holder for both Model and View in the web MVC framework — the return currency
 * of {@code HandlerAdapter.handle()}.
 *
 * Maps to: {@code org.springframework.web.servlet.ModelAndView}
 *
 * The real ModelAndView is more flexible:
 * <ul>
 *   <li>Holds either a view name (String) or a direct View object</li>
 *   <li>Uses Spring's ModelMap (extends LinkedHashMap) for the model</li>
 *   <li>Supports HttpStatusCode for setting response status</li>
 *   <li>Has a clear()/wasCleared() mechanism for interceptors to suppress rendering</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Only supports view name (String), not direct View objects</li>
 *   <li>Uses plain LinkedHashMap for the model</li>
 *   <li>No HTTP status support</li>
 *   <li>No clear/wasCleared mechanism</li>
 * </ul>
 */
public class ModelAndView {

    private String viewName;
    private final Map<String, Object> model = new LinkedHashMap<>();

    /**
     * Create an empty ModelAndView (for bean-style population).
     */
    public ModelAndView() {
    }

    /**
     * Create a ModelAndView with a view name and no model data.
     *
     * Maps to: {@code ModelAndView(String viewName)} constructor
     *
     * @param viewName the logical view name to resolve via ViewResolver
     */
    public ModelAndView(String viewName) {
        this.viewName = viewName;
    }

    /**
     * Create a ModelAndView with a view name and model data.
     *
     * Maps to: {@code ModelAndView(String viewName, Map model)} constructor
     *
     * @param viewName the logical view name
     * @param model    model attributes to pass to the view
     */
    public ModelAndView(String viewName, Map<String, ?> model) {
        this.viewName = viewName;
        if (model != null) {
            this.model.putAll(model);
        }
    }

    /**
     * Set the view name.
     */
    public void setViewName(String viewName) {
        this.viewName = viewName;
    }

    /**
     * Return the view name, or null if not set.
     */
    public String getViewName() {
        return viewName;
    }

    /**
     * Return the model map. Never null — returns an empty map if no data was added.
     *
     * Maps to: {@code ModelAndView.getModel()} which returns the underlying ModelMap
     */
    public Map<String, Object> getModel() {
        return model;
    }

    /**
     * Add a model attribute. Returns {@code this} for fluent chaining.
     *
     * Maps to: {@code ModelAndView.addObject(String, Object)}
     *
     * @param name  attribute name
     * @param value attribute value
     * @return this ModelAndView instance
     */
    public ModelAndView addObject(String name, Object value) {
        model.put(name, value);
        return this;
    }

    @Override
    public String toString() {
        return "ModelAndView [view=\"" + viewName + "\"; model=" + model + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/view/View.java` [NEW]

```java
package com.simplespringmvc.view;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.Map;

/**
 * MVC View for a web interaction. Implementations render content using
 * the model data, request, and response.
 *
 * Maps to: {@code org.springframework.web.servlet.View}
 *
 * Our simplified TemplateView does simple ${placeholder} replacement.
 */
public interface View {

    default String getContentType() {
        return null;
    }

    void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
            throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/view/ViewResolver.java` [NEW]

```java
package com.simplespringmvc.view;

/**
 * Strategy interface for resolving logical view names into View objects.
 *
 * Maps to: {@code org.springframework.web.servlet.ViewResolver}
 *
 * Simplifications:
 * <ul>
 *   <li>No Locale parameter — added in ch20</li>
 *   <li>No view caching</li>
 * </ul>
 */
public interface ViewResolver {

    View resolveViewName(String viewName) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/view/TemplateView.java` [NEW]

```java
package com.simplespringmvc.view;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * A simple View implementation that loads an HTML template from the classpath
 * and replaces {@code ${key}} placeholders with model values.
 *
 * Maps to: {@code org.springframework.web.servlet.view.InternalResourceView}
 * (conceptually — InternalResourceView forwards to a JSP via RequestDispatcher,
 * while we do direct template rendering with placeholder substitution)
 */
public class TemplateView implements View {

    /** Pattern matching ${key} placeholders. */
    private static final Pattern PLACEHOLDER_PATTERN = Pattern.compile("\\$\\{([^}]+)}");

    private final String templatePath;

    public TemplateView(String templatePath) {
        this.templatePath = templatePath;
    }

    @Override
    public String getContentType() {
        return "text/html";
    }

    @Override
    public void render(Map<String, ?> model, HttpServletRequest request,
                       HttpServletResponse response) throws Exception {
        String template = loadTemplate();
        String rendered = replacePlaceholders(template, model);

        response.setContentType("text/html");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(rendered);
        response.getWriter().flush();
    }

    private String loadTemplate() throws Exception {
        InputStream inputStream = getClass().getResourceAsStream(templatePath);
        if (inputStream == null) {
            throw new IllegalStateException(
                    "Template not found on classpath: " + templatePath);
        }
        try (inputStream) {
            return new String(inputStream.readAllBytes(), StandardCharsets.UTF_8);
        }
    }

    private String replacePlaceholders(String template, Map<String, ?> model) {
        if (model == null || model.isEmpty()) {
            return template;
        }

        Matcher matcher = PLACEHOLDER_PATTERN.matcher(template);
        StringBuilder result = new StringBuilder();

        while (matcher.find()) {
            String key = matcher.group(1);
            Object value = model.get(key);
            // Must quote both cases — $ and \ have special meaning in replacement strings.
            String replacement = (value != null) ? value.toString() : matcher.group(0);
            matcher.appendReplacement(result, Matcher.quoteReplacement(replacement));
        }
        matcher.appendTail(result);

        return result.toString();
    }

    public String getTemplatePath() {
        return templatePath;
    }

    @Override
    public String toString() {
        return "TemplateView [" + templatePath + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/view/TemplateViewResolver.java` [NEW]

```java
package com.simplespringmvc.view;

import java.io.InputStream;

/**
 * A ViewResolver that resolves view names to {@link TemplateView} instances
 * by applying a configurable prefix and suffix to the view name.
 *
 * Maps to: {@code org.springframework.web.servlet.view.InternalResourceViewResolver}
 */
public class TemplateViewResolver implements ViewResolver {

    private String prefix = "";
    private String suffix = "";

    public TemplateViewResolver() {
    }

    public TemplateViewResolver(String prefix, String suffix) {
        this.prefix = prefix;
        this.suffix = suffix;
    }

    @Override
    public View resolveViewName(String viewName) {
        String path = prefix + viewName + suffix;

        InputStream resource = getClass().getResourceAsStream(path);
        if (resource != null) {
            try {
                resource.close();
            } catch (Exception ignored) {
            }
            return new TemplateView(path);
        }
        return null;
    }

    public void setPrefix(String prefix) { this.prefix = prefix; }
    public String getPrefix() { return prefix; }
    public void setSuffix(String suffix) { this.suffix = suffix; }
    public String getSuffix() { return suffix; }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerAdapter.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface that allows the DispatcherServlet to invoke handlers
 * without knowing their specific type or calling convention.
 *
 * Maps to: {@code org.springframework.web.servlet.HandlerAdapter}
 *
 * ch13 Enhancement: handle() now returns ModelAndView instead of void.
 */
public interface HandlerAdapter {

    boolean supports(Object handler);

    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandler.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for processing the return value of a handler method.
 * Returns a ModelAndView if the return value represents a view to render,
 * or null if the response was handled directly.
 *
 * ch13 Enhancement: Changed return type from void to ModelAndView.
 */
public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(HandlerMethod handlerMethod);

    ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                   HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ViewNameReturnValueHandler.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Handles String return values by interpreting them as logical view names.
 *
 * Maps to: {@code ViewNameMethodReturnValueHandler}
 */
public class ViewNameReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        Class<?> returnType = handlerMethod.getMethod().getReturnType();
        return CharSequence.class.isAssignableFrom(returnType);
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue instanceof CharSequence viewName) {
            return new ModelAndView(viewName.toString());
        }
        return null;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ModelAndViewReturnValueHandler.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Handles return values of type ModelAndView by returning them directly.
 *
 * Maps to: {@code ModelAndViewMethodReturnValueHandler}
 */
public class ModelAndViewReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return ModelAndView.class.isAssignableFrom(handlerMethod.getMethod().getReturnType());
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue instanceof ModelAndView mv) {
            return mv;
        }
        return null;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

/**
 * Handles return values from handler methods annotated with @ResponseBody
 * by serializing the return value through an HttpMessageConverter.
 * Returns null — the response has been written directly.
 *
 * ch13 Enhancement: Now returns null (ModelAndView) instead of void.
 */
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
        return MergedAnnotationUtils.hasAnnotation(
                handlerMethod.getBeanType(), ResponseBody.class);
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue == null) {
            response.setStatus(HttpServletResponse.SC_OK);
            return null;
        }

        Class<?> valueType = returnValue.getClass();
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(valueType)) {
                converter.write(returnValue, response);
                return null;
            }
        }

        throw new IllegalStateException(
                "No suitable HttpMessageConverter found for return value of type ["
                        + valueType.getName() + "]");
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/StringReturnValueHandler.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * The default catch-all return value handler — writes any return value
 * directly as text/plain. Returns null (response handled directly).
 *
 * ch13 Enhancement: Role narrows — now only catches unusual return types
 * (void, Object, null). ViewNameReturnValueHandler handles String returns.
 * Still used by ExceptionHandlerExceptionResolver as its String handler.
 */
public class StringReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return true;
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        if (returnValue != null) {
            response.getWriter().write(returnValue.toString());
        }
        response.getWriter().flush();
        return null;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

(See the full file in `src/main/java/` — key change: `handle()` returns `ModelAndView`, handler chain includes `ModelAndViewReturnValueHandler` and `ViewNameReturnValueHandler`)

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

(See the full file in `src/main/java/` — key changes: `ViewResolver` list, `render()` method, `resolveViewName()` method, `addViewResolver()` registration)

### Test Code

#### File: `src/test/java/com/simplespringmvc/view/ModelAndViewTest.java` [NEW]

(See the full file in `src/test/java/`)

#### File: `src/test/java/com/simplespringmvc/view/TemplateViewTest.java` [NEW]

(See the full file in `src/test/java/`)

#### File: `src/test/java/com/simplespringmvc/view/TemplateViewResolverTest.java` [NEW]

(See the full file in `src/test/java/`)

#### File: `src/test/java/com/simplespringmvc/adapter/ViewNameReturnValueHandlerTest.java` [NEW]

(See the full file in `src/test/java/`)

#### File: `src/test/java/com/simplespringmvc/adapter/ModelAndViewReturnValueHandlerTest.java` [NEW]

(See the full file in `src/test/java/`)

#### File: `src/test/java/com/simplespringmvc/adapter/SimpleHandlerAdapterTest.java` [MODIFIED]

(See the full file in `src/test/java/` — key change: assertions now check returned `ModelAndView` instead of `responseBody`)

#### File: `src/test/java/com/simplespringmvc/adapter/HandlerMethodReturnValueHandlerCompositeTest.java` [MODIFIED]

(See the full file in `src/test/java/` — key change: `handleReturnValue()` now returns `ModelAndView`)

#### File: `src/test/java/com/simplespringmvc/integration/ViewResolutionIntegrationTest.java` [NEW]

(See the full file in `src/test/java/`)

#### File: `src/test/resources/templates/hello.html` [NEW]

```html
<html>
<body>
<h1>Hello, ${name}!</h1>
<p>Welcome to Simple Spring MVC.</p>
</body>
</html>
```

#### File: `src/test/resources/templates/greeting.html` [NEW]

```html
<html>
<body>
<h1>${title}</h1>
<p>Hello, ${name}! The time is ${time}.</p>
</body>
</html>
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ModelAndView** | Carrier object holding a logical view name + model data map, returned by `handle()` to tell the dispatcher what to render |
| **View** | Interface with `render(model, request, response)` — implementations write HTML to the response |
| **ViewResolver** | Strategy that maps logical view names (like "hello") to View objects (like `TemplateView("/templates/hello.html")`) |
| **TemplateView** | A View that loads HTML from the classpath and replaces `${key}` placeholders with model values |
| **null ModelAndView** | Convention meaning "response already handled" — used by @ResponseBody handlers that write JSON directly |
| **ViewResolver chaining** | DispatcherServlet iterates resolvers in order; first non-null View wins — allows mixing resolver strategies |

**Next: Chapter 14 — Content Negotiation** — Select the right `HttpMessageConverter` based on the `Accept` header and `produces`/`consumes` attributes, so the same endpoint can serve JSON, XML, or other formats based on what the client requests.
