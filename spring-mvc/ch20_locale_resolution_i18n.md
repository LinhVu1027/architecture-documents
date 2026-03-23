# Chapter 20: Locale Resolution & i18n

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The framework dispatches requests, resolves views, and renders templates -- but everything uses the JVM's default locale | A French user visiting the site sees English text, English templates, and English messages -- there is no way to detect, change, or use the user's preferred language | Build a locale resolution subsystem that detects the user's locale from the HTTP request (Accept-Language header, session, or cookie), makes it available throughout the request via a thread-local, allows locale switching via a URL parameter, and supports locale-specific templates and externalized message bundles |

---

## 20.1 The Integration Point -- `service()` in SimpleDispatcherServlet

The locale lifecycle wraps around `doDispatch()` inside the `service()` method. This is where the framework resolves the locale, sets it in a thread-local, and cleans it up when the request completes.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
    try {
        // ch20: Locale lifecycle -- expose resolver and set thread-local locale
        if (localeResolver != null) {
            request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, localeResolver);
            Locale locale = localeResolver.resolveLocale(request);
            LocaleContextHolder.setLocale(locale);
        }

        doDispatch(request, response);
    } catch (Exception ex) {
        throw new ServletException("Dispatch failed", ex);
    } finally {
        // ch20: Reset locale context to prevent thread-local leaks.
        // Maps to: FrameworkServlet.resetContextHolders() (line 1084)
        LocaleContextHolder.resetLocale();
    }
}
```

This maps to `FrameworkServlet.processRequest()` (line 982) which builds a `LocaleContext`, sets it in `LocaleContextHolder`, delegates to `doService()`/`doDispatch()`, and resets the context holders in a finally block.

Three things happen in this method:

1. **Expose the locale resolver as a request attribute.** The `LocaleChangeInterceptor` needs to find the resolver later (to call `setLocale()` when `?lang=fr` is present). Storing it as a request attribute makes it available without coupling the interceptor to the servlet. Maps to `DispatcherServlet.doService()` (line 848).

2. **Resolve the locale and set it in `LocaleContextHolder`.** The resolved locale is stored in a thread-local so any code in the request processing pipeline can access it without passing it through method parameters. Maps to `FrameworkServlet.processRequest()` (line 988-989).

3. **Reset the thread-local in the finally block.** Servlet containers reuse threads from a thread pool. Without cleanup, a French locale from one request would leak into the next request on the same thread. Maps to `FrameworkServlet.resetContextHolders()` (line 1012).

The `render()` method also changed to pass the locale to view resolvers:

```java
private void render(ModelAndView mv, HttpServletRequest request,
                    HttpServletResponse response) throws Exception {
    String viewName = mv.getViewName();

    // ch20: Step 1 -- Determine locale from LocaleContextHolder
    Locale locale = LocaleContextHolder.getLocale();
    response.setLocale(locale);

    // Step 2: Resolve view name via ViewResolver chain (now with locale)
    View view = resolveViewName(viewName, locale);
    // ...
}
```

This maps to `DispatcherServlet.render()` (line 1267) which resolves the locale from the `LocaleResolver` (line 1270), sets it on the response, and passes it to `resolveViewName()` (line 1277).

And the `resolveViewName()` method now passes the locale to each `ViewResolver`:

```java
private View resolveViewName(String viewName, Locale locale) throws Exception {
    for (ViewResolver resolver : viewResolvers) {
        View view = resolver.resolveViewName(viewName, locale);
        if (view != null) {
            return view;
        }
    }
    return null;
}
```

Maps to `DispatcherServlet.resolveViewName()` (line 1339) which iterates all `ViewResolver` beans and returns the first non-null `View`.

The dispatcher also gained a `localeResolver` field (defaulting to `AcceptHeaderLocaleResolver`) and getter/setter methods:

```java
public static final String LOCALE_RESOLVER_ATTRIBUTE =
        SimpleDispatcherServlet.class.getName() + ".LOCALE_RESOLVER";

private LocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
```

Maps to `DispatcherServlet.LOCALE_RESOLVER_ATTRIBUTE` (line 214) and `DispatcherServlet.localeResolver` (line 283). The real framework initializes the resolver from a bean named `"localeResolver"` in the `ApplicationContext` via `initLocaleResolver()` (line 481).

---

## 20.2 The LocaleResolver Strategy

The `LocaleResolver` interface defines the strategy for resolving and changing the locale.

**Creating:** `src/main/java/com/simplespringmvc/locale/LocaleResolver.java`

```java
public interface LocaleResolver {

    Locale resolveLocale(HttpServletRequest request);

    void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale);
}
```

Maps to `org.springframework.web.servlet.LocaleResolver` (line 53). The real interface is identical in shape. Since Spring 4.0 there is a sub-interface `LocaleContextResolver` that adds time zone support via `LocaleContext` -- we simplify to just `Locale`.

Two methods define the contract:

- **`resolveLocale()`** -- Reads the current locale from the request. Must never return null. Called by `SimpleDispatcherServlet.service()` at request start.
- **`setLocale()`** -- Changes the locale. Read-only resolvers (like `AcceptHeaderLocaleResolver`) throw `UnsupportedOperationException`. Mutable resolvers store the new locale in the session or a cookie.

### AcceptHeaderLocaleResolver

The default resolver -- reads the `Accept-Language` header from the browser. Read-only because you cannot change the browser's request headers from the server side.

**Creating:** `src/main/java/com/simplespringmvc/locale/AcceptHeaderLocaleResolver.java`

```java
public class AcceptHeaderLocaleResolver implements LocaleResolver {

    private Locale defaultLocale;
    private List<Locale> supportedLocales = new ArrayList<>();

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Locale requestLocale = request.getLocale();

        // If no supported locales configured, use the request locale directly
        if (supportedLocales.isEmpty()) {
            return (requestLocale != null) ? requestLocale : getEffectiveDefault();
        }

        // Exact match -- country + language
        for (Locale supported : supportedLocales) {
            if (supported.equals(requestLocale)) {
                return supported;
            }
        }

        // Language-only fallback -- match language code ignoring country
        if (requestLocale != null) {
            for (Locale supported : supportedLocales) {
                if (supported.getLanguage().equals(requestLocale.getLanguage())) {
                    return supported;
                }
            }
        }

        // No match found -- use default locale or request locale
        return (defaultLocale != null) ? defaultLocale : requestLocale;
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        throw new UnsupportedOperationException(
                "Cannot change HTTP Accept-Language header -- use SessionLocaleResolver "
                        + "or CookieLocaleResolver for mutable locale resolution");
    }

    // ... getters/setters
}
```

Maps to `org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver` (line 48). The resolution algorithm uses a three-tier fallback: exact match, language-only match, then default. The real implementation also supports a `supportedLocales` list with the same best-match logic.

### SessionLocaleResolver

Stores the locale in the HTTP session. Appropriate when the application already uses sessions.

**Creating:** `src/main/java/com/simplespringmvc/locale/SessionLocaleResolver.java`

```java
public class SessionLocaleResolver implements LocaleResolver {

    public static final String LOCALE_SESSION_ATTRIBUTE_NAME =
            SessionLocaleResolver.class.getName() + ".LOCALE";

    private Locale defaultLocale;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Locale sessionLocale = (Locale) session.getAttribute(LOCALE_SESSION_ATTRIBUTE_NAME);
            if (sessionLocale != null) {
                return sessionLocale;
            }
        }
        return (defaultLocale != null) ? defaultLocale : request.getLocale();
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        if (locale != null) {
            request.getSession().setAttribute(LOCALE_SESSION_ATTRIBUTE_NAME, locale);
        } else {
            HttpSession session = request.getSession(false);
            if (session != null) {
                session.removeAttribute(LOCALE_SESSION_ATTRIBUTE_NAME);
            }
        }
    }

    // ... getters/setters
}
```

Maps to `org.springframework.web.servlet.i18n.SessionLocaleResolver` (line 64). Note the careful session handling: `request.getSession(false)` avoids creating a session just to check for a locale, but `request.getSession()` (which creates one) is used in `setLocale()` because the caller explicitly wants to store a locale.

### CookieLocaleResolver

Stores the locale in a client-side cookie. Useful for stateless applications that do not use sessions.

**Creating:** `src/main/java/com/simplespringmvc/locale/CookieLocaleResolver.java`

```java
public class CookieLocaleResolver implements LocaleResolver {

    public static final String DEFAULT_COOKIE_NAME =
            CookieLocaleResolver.class.getName() + ".LOCALE";

    private String cookieName = DEFAULT_COOKIE_NAME;
    private int cookieMaxAge = -1;  // session cookie
    private String cookiePath = "/";
    private Locale defaultLocale;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (cookieName.equals(cookie.getName())) {
                    String value = cookie.getValue();
                    if (value != null && !value.isEmpty()) {
                        return Locale.forLanguageTag(value);
                    }
                }
            }
        }
        return (defaultLocale != null) ? defaultLocale : request.getLocale();
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        Cookie cookie = new Cookie(cookieName,
                (locale != null) ? locale.toLanguageTag() : "");
        cookie.setPath(cookiePath);
        cookie.setMaxAge((locale != null) ? cookieMaxAge : 0);
        response.addCookie(cookie);
    }

    // ... getters/setters
}
```

Maps to `org.springframework.web.servlet.i18n.CookieLocaleResolver` (line 62). The real implementation uses `ResponseCookie` (for SameSite, Secure, HttpOnly attributes) and stores time zone alongside locale. We simplify to the basic `Cookie` API. The cookie value uses BCP 47 language tags (e.g., `fr-CA`) -- consistent with the real framework since 5.1.

Key design choice: when deleting the locale (locale is null), `maxAge=0` tells the browser to delete the cookie.

---

## 20.3 LocaleContextHolder -- Thread-Local Holder

The `LocaleContextHolder` makes the current locale available anywhere in the application without passing it through method parameters.

**Creating:** `src/main/java/com/simplespringmvc/locale/LocaleContextHolder.java`

```java
public class LocaleContextHolder {

    private static final ThreadLocal<Locale> localeHolder = new ThreadLocal<>();
    private static volatile Locale defaultLocale;

    public static void setLocale(Locale locale) {
        localeHolder.set(locale);
    }

    public static Locale getLocale() {
        Locale locale = localeHolder.get();
        if (locale != null) {
            return locale;
        }
        if (defaultLocale != null) {
            return defaultLocale;
        }
        return Locale.getDefault();
    }

    public static void resetLocale() {
        localeHolder.remove();
    }

    public static void setDefaultLocale(Locale locale) {
        defaultLocale = locale;
    }

    public static Locale getDefaultLocale() {
        return defaultLocale;
    }

    private LocaleContextHolder() {
        // static utility class
    }
}
```

Maps to `org.springframework.context.i18n.LocaleContextHolder` (line 46). The real version stores a full `LocaleContext` (locale + time zone), supports inheritable thread-locals (for child threads), and has a framework-level default. We simplify to just a `Locale`.

The three-tier fallback chain in `getLocale()` ensures a non-null return: thread-local locale, framework default, JVM default. The `defaultLocale` field is `volatile` because it can be set from any thread (e.g., at application startup).

The lifecycle is:

```
Request arrives  -->  DispatcherServlet resolves locale  -->  setLocale()
Handler executes -->  getLocale() available anywhere on this thread
Request ends     -->  resetLocale() clears the thread-local
```

---

## 20.4 LocaleChangeInterceptor

An interceptor that changes the locale when a `?lang=xx` parameter is present in the request URL.

**Creating:** `src/main/java/com/simplespringmvc/locale/LocaleChangeInterceptor.java`

```java
public class LocaleChangeInterceptor implements HandlerInterceptor {

    public static final String DEFAULT_PARAM_NAME = "lang";
    private String paramName = DEFAULT_PARAM_NAME;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             HandlerMethod handler) throws Exception {
        String newLocale = request.getParameter(paramName);
        if (newLocale != null && !newLocale.isEmpty()) {
            LocaleResolver localeResolver = (LocaleResolver)
                    request.getAttribute(SimpleDispatcherServlet.LOCALE_RESOLVER_ATTRIBUTE);

            if (localeResolver != null) {
                Locale locale = parseLocale(newLocale);
                localeResolver.setLocale(request, response, locale);
                LocaleContextHolder.setLocale(locale);
            }
        }
        return true; // always continue the chain
    }

    private Locale parseLocale(String localeString) {
        return Locale.forLanguageTag(localeString.replace('_', '-'));
    }

    // ... getters/setters
}
```

Maps to `org.springframework.web.servlet.i18n.LocaleChangeInterceptor` (line 43). The real implementation uses `"locale"` as the default parameter name -- we use `"lang"` which is more common in modern applications.

The interceptor acts as the bridge between three components:

1. **URL parameter** (`?lang=fr`) -- the user's intent to change language
2. **LocaleResolver** -- persists the new locale (session, cookie, etc.)
3. **LocaleContextHolder** -- updates the thread-local so the rest of the request sees the new locale

The resolver is found via the request attribute set by `SimpleDispatcherServlet.service()` at request start. The interceptor then calls both `localeResolver.setLocale()` (for persistence) and `LocaleContextHolder.setLocale()` (for the current request). This two-step update ensures the locale change takes effect immediately and persists for future requests.

---

## 20.5 MessageSource & i18n

The `MessageSource` interface externalizes messages into locale-specific property files.

**Creating:** `src/main/java/com/simplespringmvc/i18n/MessageSource.java`

```java
public interface MessageSource {

    String getMessage(String code, Object[] args, String defaultMessage, Locale locale);

    String getMessage(String code, Object[] args, Locale locale) throws NoSuchMessageException;
}
```

Maps to `org.springframework.context.MessageSource` (line 40). The real interface has a third method accepting `MessageSourceResolvable` -- we simplify to two methods. Note that the `Locale` parameter is explicit -- `MessageSource` does not look up locale from a thread-local. The caller is responsible for obtaining the locale (typically from `LocaleContextHolder`).

**Creating:** `src/main/java/com/simplespringmvc/i18n/NoSuchMessageException.java`

```java
public class NoSuchMessageException extends RuntimeException {

    public NoSuchMessageException(String code, Locale locale) {
        super("No message found under code '" + code + "' for locale '" + locale + "'");
    }
}
```

### ResourceBundleMessageSource

The implementation backed by JDK `ResourceBundle`:

**Creating:** `src/main/java/com/simplespringmvc/i18n/ResourceBundleMessageSource.java`

```java
public class ResourceBundleMessageSource implements MessageSource {

    private String[] basenames = new String[0];
    private final Map<String, ResourceBundle> bundleCache = new ConcurrentHashMap<>();

    @Override
    public String getMessage(String code, Object[] args, String defaultMessage, Locale locale) {
        String message = resolveMessage(code, locale);
        if (message == null) {
            return formatMessage(defaultMessage, args, locale);
        }
        return formatMessage(message, args, locale);
    }

    @Override
    public String getMessage(String code, Object[] args, Locale locale)
            throws NoSuchMessageException {
        String message = resolveMessage(code, locale);
        if (message == null) {
            throw new NoSuchMessageException(code, locale);
        }
        return formatMessage(message, args, locale);
    }

    private String resolveMessage(String code, Locale locale) {
        for (String basename : basenames) {
            ResourceBundle bundle = getBundle(basename, locale);
            if (bundle != null) {
                try {
                    return bundle.getString(code);
                } catch (MissingResourceException ignored) {
                }
            }
        }
        return null;
    }

    private ResourceBundle getBundle(String basename, Locale locale) {
        String cacheKey = basename + "_" + locale.toLanguageTag();
        return bundleCache.computeIfAbsent(cacheKey, key -> {
            try {
                return ResourceBundle.getBundle(basename, locale);
            } catch (MissingResourceException e) {
                return null;
            }
        });
    }

    private String formatMessage(String message, Object[] args, Locale locale) {
        if (message == null) {
            return null;
        }
        if (args == null || args.length == 0) {
            return message;
        }
        return new MessageFormat(message, locale).format(args);
    }

    public void setBasenames(String... basenames) {
        this.basenames = (basenames != null) ? basenames : new String[0];
        bundleCache.clear();
    }

    // ...
}
```

Maps to `org.springframework.context.support.ResourceBundleMessageSource` (line 81). The JDK `ResourceBundle` provides the actual locale fallback -- when requesting `messages_fr`, if the file does not exist, the JDK automatically falls back to `messages.properties`.

The resolution algorithm iterates basenames in order and returns the first match. The cache key is `basename + "_" + languageTag`, so `"messages"` + `Locale.FRENCH` becomes `"messages_fr"`. The `ConcurrentHashMap.computeIfAbsent()` provides thread-safe lazy loading.

Message formatting uses `java.text.MessageFormat` for placeholders like `{0}`:

```properties
# messages.properties
greeting.with.name=Hello, {0}!

# messages_fr.properties
greeting.with.name=Bonjour, {0} !
```

```java
messageSource.getMessage("greeting.with.name", new Object[]{"Alice"}, null, Locale.FRENCH)
// Returns: "Bonjour, Alice !"
```

---

## 20.6 View Enhancement -- Locale-Aware Templates

### ViewResolver Interface Change

The `ViewResolver` interface now accepts a `Locale` parameter:

**Modifying:** `src/main/java/com/simplespringmvc/view/ViewResolver.java`

```java
public interface ViewResolver {

    View resolveViewName(String viewName, Locale locale) throws Exception;
}
```

Maps to `org.springframework.web.servlet.ViewResolver.resolveViewName(String, Locale)`. The real interface has always had the `Locale` parameter. We added it in this chapter because locale resolution was not available before.

### TemplateViewResolver -- Locale-Specific Templates

The `TemplateViewResolver` now tries a locale-specific template first, then falls back to the default:

**Modifying:** `src/main/java/com/simplespringmvc/view/TemplateViewResolver.java`

```java
@Override
public View resolveViewName(String viewName, Locale locale) {
    // ch20: Try locale-specific template first (e.g., hello_fr.html)
    if (locale != null) {
        String localizedPath = prefix + viewName + "_" + locale.getLanguage() + suffix;
        if (resourceExists(localizedPath)) {
            return createView(localizedPath);
        }
    }

    // Fall back to default template (e.g., hello.html)
    String path = prefix + viewName + suffix;
    if (resourceExists(path)) {
        return createView(path);
    }

    return null;
}
```

Resolution order for `viewName="hello"` with `locale=Locale.FRENCH`:

1. Try `/templates/hello_fr.html` -- locale-specific template
2. Fall back to `/templates/hello.html` -- default template

The resolver also gained `MessageSource` support. When set, every `TemplateView` it creates receives the `MessageSource` for `#{msg.code}` placeholder resolution:

```java
private View createView(String path) {
    TemplateView view = new TemplateView(path);
    if (messageSource != null) {
        view.setMessageSource(messageSource);
    }
    return view;
}
```

### TemplateView -- Message Placeholders

The `TemplateView` now supports two placeholder syntaxes:

**Modifying:** `src/main/java/com/simplespringmvc/view/TemplateView.java`

```java
/** Pattern matching ${key} placeholders (model values). */
private static final Pattern PLACEHOLDER_PATTERN = Pattern.compile("\\$\\{([^}]+)}");

/** Pattern matching #{msg.code} placeholders (i18n messages). */
private static final Pattern MESSAGE_PATTERN = Pattern.compile("#\\{([^}]+)}");

@Override
public void render(Map<String, ?> model, HttpServletRequest request,
                   HttpServletResponse response) throws Exception {
    String template = loadTemplate();
    String rendered = replacePlaceholders(template, model);
    rendered = replaceMessagePlaceholders(rendered);  // ch20
    // ... write to response
}

private String replaceMessagePlaceholders(String template) {
    if (messageSource == null) {
        return template;
    }

    Locale locale = LocaleContextHolder.getLocale();
    Matcher matcher = MESSAGE_PATTERN.matcher(template);
    StringBuilder result = new StringBuilder();

    while (matcher.find()) {
        String code = matcher.group(1);
        String message = messageSource.getMessage(code, null, matcher.group(0), locale);
        matcher.appendReplacement(result, Matcher.quoteReplacement(message));
    }
    matcher.appendTail(result);

    return result.toString();
}
```

The template `i18n.html` uses `#{msg.code}` syntax:

```html
<html>
<body>
<h1>#{greeting.hello}</h1>
<p>#{greeting.welcome}</p>
<nav>#{nav.home} | #{nav.about}</nav>
</body>
</html>
```

With `Locale.FRENCH`, this renders:

```html
<h1>Bonjour</h1>
<p>Bienvenue sur notre site</p>
<nav>Accueil | A propos</nav>
```

The default message in `getMessage()` is set to the placeholder itself (`matcher.group(0)`), so unresolved codes are left as-is rather than causing errors.

---

## 20.7 ServletRequestArgumentResolver -- Injecting Locale

The `ServletRequestArgumentResolver` resolves Servlet API types and `Locale` as handler method parameters:

**Creating:** `src/main/java/com/simplespringmvc/adapter/ServletRequestArgumentResolver.java`

```java
public class ServletRequestArgumentResolver implements HandlerMethodArgumentResolver {

    public static final String RESPONSE_ATTRIBUTE =
            ServletRequestArgumentResolver.class.getName() + ".RESPONSE";

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        Class<?> type = parameter.getParameterType();
        return HttpServletRequest.class.isAssignableFrom(type)
                || HttpServletResponse.class.isAssignableFrom(type)
                || HttpSession.class.isAssignableFrom(type)
                || Locale.class == type;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) {
        Class<?> type = parameter.getParameterType();

        if (HttpServletRequest.class.isAssignableFrom(type)) {
            return request;
        }
        if (HttpServletResponse.class.isAssignableFrom(type)) {
            return request.getAttribute(RESPONSE_ATTRIBUTE);
        }
        if (HttpSession.class.isAssignableFrom(type)) {
            return request.getSession();
        }
        if (Locale.class == type) {
            return LocaleContextHolder.getLocale();
        }

        return null;
    }
}
```

Maps to `org.springframework.web.servlet.mvc.method.annotation.ServletRequestMethodArgumentResolver` (line 70). The real implementation supports many more types (`TimeZone`, `ZoneId`, `Principal`, `InputStream`, `OutputStream`, etc.).

The `HttpServletResponse` requires a workaround: the `HandlerMethodArgumentResolver` interface only receives the `HttpServletRequest`. The `SimpleHandlerAdapter` stores the response as a request attribute before argument resolution:

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`

```java
// ch20: Store response as request attribute for ServletRequestArgumentResolver
request.setAttribute(ServletRequestArgumentResolver.RESPONSE_ATTRIBUTE, response);
```

The `ServletRequestArgumentResolver` is registered first in the argument resolver chain (before annotation-based resolvers) because it matches by type, not annotation:

```java
// ch20: Servlet API + Locale resolver -- handles HttpServletRequest, HttpServletResponse,
// HttpSession, and Locale as method parameters. Comes FIRST because it matches by type,
// not annotation, and should be checked before annotation-based resolvers.
argumentResolvers.addResolver(new ServletRequestArgumentResolver());
```

Usage in a controller:

```java
@GetMapping("/locale-info")
@ResponseBody
public Map<String, String> localeInfo(Locale locale) {
    return Map.of("locale", locale.toLanguageTag());
}
```

---

## 20.8 Try It Yourself

<details>
<summary>Challenge 1: Add a FixedLocaleResolver that always returns a configured locale</summary>

Create a `FixedLocaleResolver` that ignores the request and always returns the same configured locale. Both `resolveLocale()` and `setLocale()` should be read-only (throw `UnsupportedOperationException` from `setLocale()`).

```java
public class FixedLocaleResolver implements LocaleResolver {

    private final Locale locale;

    public FixedLocaleResolver(Locale locale) {
        this.locale = locale;
    }

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        return locale;
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        throw new UnsupportedOperationException("FixedLocaleResolver does not support setLocale");
    }
}
```

This is useful for applications that serve a single language -- no locale detection or switching needed.
</details>

<details>
<summary>Challenge 2: Add time zone support to LocaleContextHolder</summary>

Extend `LocaleContextHolder` to also store a `TimeZone` alongside the locale. The real framework does this via a `TimeZoneAwareLocaleContext`.

```java
private static final ThreadLocal<TimeZone> timeZoneHolder = new ThreadLocal<>();

public static void setTimeZone(TimeZone timeZone) {
    timeZoneHolder.set(timeZone);
}

public static TimeZone getTimeZone() {
    TimeZone tz = timeZoneHolder.get();
    return (tz != null) ? tz : TimeZone.getDefault();
}

public static void resetLocale() {
    localeHolder.remove();
    timeZoneHolder.remove(); // clean up both
}
```

The `SessionLocaleResolver` and `CookieLocaleResolver` in the real framework store both locale and time zone.
</details>

<details>
<summary>Challenge 3: Add parent MessageSource chaining</summary>

The real `MessageSource` supports a parent source -- when a code is not found in the current source, it delegates to the parent. Implement this by adding a `parentMessageSource` field:

```java
private MessageSource parentMessageSource;

private String resolveMessage(String code, Locale locale) {
    // Try this source first
    for (String basename : basenames) {
        ResourceBundle bundle = getBundle(basename, locale);
        if (bundle != null) {
            try {
                return bundle.getString(code);
            } catch (MissingResourceException ignored) {}
        }
    }
    // Fall back to parent
    if (parentMessageSource != null) {
        return parentMessageSource.getMessage(code, null, null, locale);
    }
    return null;
}
```

This enables modular message sources -- each module provides its own messages, with a shared parent for common messages.
</details>

---

## 20.9 Tests

### LocaleContextHolderTest

Validates thread-local locale storage, isolation between threads, and the three-tier fallback chain:

```java
package com.simplespringmvc.locale;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

class LocaleContextHolderTest {

    @AfterEach
    void cleanup() {
        LocaleContextHolder.resetLocale();
        LocaleContextHolder.setDefaultLocale(null);
    }

    @Nested
    @DisplayName("Thread-local locale")
    class ThreadLocalLocale {

        @Test
        @DisplayName("should return set locale from same thread")
        void shouldReturnSetLocale_WhenOnSameThread() {
            LocaleContextHolder.setLocale(Locale.FRENCH);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.FRENCH);
        }

        @Test
        @DisplayName("should return JVM default when no locale set")
        void shouldReturnJvmDefault_WhenNoLocaleSet() {
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.getDefault());
        }

        @Test
        @DisplayName("should clear locale on reset")
        void shouldClearLocale_WhenReset() {
            LocaleContextHolder.setLocale(Locale.GERMAN);
            LocaleContextHolder.resetLocale();
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.getDefault());
        }

        @Test
        @DisplayName("should isolate locales between threads")
        void shouldIsolateLocales_WhenDifferentThreads() throws Exception {
            LocaleContextHolder.setLocale(Locale.JAPANESE);

            Thread otherThread = new Thread(() -> {
                assertThat(LocaleContextHolder.getLocale()).isNotEqualTo(Locale.JAPANESE);
            });
            otherThread.start();
            otherThread.join();
        }
    }

    @Nested
    @DisplayName("Default locale")
    class DefaultLocale {

        @Test
        @DisplayName("should return framework default when no thread-local set")
        void shouldReturnFrameworkDefault_WhenNoThreadLocal() {
            LocaleContextHolder.setDefaultLocale(Locale.ITALIAN);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.ITALIAN);
        }

        @Test
        @DisplayName("should prefer thread-local over framework default")
        void shouldPreferThreadLocal_OverFrameworkDefault() {
            LocaleContextHolder.setDefaultLocale(Locale.ITALIAN);
            LocaleContextHolder.setLocale(Locale.KOREAN);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.KOREAN);
        }

        @Test
        @DisplayName("should fall back to JVM default when framework default cleared")
        void shouldFallBackToJvm_WhenFrameworkDefaultCleared() {
            LocaleContextHolder.setDefaultLocale(Locale.ITALIAN);
            LocaleContextHolder.setDefaultLocale(null);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.getDefault());
        }
    }
}
```

### AcceptHeaderLocaleResolverTest

Validates resolution from `Accept-Language`, supported locales matching, and read-only enforcement:

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class AcceptHeaderLocaleResolverTest {

    private AcceptHeaderLocaleResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        resolver = new AcceptHeaderLocaleResolver();
        request = mock(HttpServletRequest.class);
    }

    @Nested
    @DisplayName("resolveLocale()")
    class ResolveLocale {

        @Test
        @DisplayName("should return request locale from Accept-Language header")
        void shouldReturnRequestLocale_WhenNoSupportedLocalesConfigured() {
            when(request.getLocale()).thenReturn(Locale.FRENCH);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.FRENCH);
        }

        @Test
        @DisplayName("should return exact match from supported locales")
        void shouldReturnExactMatch_WhenInSupportedLocales() {
            resolver.setSupportedLocales(List.of(Locale.ENGLISH, Locale.FRENCH, Locale.GERMAN));
            when(request.getLocale()).thenReturn(Locale.FRENCH);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.FRENCH);
        }

        @Test
        @DisplayName("should fall back to language match when no exact match")
        void shouldFallBackToLanguageMatch_WhenNoExactMatch() {
            resolver.setSupportedLocales(List.of(Locale.ENGLISH, Locale.FRENCH));
            when(request.getLocale()).thenReturn(Locale.CANADA_FRENCH); // fr_CA
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.FRENCH);
        }

        @Test
        @DisplayName("should return default locale when no match found")
        void shouldReturnDefaultLocale_WhenNoMatchFound() {
            resolver.setSupportedLocales(List.of(Locale.ENGLISH, Locale.FRENCH));
            resolver.setDefaultLocale(Locale.ENGLISH);
            when(request.getLocale()).thenReturn(Locale.JAPANESE);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.ENGLISH);
        }

        @Test
        @DisplayName("should return request locale when no match and no default")
        void shouldReturnRequestLocale_WhenNoMatchAndNoDefault() {
            resolver.setSupportedLocales(List.of(Locale.ENGLISH, Locale.FRENCH));
            when(request.getLocale()).thenReturn(Locale.JAPANESE);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.JAPANESE);
        }
    }

    @Nested
    @DisplayName("setLocale()")
    class SetLocale {

        @Test
        @DisplayName("should throw UnsupportedOperationException")
        void shouldThrowUnsupportedOperation() {
            assertThatThrownBy(() -> resolver.setLocale(request, null, Locale.FRENCH))
                    .isInstanceOf(UnsupportedOperationException.class)
                    .hasMessageContaining("Accept-Language");
        }
    }
}
```

### SessionLocaleResolverTest

Validates session-based locale storage, retrieval, and null-safe session handling:

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class SessionLocaleResolverTest {

    private SessionLocaleResolver resolver;
    private HttpServletRequest request;
    private HttpServletResponse response;
    private HttpSession session;

    @BeforeEach
    void setUp() {
        resolver = new SessionLocaleResolver();
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        session = mock(HttpSession.class);
    }

    @Nested
    @DisplayName("resolveLocale()")
    class ResolveLocale {

        @Test
        @DisplayName("should return locale from session when present")
        void shouldReturnSessionLocale_WhenPresent() {
            when(request.getSession(false)).thenReturn(session);
            when(session.getAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME))
                    .thenReturn(Locale.GERMAN);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.GERMAN);
        }

        @Test
        @DisplayName("should return default locale when no session locale")
        void shouldReturnDefaultLocale_WhenNoSessionLocale() {
            resolver.setDefaultLocale(Locale.ITALIAN);
            when(request.getSession(false)).thenReturn(session);
            when(session.getAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME))
                    .thenReturn(null);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.ITALIAN);
        }

        @Test
        @DisplayName("should return request locale when no session and no default")
        void shouldReturnRequestLocale_WhenNoSessionAndNoDefault() {
            when(request.getSession(false)).thenReturn(null);
            when(request.getLocale()).thenReturn(Locale.JAPANESE);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.JAPANESE);
        }
    }

    @Nested
    @DisplayName("setLocale()")
    class SetLocale {

        @Test
        @DisplayName("should store locale in session")
        void shouldStoreLocaleInSession() {
            when(request.getSession()).thenReturn(session);
            resolver.setLocale(request, response, Locale.FRENCH);
            verify(session).setAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME,
                    Locale.FRENCH);
        }

        @Test
        @DisplayName("should remove session attribute when locale is null")
        void shouldRemoveAttribute_WhenLocaleIsNull() {
            when(request.getSession(false)).thenReturn(session);
            resolver.setLocale(request, response, null);
            verify(session).removeAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME);
        }

        @Test
        @DisplayName("should not create session when removing null locale with no session")
        void shouldNotCreateSession_WhenRemovingNullLocaleWithNoSession() {
            when(request.getSession(false)).thenReturn(null);
            resolver.setLocale(request, response, null);
            verify(request, never()).getSession();
        }
    }
}
```

### CookieLocaleResolverTest

Validates cookie-based locale storage with BCP 47 tags, cookie deletion, and custom cookie settings:

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class CookieLocaleResolverTest {

    private CookieLocaleResolver resolver;
    private HttpServletRequest request;
    private HttpServletResponse response;

    @BeforeEach
    void setUp() {
        resolver = new CookieLocaleResolver();
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
    }

    @Nested
    @DisplayName("resolveLocale()")
    class ResolveLocale {

        @Test
        @DisplayName("should return locale from cookie when present")
        void shouldReturnCookieLocale_WhenCookiePresent() {
            Cookie localeCookie = new Cookie(CookieLocaleResolver.DEFAULT_COOKIE_NAME, "fr");
            when(request.getCookies()).thenReturn(new Cookie[]{localeCookie});
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.FRENCH);
        }

        @Test
        @DisplayName("should parse full BCP 47 language tag from cookie")
        void shouldParseBcp47Tag_WhenFullTag() {
            Cookie localeCookie = new Cookie(CookieLocaleResolver.DEFAULT_COOKIE_NAME, "fr-CA");
            when(request.getCookies()).thenReturn(new Cookie[]{localeCookie});
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.CANADA_FRENCH);
        }

        @Test
        @DisplayName("should return default locale when no cookie")
        void shouldReturnDefaultLocale_WhenNoCookie() {
            resolver.setDefaultLocale(Locale.GERMAN);
            when(request.getCookies()).thenReturn(null);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.GERMAN);
        }

        @Test
        @DisplayName("should return request locale when no cookie and no default")
        void shouldReturnRequestLocale_WhenNoCookieAndNoDefault() {
            when(request.getCookies()).thenReturn(null);
            when(request.getLocale()).thenReturn(Locale.KOREAN);
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.KOREAN);
        }

        @Test
        @DisplayName("should use custom cookie name")
        void shouldUseCustomCookieName() {
            resolver.setCookieName("MY_LOCALE");
            Cookie localeCookie = new Cookie("MY_LOCALE", "de");
            when(request.getCookies()).thenReturn(new Cookie[]{localeCookie});
            Locale locale = resolver.resolveLocale(request);
            assertThat(locale).isEqualTo(Locale.GERMAN);
        }
    }

    @Nested
    @DisplayName("setLocale()")
    class SetLocale {

        @Test
        @DisplayName("should add cookie with BCP 47 language tag")
        void shouldAddCookie_WithBcp47Tag() {
            resolver.setLocale(request, response, Locale.FRENCH);

            ArgumentCaptor<Cookie> cookieCaptor = ArgumentCaptor.forClass(Cookie.class);
            verify(response).addCookie(cookieCaptor.capture());

            Cookie cookie = cookieCaptor.getValue();
            assertThat(cookie.getName()).isEqualTo(CookieLocaleResolver.DEFAULT_COOKIE_NAME);
            assertThat(cookie.getValue()).isEqualTo("fr");
            assertThat(cookie.getPath()).isEqualTo("/");
        }

        @Test
        @DisplayName("should delete cookie when locale is null")
        void shouldDeleteCookie_WhenLocaleIsNull() {
            resolver.setLocale(request, response, null);

            ArgumentCaptor<Cookie> cookieCaptor = ArgumentCaptor.forClass(Cookie.class);
            verify(response).addCookie(cookieCaptor.capture());

            Cookie cookie = cookieCaptor.getValue();
            assertThat(cookie.getMaxAge()).isEqualTo(0);
        }

        @Test
        @DisplayName("should respect custom cookie max age")
        void shouldRespectCustomMaxAge() {
            resolver.setCookieMaxAge(3600);
            resolver.setLocale(request, response, Locale.GERMAN);

            ArgumentCaptor<Cookie> cookieCaptor = ArgumentCaptor.forClass(Cookie.class);
            verify(response).addCookie(cookieCaptor.capture());

            assertThat(cookieCaptor.getValue().getMaxAge()).isEqualTo(3600);
        }
    }
}
```

### LocaleChangeInterceptorTest

Validates locale change from URL parameters in multiple formats:

```java
package com.simplespringmvc.locale;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class LocaleChangeInterceptorTest {

    private LocaleChangeInterceptor interceptor;
    private HttpServletRequest request;
    private HttpServletResponse response;
    private HandlerMethod handler;
    private SessionLocaleResolver localeResolver;

    @BeforeEach
    void setUp() throws Exception {
        interceptor = new LocaleChangeInterceptor();
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        handler = new HandlerMethod(this, getClass().getMethod("dummyHandler"));

        localeResolver = new SessionLocaleResolver();
        when(request.getAttribute(SimpleDispatcherServlet.LOCALE_RESOLVER_ATTRIBUTE))
                .thenReturn(localeResolver);

        HttpSession session = mock(HttpSession.class);
        when(request.getSession()).thenReturn(session);
        when(request.getSession(false)).thenReturn(session);
    }

    @AfterEach
    void cleanup() {
        LocaleContextHolder.resetLocale();
    }

    public String dummyHandler() { return "ok"; }

    @Nested
    @DisplayName("preHandle()")
    class PreHandle {

        @Test
        @DisplayName("should change locale when lang parameter present")
        void shouldChangeLocale_WhenLangParameterPresent() throws Exception {
            when(request.getParameter("lang")).thenReturn("fr");
            boolean result = interceptor.preHandle(request, response, handler);
            assertThat(result).isTrue();
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.FRENCH);
        }

        @Test
        @DisplayName("should support BCP 47 language tags with country")
        void shouldSupportBcp47Tags_WithCountry() throws Exception {
            when(request.getParameter("lang")).thenReturn("fr-CA");
            interceptor.preHandle(request, response, handler);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.CANADA_FRENCH);
        }

        @Test
        @DisplayName("should support underscore format")
        void shouldSupportUnderscoreFormat() throws Exception {
            when(request.getParameter("lang")).thenReturn("fr_CA");
            interceptor.preHandle(request, response, handler);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.CANADA_FRENCH);
        }

        @Test
        @DisplayName("should not change locale when no parameter present")
        void shouldNotChangeLocale_WhenNoParameter() throws Exception {
            LocaleContextHolder.setLocale(Locale.ENGLISH);
            when(request.getParameter("lang")).thenReturn(null);
            interceptor.preHandle(request, response, handler);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.ENGLISH);
        }

        @Test
        @DisplayName("should use custom parameter name")
        void shouldUseCustomParamName() throws Exception {
            interceptor.setParamName("locale");
            when(request.getParameter("locale")).thenReturn("de");
            interceptor.preHandle(request, response, handler);
            assertThat(LocaleContextHolder.getLocale()).isEqualTo(Locale.GERMAN);
        }

        @Test
        @DisplayName("should always return true to continue chain")
        void shouldAlwaysReturnTrue() throws Exception {
            when(request.getParameter("lang")).thenReturn(null);
            assertThat(interceptor.preHandle(request, response, handler)).isTrue();
        }
    }
}
```

### ResourceBundleMessageSourceTest

Validates message resolution across locales, formatting with arguments, and error handling:

```java
package com.simplespringmvc.i18n;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ResourceBundleMessageSourceTest {

    private ResourceBundleMessageSource messageSource;

    @BeforeEach
    void setUp() {
        messageSource = new ResourceBundleMessageSource();
        messageSource.setBasenames("messages");
    }

    @Nested
    @DisplayName("getMessage() with default message")
    class GetMessageWithDefault {

        @Test
        @DisplayName("should return English message for English locale")
        void shouldReturnEnglishMessage_WhenEnglishLocale() {
            String message = messageSource.getMessage("greeting.hello", null, null, Locale.ENGLISH);
            assertThat(message).isEqualTo("Hello");
        }

        @Test
        @DisplayName("should return French message for French locale")
        void shouldReturnFrenchMessage_WhenFrenchLocale() {
            String message = messageSource.getMessage("greeting.hello", null, null, Locale.FRENCH);
            assertThat(message).isEqualTo("Bonjour");
        }

        @Test
        @DisplayName("should return German message for German locale")
        void shouldReturnGermanMessage_WhenGermanLocale() {
            String message = messageSource.getMessage("greeting.hello", null, null, Locale.GERMAN);
            assertThat(message).isEqualTo("Hallo");
        }

        @Test
        @DisplayName("should return default message when code not found")
        void shouldReturnDefaultMessage_WhenCodeNotFound() {
            String message = messageSource.getMessage("nonexistent", null, "Fallback", Locale.ENGLISH);
            assertThat(message).isEqualTo("Fallback");
        }

        @Test
        @DisplayName("should return null when code not found and no default")
        void shouldReturnNull_WhenCodeNotFoundAndNoDefault() {
            String message = messageSource.getMessage("nonexistent", null, null, Locale.ENGLISH);
            assertThat(message).isNull();
        }
    }

    @Nested
    @DisplayName("getMessage() throwing")
    class GetMessageThrowing {

        @Test
        @DisplayName("should return message when code exists")
        void shouldReturnMessage_WhenCodeExists() {
            String message = messageSource.getMessage("greeting.hello", null, Locale.ENGLISH);
            assertThat(message).isEqualTo("Hello");
        }

        @Test
        @DisplayName("should throw NoSuchMessageException when code not found")
        void shouldThrowException_WhenCodeNotFound() {
            assertThatThrownBy(() ->
                    messageSource.getMessage("nonexistent", null, Locale.ENGLISH))
                    .isInstanceOf(NoSuchMessageException.class)
                    .hasMessageContaining("nonexistent");
        }
    }

    @Nested
    @DisplayName("Message formatting")
    class MessageFormatting {

        @Test
        @DisplayName("should format message with arguments")
        void shouldFormatMessage_WithArguments() {
            String message = messageSource.getMessage(
                    "greeting.with.name", new Object[]{"Alice"}, null, Locale.ENGLISH);
            assertThat(message).isEqualTo("Hello, Alice!");
        }

        @Test
        @DisplayName("should format French message with arguments")
        void shouldFormatFrenchMessage_WithArguments() {
            String message = messageSource.getMessage(
                    "greeting.with.name", new Object[]{"Alice"}, null, Locale.FRENCH);
            assertThat(message).isEqualTo("Bonjour, Alice !");
        }
    }

    @Nested
    @DisplayName("Multiple basenames")
    class MultipleBasenames {

        @Test
        @DisplayName("should search basenames in order and return first match")
        void shouldSearchBasenamesInOrder() {
            messageSource.setBasenames("messages", "other");
            String message = messageSource.getMessage("greeting.hello", null, null, Locale.ENGLISH);
            assertThat(message).isEqualTo("Hello");
        }
    }

    @Nested
    @DisplayName("Locale fallback")
    class LocaleFallback {

        @Test
        @DisplayName("should fall back to base bundle for unsupported locale")
        void shouldFallBackToBaseBundle_WhenLocaleNotSupported() {
            String message = messageSource.getMessage("greeting.hello", null, null, Locale.JAPANESE);
            assertThat(message).isEqualTo("Hello");
        }
    }
}
```

### ServletRequestArgumentResolverTest

Validates injection of `HttpServletRequest`, `HttpServletResponse`, `HttpSession`, and `Locale`:

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.locale.LocaleContextHolder;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class ServletRequestArgumentResolverTest {

    private ServletRequestArgumentResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        resolver = new ServletRequestArgumentResolver();
        request = mock(HttpServletRequest.class);
    }

    @AfterEach
    void cleanup() {
        LocaleContextHolder.resetLocale();
    }

    @SuppressWarnings("unused")
    public void handlerWithRequest(HttpServletRequest request) {}
    @SuppressWarnings("unused")
    public void handlerWithResponse(HttpServletResponse response) {}
    @SuppressWarnings("unused")
    public void handlerWithSession(HttpSession session) {}
    @SuppressWarnings("unused")
    public void handlerWithLocale(Locale locale) {}
    @SuppressWarnings("unused")
    public void handlerWithString(String name) {}

    private MethodParameter parameterFor(String methodName) throws Exception {
        for (Method m : getClass().getMethods()) {
            if (m.getName().equals(methodName)) {
                return new MethodParameter(m, 0);
            }
        }
        throw new IllegalArgumentException("No method: " + methodName);
    }

    @Nested
    @DisplayName("supportsParameter()")
    class SupportsParameter {

        @Test
        @DisplayName("should support HttpServletRequest parameter")
        void shouldSupportRequest() throws Exception {
            assertThat(resolver.supportsParameter(parameterFor("handlerWithRequest"))).isTrue();
        }

        @Test
        @DisplayName("should support HttpServletResponse parameter")
        void shouldSupportResponse() throws Exception {
            assertThat(resolver.supportsParameter(parameterFor("handlerWithResponse"))).isTrue();
        }

        @Test
        @DisplayName("should support HttpSession parameter")
        void shouldSupportSession() throws Exception {
            assertThat(resolver.supportsParameter(parameterFor("handlerWithSession"))).isTrue();
        }

        @Test
        @DisplayName("should support Locale parameter")
        void shouldSupportLocale() throws Exception {
            assertThat(resolver.supportsParameter(parameterFor("handlerWithLocale"))).isTrue();
        }

        @Test
        @DisplayName("should not support String parameter")
        void shouldNotSupportString() throws Exception {
            assertThat(resolver.supportsParameter(parameterFor("handlerWithString"))).isFalse();
        }
    }

    @Nested
    @DisplayName("resolveArgument()")
    class ResolveArgument {

        @Test
        @DisplayName("should return the request itself for HttpServletRequest")
        void shouldReturnRequest_ForRequestParam() throws Exception {
            Object result = resolver.resolveArgument(parameterFor("handlerWithRequest"), request);
            assertThat(result).isSameAs(request);
        }

        @Test
        @DisplayName("should return response from request attribute for HttpServletResponse")
        void shouldReturnResponse_FromRequestAttribute() throws Exception {
            HttpServletResponse response = mock(HttpServletResponse.class);
            when(request.getAttribute(ServletRequestArgumentResolver.RESPONSE_ATTRIBUTE))
                    .thenReturn(response);
            Object result = resolver.resolveArgument(parameterFor("handlerWithResponse"), request);
            assertThat(result).isSameAs(response);
        }

        @Test
        @DisplayName("should return session for HttpSession")
        void shouldReturnSession_ForSessionParam() throws Exception {
            HttpSession session = mock(HttpSession.class);
            when(request.getSession()).thenReturn(session);
            Object result = resolver.resolveArgument(parameterFor("handlerWithSession"), request);
            assertThat(result).isSameAs(session);
        }

        @Test
        @DisplayName("should return locale from LocaleContextHolder")
        void shouldReturnLocale_FromContextHolder() throws Exception {
            LocaleContextHolder.setLocale(Locale.FRENCH);
            Object result = resolver.resolveArgument(parameterFor("handlerWithLocale"), request);
            assertThat(result).isEqualTo(Locale.FRENCH);
        }
    }
}
```

### LocaleIntegrationTest

End-to-end test verifying the full locale pipeline through an embedded Tomcat:

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.GetMapping;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.i18n.MessageSource;
import com.simplespringmvc.i18n.ResourceBundleMessageSource;
import com.simplespringmvc.locale.LocaleChangeInterceptor;
import com.simplespringmvc.locale.LocaleContextHolder;
import com.simplespringmvc.locale.SessionLocaleResolver;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import com.simplespringmvc.view.ModelAndView;
import com.simplespringmvc.view.TemplateViewResolver;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.net.CookieManager;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Locale;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class LocaleIntegrationTest {

    @Controller
    static class LocaleController {

        @GetMapping("/locale-info")
        @ResponseBody
        public Map<String, String> localeInfo(Locale locale) {
            return Map.of("locale", locale.toLanguageTag());
        }

        @GetMapping("/hello")
        public ModelAndView hello() {
            return new ModelAndView("hello")
                    .addObject("name", "World");
        }

        @GetMapping("/i18n")
        public ModelAndView i18nPage() {
            return new ModelAndView("i18n");
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new LocaleController());

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        servlet.setLocaleResolver(new SessionLocaleResolver());
        servlet.addInterceptor(new LocaleChangeInterceptor());

        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasenames("messages");

        TemplateViewResolver viewResolver = new TemplateViewResolver("/templates/", ".html");
        viewResolver.setMessageSource(messageSource);
        servlet.addViewResolver(viewResolver);

        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        baseUrl = "http://localhost:" + tomcat.getPort();
        httpClient = HttpClient.newBuilder()
                .cookieHandler(new CookieManager())
                .build();
    }

    @AfterEach
    void tearDown() throws Exception {
        tomcat.stop();
    }

    @Nested
    @DisplayName("Locale resolution")
    class LocaleResolution {

        @Test
        @DisplayName("should resolve locale from Accept-Language header")
        void shouldResolveLocale_FromAcceptLanguageHeader() throws Exception {
            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/locale-info"))
                            .header("Accept-Language", "de")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("\"locale\"");
            assertThat(response.body()).contains("de");
        }

        @Test
        @DisplayName("should inject Locale into controller method parameter")
        void shouldInjectLocale_IntoMethodParameter() throws Exception {
            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/locale-info"))
                            .header("Accept-Language", "fr")
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("fr");
        }
    }

    @Nested
    @DisplayName("Locale change via ?lang=")
    class LocaleChange {

        @Test
        @DisplayName("should change locale when ?lang= parameter is present")
        void shouldChangeLocale_WhenLangParamPresent() throws Exception {
            httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/locale-info?lang=fr"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/locale-info"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("fr");
        }
    }

    @Nested
    @DisplayName("Locale-specific views")
    class LocaleSpecificViews {

        @Test
        @DisplayName("should render French template when locale is French")
        void shouldRenderFrenchTemplate_WhenLocaleFrench() throws Exception {
            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/hello?lang=fr"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("Bonjour, World !");
        }

        @Test
        @DisplayName("should render default template when no locale-specific template exists")
        void shouldRenderDefaultTemplate_WhenNoLocaleTemplate() throws Exception {
            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/hello?lang=de"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("Hello, World!");
        }
    }

    @Nested
    @DisplayName("MessageSource i18n in templates")
    class MessageSourceI18n {

        @Test
        @DisplayName("should resolve #{msg.code} to English messages")
        void shouldResolveMessages_ToEnglish() throws Exception {
            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/i18n?lang=en"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("Hello");
            assertThat(response.body()).contains("Welcome to our site");
            assertThat(response.body()).contains("Home");
            assertThat(response.body()).contains("About");
        }

        @Test
        @DisplayName("should resolve #{msg.code} to French messages")
        void shouldResolveMessages_ToFrench() throws Exception {
            HttpResponse<String> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create(baseUrl + "/i18n?lang=fr"))
                            .GET().build(),
                    HttpResponse.BodyHandlers.ofString());

            assertThat(response.statusCode()).isEqualTo(200);
            assertThat(response.body()).contains("Bonjour");
            assertThat(response.body()).contains("Bienvenue sur notre site");
            assertThat(response.body()).contains("Accueil");
        }
    }
}
```

---

## 20.10 Why This Works

> **Insight: Strategy Pattern at Three Levels**
>
> The locale subsystem uses the Strategy pattern at three different levels. The `LocaleResolver` interface is a strategy for *where* the locale comes from (header, session, cookie). The `MessageSource` interface is a strategy for *where* messages live (resource bundles, database, etc.). The `ViewResolver` already used Strategy for *how* views are resolved -- and now it incorporates locale as an input to the strategy. Each strategy can be swapped independently without affecting the others. A session-based locale resolver works with any message source, and a resource bundle message source works with any locale resolver.

> **Insight: Thread-Local as Implicit Context**
>
> The `LocaleContextHolder` thread-local pattern solves a fundamental problem: how to make contextual data (the current locale) available to code deep in the call stack without threading it through every method signature. Without the thread-local, `LocaleContextHolder.getLocale()` would need to become a parameter on `render()`, `resolveViewName()`, `TemplateView.render()`, `replaceMessagePlaceholders()`, every argument resolver, and every controller method. The thread-local eliminates this "parameter drilling" -- but with a critical trade-off: the lifecycle must be carefully managed in a try/finally block to prevent leaks across requests on the same thread. The real framework uses this same pattern for `RequestContextHolder`, `SecurityContextHolder`, and `TransactionSynchronizationManager`.

> **Insight: Request Attribute as Service Locator**
>
> The `LOCALE_RESOLVER_ATTRIBUTE` request attribute is a lightweight service locator. The dispatcher stores the locale resolver at request start, and the `LocaleChangeInterceptor` retrieves it later. This avoids coupling the interceptor to the servlet -- the interceptor depends only on the `LocaleResolver` interface and the attribute name. The real framework uses this same pattern for `HANDLER_MAPPING_ATTRIBUTE`, `HANDLER_ADAPTER_ATTRIBUTE`, `LOCALE_RESOLVER_ATTRIBUTE`, and many others. It is essentially dependency injection through the request scope.

---

## 20.11 What We Enhanced

| File | Enhancement | Maps to |
|------|-------------|---------|
| `SimpleDispatcherServlet` | Added locale lifecycle in `service()`: expose resolver as request attribute, resolve locale into `LocaleContextHolder`, reset in finally block. Added `localeResolver` field (defaults to `AcceptHeaderLocaleResolver`), `LOCALE_RESOLVER_ATTRIBUTE` constant, locale-aware `render()` and `resolveViewName()`, setter/getter | `FrameworkServlet.processRequest()` (line 982), `DispatcherServlet.doService()` (line 848), `DispatcherServlet.render()` (line 1267), `DispatcherServlet.resolveViewName()` (line 1339) |
| `ViewResolver` | Added `Locale` parameter to `resolveViewName()` | `ViewResolver.resolveViewName(String, Locale)` |
| `TemplateViewResolver` | Locale-specific template selection (tries `viewName_lang.suffix` first), `MessageSource` support (passes to created `TemplateView` instances) | `UrlBasedViewResolver.createView()` (line 463) |
| `TemplateView` | Added `#{msg.code}` placeholder support via `MessageSource` and `LocaleContextHolder`, `MESSAGE_PATTERN` regex | `InternalResourceView.renderMergedOutputModel()` with JSTL message tags |
| `SimpleHandlerAdapter` | Registered `ServletRequestArgumentResolver` first in resolver chain, stores `HttpServletResponse` as request attribute before argument resolution | `RequestMappingHandlerAdapter.getDefaultArgumentResolvers()` |

---

## 20.12 Connection to Real Framework

| Our File | Real Framework File | Line/Type | Notes |
|----------|-------------------|-----------|-------|
| `LocaleResolver.java` | `spring-webmvc/../LocaleResolver.java` | :53 interface | Same two-method contract. Real version has `LocaleContextResolver` sub-interface for time zone support |
| `AcceptHeaderLocaleResolver.java` | `spring-webmvc/../i18n/AcceptHeaderLocaleResolver.java` | :48 class | We support `supportedLocales` matching. Real version extends `AbstractLocaleResolver` |
| `SessionLocaleResolver.java` | `spring-webmvc/../i18n/SessionLocaleResolver.java` | :64 class | Same session attribute pattern. Real version extends `AbstractLocaleContextResolver`, stores time zone |
| `CookieLocaleResolver.java` | `spring-webmvc/../i18n/CookieLocaleResolver.java` | :62 class | We use basic `Cookie` API. Real version uses `ResponseCookie` with SameSite/Secure/HttpOnly |
| `LocaleChangeInterceptor.java` | `spring-webmvc/../i18n/LocaleChangeInterceptor.java` | :43 class | We default to "lang" param. Real version defaults to "locale", supports HTTP method restrictions |
| `LocaleContextHolder.java` | `spring-context/../i18n/LocaleContextHolder.java` | :46 class | We store Locale directly. Real version stores `LocaleContext` (locale + time zone), supports inheritable thread-locals |
| `MessageSource.java` | `spring-context/../MessageSource.java` | :40 interface | We have two methods. Real version has three (+ `MessageSourceResolvable`) |
| `ResourceBundleMessageSource.java` | `spring-context/../support/ResourceBundleMessageSource.java` | :81 class | We use simple cache. Real version has triple-nested cache, custom `ResourceBundle.Control`, hot-reloading |
| `ServletRequestArgumentResolver.java` | `spring-webmvc/../annotation/ServletRequestMethodArgumentResolver.java` | :70 class | We support 4 types. Real version supports 12+ types (Principal, TimeZone, InputStream, etc.) |
| `SimpleDispatcherServlet.service()` | `FrameworkServlet.processRequest()` | :982 method | Locale lifecycle: resolve, set thread-local, dispatch, reset. Real version also handles `RequestAttributes` |
| `SimpleDispatcherServlet.render()` | `DispatcherServlet.render()` | :1267 method | Locale-aware view resolution. Real version resolves locale via `localeResolver.resolveLocale()` at line 1270 |

> All line references are based on commit `11ab0b4351`.

---

## 20.13 Complete Code

### New Files

#### `src/main/java/com/simplespringmvc/locale/LocaleContextHolder.java`

```java
package com.simplespringmvc.locale;

import java.util.Locale;

/**
 * Thread-local holder that makes the current {@link Locale} available anywhere
 * in the application without passing it through method parameters.
 *
 * Maps to: {@code org.springframework.context.i18n.LocaleContextHolder}
 *
 * The real LocaleContextHolder:
 * <ul>
 *   <li>Stores a full {@code LocaleContext} (locale + optional time zone)</li>
 *   <li>Supports both inheritable and non-inheritable thread-locals</li>
 *   <li>Has framework-level default locale and time zone</li>
 *   <li>Works with {@code SimpleLocaleContext} and {@code TimeZoneAwareLocaleContext}</li>
 * </ul>
 *
 * We simplify to just a {@code Locale} (no time zone, no inheritable mode).
 * The lifecycle is:
 * <pre>
 *   Request arrives -> DispatcherServlet resolves locale -> setLocale()
 *   Handler executes -> getLocale() available anywhere on this thread
 *   Request ends -> resetLocale() clears the thread-local
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Stores Locale directly instead of LocaleContext</li>
 *   <li>No inheritable thread-local support</li>
 *   <li>No time zone support</li>
 * </ul>
 */
public class LocaleContextHolder {

    private static final ThreadLocal<Locale> localeHolder = new ThreadLocal<>();

    private static volatile Locale defaultLocale;

    /**
     * Set the locale for the current thread.
     *
     * Maps to: {@code LocaleContextHolder.setLocaleContext(LocaleContext, boolean)}
     *
     * Called by {@code SimpleDispatcherServlet.service()} at request start,
     * and by {@code LocaleChangeInterceptor} when a locale-change parameter
     * is detected.
     *
     * @param locale the locale to associate with this thread
     */
    public static void setLocale(Locale locale) {
        localeHolder.set(locale);
    }

    /**
     * Return the locale for the current thread, with fallback chain:
     * thread-local -> framework default -> JVM default.
     *
     * Maps to: {@code LocaleContextHolder.getLocale()}
     * Never returns null.
     *
     * @return the current locale
     */
    public static Locale getLocale() {
        Locale locale = localeHolder.get();
        if (locale != null) {
            return locale;
        }
        if (defaultLocale != null) {
            return defaultLocale;
        }
        return Locale.getDefault();
    }

    /**
     * Clear the locale for the current thread.
     *
     * Maps to: {@code LocaleContextHolder.resetLocaleContext()}
     *
     * Called by {@code SimpleDispatcherServlet.service()} in the finally block
     * after request processing completes.
     */
    public static void resetLocale() {
        localeHolder.remove();
    }

    /**
     * Set the framework-level default locale, used when no thread-local is set.
     *
     * Maps to: {@code LocaleContextHolder.setDefaultLocale(Locale)}
     *
     * @param locale the default locale, or null to use JVM default
     */
    public static void setDefaultLocale(Locale locale) {
        defaultLocale = locale;
    }

    /**
     * Return the framework-level default locale, or null if not set.
     */
    public static Locale getDefaultLocale() {
        return defaultLocale;
    }

    private LocaleContextHolder() {
        // static utility class
    }
}
```

#### `src/main/java/com/simplespringmvc/locale/LocaleResolver.java`

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.Locale;

/**
 * Strategy interface for resolving the current locale from an HTTP request,
 * and optionally changing it via the response.
 *
 * Maps to: {@code org.springframework.web.servlet.LocaleResolver}
 *
 * The real interface has an extension {@code LocaleContextResolver} (since 4.0)
 * that adds time zone support via {@code LocaleContext}. We simplify to
 * just {@code Locale} -- no time zone, no LocaleContext wrapper.
 *
 * Three built-in strategies:
 * <ul>
 *   <li>{@link AcceptHeaderLocaleResolver} -- reads Accept-Language (read-only, default)</li>
 *   <li>{@link SessionLocaleResolver} -- stores locale in HTTP session</li>
 *   <li>{@link CookieLocaleResolver} -- stores locale in a client-side cookie</li>
 * </ul>
 *
 * The DispatcherServlet holds one LocaleResolver and delegates all locale
 * decisions to it -- this is the Strategy pattern in action.
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code LocaleContextResolver} extension -- just Locale, no TimeZone</li>
 *   <li>No default strategy loading from properties file</li>
 * </ul>
 */
public interface LocaleResolver {

    /**
     * Resolve the current locale from the given request.
     * Must never return null.
     *
     * Maps to: {@code LocaleResolver.resolveLocale(HttpServletRequest)}
     *
     * @param request the current HTTP request
     * @return the resolved locale (never null)
     */
    Locale resolveLocale(HttpServletRequest request);

    /**
     * Set the current locale to the given one.
     * May throw {@code UnsupportedOperationException} for read-only resolvers
     * (e.g., AcceptHeaderLocaleResolver -- you can't change the browser's headers).
     *
     * Maps to: {@code LocaleResolver.setLocale(HttpServletRequest, HttpServletResponse, Locale)}
     *
     * @param request  the current HTTP request
     * @param response the current HTTP response
     * @param locale   the new locale, or null to reset to default
     */
    void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale);
}
```

#### `src/main/java/com/simplespringmvc/locale/AcceptHeaderLocaleResolver.java`

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Locale;

/**
 * Resolves the locale from the HTTP {@code Accept-Language} header.
 * This is the default LocaleResolver -- read-only, since we can't change
 * the browser's request headers from the server side.
 *
 * Maps to: {@code org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver}
 *
 * The real implementation:
 * <ul>
 *   <li>Extends {@code AbstractLocaleResolver} (provides defaultLocale getter/setter)</li>
 *   <li>Has {@code supportedLocales} list for filtering -- finds best match</li>
 *   <li>Uses quality-factor sorting from the Accept-Language header</li>
 *   <li>Throws {@code UnsupportedOperationException} from {@code setLocale()}</li>
 * </ul>
 *
 * Our implementation supports:
 * <ul>
 *   <li>Reading locale from {@code request.getLocale()} (Servlet container parses Accept-Language)</li>
 *   <li>Configurable supported locales with best-match selection</li>
 *   <li>Configurable default locale</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No quality-factor based matching -- Servlet container handles Accept-Language parsing</li>
 *   <li>Does not iterate all accepted locales -- uses the primary one from request.getLocale()</li>
 * </ul>
 */
public class AcceptHeaderLocaleResolver implements LocaleResolver {

    private Locale defaultLocale;
    private List<Locale> supportedLocales = new ArrayList<>();

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Locale requestLocale = request.getLocale();

        // If no supported locales configured, use the request locale directly
        if (supportedLocales.isEmpty()) {
            return (requestLocale != null) ? requestLocale : getEffectiveDefault();
        }

        // Exact match -- country + language
        for (Locale supported : supportedLocales) {
            if (supported.equals(requestLocale)) {
                return supported;
            }
        }

        // Language-only fallback -- match language code ignoring country
        if (requestLocale != null) {
            for (Locale supported : supportedLocales) {
                if (supported.getLanguage().equals(requestLocale.getLanguage())) {
                    return supported;
                }
            }
        }

        // No match found -- use default locale or request locale
        return (defaultLocale != null) ? defaultLocale : requestLocale;
    }

    /**
     * Cannot change the Accept-Language header from the server side.
     *
     * @throws UnsupportedOperationException always
     */
    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        throw new UnsupportedOperationException(
                "Cannot change HTTP Accept-Language header -- use SessionLocaleResolver "
                        + "or CookieLocaleResolver for mutable locale resolution");
    }

    private Locale getEffectiveDefault() {
        return (defaultLocale != null) ? defaultLocale : Locale.getDefault();
    }

    public void setDefaultLocale(Locale defaultLocale) {
        this.defaultLocale = defaultLocale;
    }

    public Locale getDefaultLocale() {
        return defaultLocale;
    }

    public void setSupportedLocales(List<Locale> supportedLocales) {
        this.supportedLocales = (supportedLocales != null)
                ? new ArrayList<>(supportedLocales) : new ArrayList<>();
    }

    public List<Locale> getSupportedLocales() {
        return Collections.unmodifiableList(supportedLocales);
    }
}
```

#### `src/main/java/com/simplespringmvc/locale/SessionLocaleResolver.java`

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

import java.util.Locale;

/**
 * Stores the locale in the {@link HttpSession}. Appropriate when the application
 * already uses sessions. Settings are temporary -- lost when the session terminates.
 *
 * Maps to: {@code org.springframework.web.servlet.i18n.SessionLocaleResolver}
 *
 * The real implementation:
 * <ul>
 *   <li>Extends {@code AbstractLocaleContextResolver} (locale + time zone)</li>
 *   <li>Stores both locale and time zone as separate session attributes</li>
 *   <li>Has configurable default locale/time zone functions (since 6.0)</li>
 *   <li>Uses {@code WebUtils.getSessionAttribute()} for safe session access</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No time zone support</li>
 *   <li>No configurable default locale function</li>
 *   <li>Direct session access instead of WebUtils</li>
 * </ul>
 */
public class SessionLocaleResolver implements LocaleResolver {

    /**
     * Session attribute name for the stored locale.
     *
     * Maps to: {@code SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME}
     * Uses the class name as prefix to avoid collisions with other frameworks.
     */
    public static final String LOCALE_SESSION_ATTRIBUTE_NAME =
            SessionLocaleResolver.class.getName() + ".LOCALE";

    private Locale defaultLocale;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // Try session first (without creating one if it doesn't exist)
        HttpSession session = request.getSession(false);
        if (session != null) {
            Locale sessionLocale = (Locale) session.getAttribute(LOCALE_SESSION_ATTRIBUTE_NAME);
            if (sessionLocale != null) {
                return sessionLocale;
            }
        }
        // Fall back to configured default, then Accept-Language
        return (defaultLocale != null) ? defaultLocale : request.getLocale();
    }

    /**
     * Store the locale in the HTTP session, or remove it if locale is null.
     *
     * Maps to: {@code SessionLocaleResolver.setLocaleContext()} (line 131)
     * which uses {@code WebUtils.setSessionAttribute()} for safe null handling.
     */
    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        if (locale != null) {
            request.getSession().setAttribute(LOCALE_SESSION_ATTRIBUTE_NAME, locale);
        } else {
            HttpSession session = request.getSession(false);
            if (session != null) {
                session.removeAttribute(LOCALE_SESSION_ATTRIBUTE_NAME);
            }
        }
    }

    public void setDefaultLocale(Locale defaultLocale) {
        this.defaultLocale = defaultLocale;
    }

    public Locale getDefaultLocale() {
        return defaultLocale;
    }
}
```

#### `src/main/java/com/simplespringmvc/locale/CookieLocaleResolver.java`

```java
package com.simplespringmvc.locale;

import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.Locale;

/**
 * Stores the locale in a client-side cookie. Useful for stateless applications
 * that do not use HTTP sessions -- the locale preference persists across
 * sessions because it's stored on the client.
 *
 * Maps to: {@code org.springframework.web.servlet.i18n.CookieLocaleResolver}
 *
 * The real implementation:
 * <ul>
 *   <li>Extends {@code AbstractLocaleContextResolver} (locale + time zone)</li>
 *   <li>Cookie format: {@code <locale>/<timeZoneId>} (or "-" for null locale)</li>
 *   <li>Uses {@code ResponseCookie} for modern cookie attributes (SameSite, etc.)</li>
 *   <li>Supports BCP 47 language tags (default since 5.1)</li>
 *   <li>Caches parsed locale as request attribute to avoid re-parsing</li>
 *   <li>Has configurable default locale/time zone functions</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No time zone support -- stores only locale in cookie</li>
 *   <li>Uses simple {@code Cookie} API instead of {@code ResponseCookie}</li>
 *   <li>No SameSite, Secure, or HttpOnly cookie attributes</li>
 *   <li>No request-attribute caching of parsed locale</li>
 *   <li>Always uses BCP 47 language tags</li>
 * </ul>
 */
public class CookieLocaleResolver implements LocaleResolver {

    /**
     * Default cookie name, using class name as prefix to avoid collisions.
     *
     * Maps to: {@code CookieLocaleResolver.DEFAULT_COOKIE_NAME}
     */
    public static final String DEFAULT_COOKIE_NAME =
            CookieLocaleResolver.class.getName() + ".LOCALE";

    private String cookieName = DEFAULT_COOKIE_NAME;
    private int cookieMaxAge = -1;  // -1 = session cookie (deleted when browser closes)
    private String cookiePath = "/";
    private Locale defaultLocale;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (cookieName.equals(cookie.getName())) {
                    String value = cookie.getValue();
                    if (value != null && !value.isEmpty()) {
                        return Locale.forLanguageTag(value);
                    }
                }
            }
        }
        // No cookie found -- fall back to default, then Accept-Language
        return (defaultLocale != null) ? defaultLocale : request.getLocale();
    }

    /**
     * Store the locale in a cookie, or remove the cookie if locale is null.
     *
     * Maps to: {@code CookieLocaleResolver.setLocaleContext()} (line 211)
     * which builds a {@code ResponseCookie} via the Set-Cookie header.
     * We simplify to the basic {@code Cookie} API.
     */
    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        Cookie cookie = new Cookie(cookieName,
                (locale != null) ? locale.toLanguageTag() : "");
        cookie.setPath(cookiePath);
        // maxAge=0 deletes the cookie; otherwise use the configured maxAge
        cookie.setMaxAge((locale != null) ? cookieMaxAge : 0);
        response.addCookie(cookie);
    }

    public void setCookieName(String cookieName) {
        this.cookieName = cookieName;
    }

    public String getCookieName() {
        return cookieName;
    }

    public void setCookieMaxAge(int cookieMaxAge) {
        this.cookieMaxAge = cookieMaxAge;
    }

    public int getCookieMaxAge() {
        return cookieMaxAge;
    }

    public void setCookiePath(String cookiePath) {
        this.cookiePath = cookiePath;
    }

    public String getCookiePath() {
        return cookiePath;
    }

    public void setDefaultLocale(Locale defaultLocale) {
        this.defaultLocale = defaultLocale;
    }

    public Locale getDefaultLocale() {
        return defaultLocale;
    }
}
```

#### `src/main/java/com/simplespringmvc/locale/LocaleChangeInterceptor.java`

```java
package com.simplespringmvc.locale;

import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.Locale;

/**
 * An interceptor that changes the current locale when a configurable
 * request parameter is present (e.g., {@code ?lang=fr}).
 *
 * Maps to: {@code org.springframework.web.servlet.i18n.LocaleChangeInterceptor}
 *
 * Acts as the bridge between user-initiated locale changes (URL parameter)
 * and the {@link LocaleResolver}. The interceptor reads the parameter value,
 * calls {@code localeResolver.setLocale()}, and updates the thread-local
 * in {@link LocaleContextHolder}.
 *
 * The real implementation:
 * <ul>
 *   <li>Default parameter name: "locale"</li>
 *   <li>Supports restricting to specific HTTP methods</li>
 *   <li>Has {@code ignoreInvalidLocale} flag</li>
 *   <li>Delegates to {@code StringUtils.parseLocale()} for flexible parsing</li>
 *   <li>Gets LocaleResolver from request attribute via {@code RequestContextUtils}</li>
 * </ul>
 *
 * Our implementation uses "lang" as the default parameter name (more common in
 * modern applications) and always allows locale changes on any HTTP method.
 *
 * Simplifications:
 * <ul>
 *   <li>Default param name "lang" instead of "locale"</li>
 *   <li>No HTTP method restrictions</li>
 *   <li>No ignoreInvalidLocale flag</li>
 *   <li>Uses {@code Locale.forLanguageTag()} for BCP 47 parsing</li>
 * </ul>
 */
public class LocaleChangeInterceptor implements HandlerInterceptor {

    /** Default request parameter name: {@code "lang"}. */
    public static final String DEFAULT_PARAM_NAME = "lang";

    private String paramName = DEFAULT_PARAM_NAME;

    /**
     * If the locale parameter is present, change the locale via the
     * LocaleResolver and update the thread-local.
     *
     * Maps to: {@code LocaleChangeInterceptor.preHandle()} (line 118)
     * The real version obtains the LocaleResolver from the request attribute
     * set by DispatcherServlet in doService().
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             HandlerMethod handler) throws Exception {
        String newLocale = request.getParameter(paramName);
        if (newLocale != null && !newLocale.isEmpty()) {
            // Get the locale resolver from the request attribute
            // (set by SimpleDispatcherServlet at request start)
            LocaleResolver localeResolver = (LocaleResolver)
                    request.getAttribute(SimpleDispatcherServlet.LOCALE_RESOLVER_ATTRIBUTE);

            if (localeResolver != null) {
                // Parse the locale string -- support both "fr" and "fr-FR" formats
                Locale locale = parseLocale(newLocale);

                // Store via the resolver (session, cookie, etc.)
                localeResolver.setLocale(request, response, locale);

                // Update the thread-local so the rest of the request sees the new locale
                LocaleContextHolder.setLocale(locale);
            }
        }
        return true; // always continue the chain
    }

    /**
     * Parse a locale string, supporting both BCP 47 tags (fr-FR) and
     * legacy format (fr_FR).
     *
     * Maps to: {@code LocaleChangeInterceptor.parseLocaleValue()} (line 158)
     * which delegates to {@code StringUtils.parseLocale()}.
     */
    private Locale parseLocale(String localeString) {
        // Normalize underscore to hyphen for BCP 47 compatibility
        return Locale.forLanguageTag(localeString.replace('_', '-'));
    }

    public void setParamName(String paramName) {
        this.paramName = paramName;
    }

    public String getParamName() {
        return paramName;
    }
}
```

#### `src/main/java/com/simplespringmvc/i18n/MessageSource.java`

```java
package com.simplespringmvc.i18n;

import java.util.Locale;

/**
 * Strategy interface for resolving internationalized messages.
 * Takes a message code and a Locale and returns the localized string.
 *
 * Maps to: {@code org.springframework.context.MessageSource}
 *
 * The real interface:
 * <ul>
 *   <li>Has three overloaded {@code getMessage()} methods</li>
 *   <li>Supports {@code MessageSourceResolvable} for encapsulating code + args + default</li>
 *   <li>Supports hierarchical MessageSource chaining (parent sources)</li>
 *   <li>The {@code ApplicationContext} itself implements MessageSource</li>
 * </ul>
 *
 * The Locale parameter is always explicit -- MessageSource does NOT look up
 * locale from a thread-local. The caller is responsible for obtaining the
 * locale (typically from {@code LocaleContextHolder}).
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code MessageSourceResolvable} support</li>
 *   <li>No parent MessageSource chaining</li>
 * </ul>
 */
public interface MessageSource {

    /**
     * Resolve a message with a fallback default message.
     * Returns the default message if the code is not found.
     *
     * Maps to: {@code MessageSource.getMessage(String, Object[], String, Locale)}
     *
     * @param code           the message code to look up (e.g., "greeting.hello")
     * @param args           arguments for message placeholders (e.g., {0}, {1}), or null
     * @param defaultMessage fallback message if code not found, or null
     * @param locale         the locale for message resolution
     * @return the resolved message, or defaultMessage if not found
     */
    String getMessage(String code, Object[] args, String defaultMessage, Locale locale);

    /**
     * Resolve a message, throwing if not found.
     *
     * Maps to: {@code MessageSource.getMessage(String, Object[], Locale)}
     *
     * @param code   the message code to look up
     * @param args   arguments for message placeholders, or null
     * @param locale the locale for message resolution
     * @return the resolved message
     * @throws NoSuchMessageException if the message code is not found
     */
    String getMessage(String code, Object[] args, Locale locale) throws NoSuchMessageException;
}
```

#### `src/main/java/com/simplespringmvc/i18n/NoSuchMessageException.java`

```java
package com.simplespringmvc.i18n;

import java.util.Locale;

/**
 * Exception thrown when a message code cannot be resolved for a given locale.
 *
 * Maps to: {@code org.springframework.context.NoSuchMessageException}
 */
public class NoSuchMessageException extends RuntimeException {

    public NoSuchMessageException(String code, Locale locale) {
        super("No message found under code '" + code + "' for locale '" + locale + "'");
    }
}
```

#### `src/main/java/com/simplespringmvc/i18n/ResourceBundleMessageSource.java`

```java
package com.simplespringmvc.i18n;

import java.text.MessageFormat;
import java.util.Locale;
import java.util.MissingResourceException;
import java.util.ResourceBundle;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

/**
 * A {@link MessageSource} implementation backed by JDK {@link ResourceBundle}.
 * Reads {@code .properties} files from the classpath using configured basenames.
 *
 * Maps to: {@code org.springframework.context.support.ResourceBundleMessageSource}
 *
 * Example:
 * <pre>
 *   setBasenames("messages")  ->  loads messages.properties, messages_fr.properties, etc.
 *   getMessage("greeting", null, Locale.FRENCH)  ->  "Bonjour"
 * </pre>
 *
 * The real implementation:
 * <ul>
 *   <li>Extends {@code AbstractResourceBasedMessageSource} -> {@code AbstractMessageSource}</li>
 *   <li>Triple-nested ConcurrentHashMap cache: bundle -> code -> locale -> MessageFormat</li>
 *   <li>Custom {@code ResourceBundle.Control} for charset support and reload policies</li>
 *   <li>Supports multiple basenames (iterated in order, first match wins)</li>
 *   <li>Configurable cacheMillis for hot-reloading</li>
 *   <li>Parent MessageSource chaining</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Simple caching -- bundles are cached once per basename+locale</li>
 *   <li>No custom ResourceBundle.Control -- uses JDK defaults</li>
 *   <li>No cacheMillis / hot-reloading</li>
 *   <li>No parent MessageSource</li>
 *   <li>No default encoding configuration (uses JDK default for properties)</li>
 * </ul>
 */
public class ResourceBundleMessageSource implements MessageSource {

    private String[] basenames = new String[0];

    /**
     * Cache of loaded ResourceBundles: "basename_locale" -> ResourceBundle.
     *
     * Maps to: {@code ResourceBundleMessageSource.cachedResourceBundles}
     * (ConcurrentHashMap<String, Map<Locale, ResourceBundle>>)
     */
    private final Map<String, ResourceBundle> bundleCache = new ConcurrentHashMap<>();

    @Override
    public String getMessage(String code, Object[] args, String defaultMessage, Locale locale) {
        String message = resolveMessage(code, locale);
        if (message == null) {
            return formatMessage(defaultMessage, args, locale);
        }
        return formatMessage(message, args, locale);
    }

    @Override
    public String getMessage(String code, Object[] args, Locale locale) throws NoSuchMessageException {
        String message = resolveMessage(code, locale);
        if (message == null) {
            throw new NoSuchMessageException(code, locale);
        }
        return formatMessage(message, args, locale);
    }

    /**
     * Look up a message code across all configured basenames.
     * Returns the first match, or null if not found in any bundle.
     *
     * Maps to: {@code ResourceBundleMessageSource.resolveCodeWithoutArguments()}
     * which iterates basenames and looks up the code in each ResourceBundle.
     */
    private String resolveMessage(String code, Locale locale) {
        for (String basename : basenames) {
            ResourceBundle bundle = getBundle(basename, locale);
            if (bundle != null) {
                try {
                    return bundle.getString(code);
                } catch (MissingResourceException ignored) {
                    // Code not in this bundle -- try the next basename
                }
            }
        }
        return null;
    }

    /**
     * Load or retrieve a cached ResourceBundle for the given basename and locale.
     *
     * Maps to: {@code ResourceBundleMessageSource.getResourceBundle()}
     * The real version has a two-tier cache with reload support.
     */
    private ResourceBundle getBundle(String basename, Locale locale) {
        String cacheKey = basename + "_" + locale.toLanguageTag();
        return bundleCache.computeIfAbsent(cacheKey, key -> {
            try {
                return ResourceBundle.getBundle(basename, locale);
            } catch (MissingResourceException e) {
                return null;
            }
        });
    }

    /**
     * Format a message with arguments using {@link MessageFormat}.
     * Supports placeholders like {0}, {1}, etc.
     *
     * Maps to: {@code AbstractMessageSource.formatMessage()}
     */
    private String formatMessage(String message, Object[] args, Locale locale) {
        if (message == null) {
            return null;
        }
        if (args == null || args.length == 0) {
            return message;
        }
        return new MessageFormat(message, locale).format(args);
    }

    /**
     * Set the basenames for resource bundle lookup.
     * Each basename is used to load .properties files (e.g., "messages"
     * loads messages.properties, messages_fr.properties, etc.).
     *
     * Maps to: {@code AbstractResourceBasedMessageSource.setBasenames(String...)}
     *
     * @param basenames the basenames to use (classpath resource names without extension)
     */
    public void setBasenames(String... basenames) {
        this.basenames = (basenames != null) ? basenames : new String[0];
        // Clear cache when basenames change
        bundleCache.clear();
    }

    /**
     * Returns the configured basenames.
     */
    public String[] getBasenames() {
        return basenames.clone();
    }
}
```

#### `src/main/java/com/simplespringmvc/adapter/ServletRequestArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.locale.LocaleContextHolder;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

import java.util.Locale;

/**
 * Resolves Servlet API types and {@link Locale} as handler method parameters.
 * Allows controllers to declare these types directly in method signatures.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ServletRequestMethodArgumentResolver}
 *
 * The real implementation supports a wider range of types:
 * <ul>
 *   <li>{@code HttpServletRequest}, {@code HttpServletResponse}</li>
 *   <li>{@code HttpSession}, {@code Principal}, {@code Locale}</li>
 *   <li>{@code TimeZone}, {@code ZoneId}</li>
 *   <li>{@code InputStream}, {@code OutputStream}, {@code Reader}, {@code Writer}</li>
 *   <li>{@code HttpMethod}, {@code PushBuilder}</li>
 * </ul>
 *
 * We support the most common types: request, response, session, and locale.
 *
 * <h3>Usage:</h3>
 * <pre>
 *   {@literal @}GetMapping("/info")
 *   public String info(HttpServletRequest request, Locale locale) {
 *       // request and locale are injected automatically
 *   }
 * </pre>
 *
 * <h3>How HttpServletResponse is available:</h3>
 * The argument resolver interface only receives the request. To make the
 * response available, the {@code SimpleHandlerAdapter} stores it as a
 * request attribute before argument resolution.
 *
 * Simplifications:
 * <ul>
 *   <li>Only HttpServletRequest, HttpServletResponse, HttpSession, and Locale</li>
 *   <li>HttpServletResponse obtained via request attribute</li>
 *   <li>Locale obtained from LocaleContextHolder</li>
 * </ul>
 */
public class ServletRequestArgumentResolver implements HandlerMethodArgumentResolver {

    /**
     * Request attribute key for the HttpServletResponse.
     * Set by SimpleHandlerAdapter.handle() before argument resolution.
     */
    public static final String RESPONSE_ATTRIBUTE =
            ServletRequestArgumentResolver.class.getName() + ".RESPONSE";

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        Class<?> type = parameter.getParameterType();
        return HttpServletRequest.class.isAssignableFrom(type)
                || HttpServletResponse.class.isAssignableFrom(type)
                || HttpSession.class.isAssignableFrom(type)
                || Locale.class == type;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) {
        Class<?> type = parameter.getParameterType();

        if (HttpServletRequest.class.isAssignableFrom(type)) {
            return request;
        }
        if (HttpServletResponse.class.isAssignableFrom(type)) {
            return request.getAttribute(RESPONSE_ATTRIBUTE);
        }
        if (HttpSession.class.isAssignableFrom(type)) {
            return request.getSession();
        }
        if (Locale.class == type) {
            return LocaleContextHolder.getLocale();
        }

        return null;
    }
}
```

### Modified Files

#### `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

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
import com.simplespringmvc.locale.AcceptHeaderLocaleResolver;
import com.simplespringmvc.locale.LocaleContextHolder;
import com.simplespringmvc.locale.LocaleResolver;
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
import java.util.Locale;
import java.util.Map;

/**
 * The front controller that receives ALL HTTP requests and dispatches them
 * to the appropriate handler.
 *
 * Maps to: {@code org.springframework.web.servlet.DispatcherServlet}
 *
 * The real DispatcherServlet sits at the bottom of a three-class hierarchy:
 * <pre>
 *   HttpServletBean          -> maps init-params to bean properties
 *     +-- FrameworkServlet   -> manages WebApplicationContext, request lifecycle
 *           +-- DispatcherServlet -> the actual dispatch logic
 * </pre>
 *
 * We collapse all three layers into one class. The key method is {@link #doDispatch}
 * which is the central routing point -- every future feature adds a step here:
 * <ul>
 *   <li>ch03: handler mapping lookup</li>
 *   <li>ch04: handler adapter invocation</li>
 *   <li>ch11: interceptor pre/post processing</li>
 *   <li>ch12: exception handler resolution</li>
 *   <li>ch13: view resolution and rendering</li>
 *   <li>ch18: flash map lifecycle and redirect handling</li>
 *   <li>ch19: CORS processing in getHandler()</li>
 *   <li>ch20: locale resolution lifecycle</li>
 * </ul>
 *
 * <h3>ch18 Enhancement:</h3>
 * The dispatch pipeline now includes flash map management and redirect support:
 * <pre>
 *   0. Flash map lifecycle: retrieve input flash map, create output flash map
 *   1. getHandler(request) -> HandlerExecutionChain
 *   2. chain.applyPreHandle()
 *   3. ha.handle() -> returns ModelAndView (null if response handled directly)
 *   4. chain.applyPostHandle()
 *   5. chain.triggerAfterCompletion() (always)
 *   6. If exception -> processHandlerException()
 *   7. If ModelAndView non-null:
 *      7a. If view name starts with "redirect:" -> handle redirect + save flash map
 *      7b. Otherwise -> resolve and render view (merge input flash attrs into model)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy -- one class does everything</li>
 *   <li>No WebApplicationContext -- uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No theme resolution</li>
 *   <li>Falls back to text/plain when view can't be resolved (real framework throws)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    /**
     * ch20: Request attribute under which the LocaleResolver is stored.
     * Used by {@code LocaleChangeInterceptor} to find the resolver.
     *
     * Maps to: {@code DispatcherServlet.LOCALE_RESOLVER_ATTRIBUTE} (line 178)
     */
    public static final String LOCALE_RESOLVER_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".LOCALE_RESOLVER";

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
     * (or from DispatcherServlet.properties defaults). Iterated in order --
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

    /**
     * ch20: Locale resolver for determining the current locale.
     * Defaults to {@link AcceptHeaderLocaleResolver}.
     *
     * Maps to: {@code DispatcherServlet.localeResolver} (line 178)
     * The real version is initialized from a "localeResolver" bean in the context,
     * defaulting to AcceptHeaderLocaleResolver.
     */
    private LocaleResolver localeResolver = new AcceptHeaderLocaleResolver();

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
     * Override service() to route ALL HTTP methods through doDispatch(),
     * with locale lifecycle management wrapping the dispatch.
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870) and
     * {@code FrameworkServlet.processRequest()} (line 982)
     *
     * <h3>ch20 Enhancement:</h3>
     * The locale lifecycle mirrors {@code FrameworkServlet.processRequest()}:
     * <pre>
     *   1. Expose locale resolver as request attribute (for interceptors)
     *   2. Resolve locale from request -> set in LocaleContextHolder
     *   3. doDispatch() -- locale available to all code via LocaleContextHolder
     *   4. finally: reset LocaleContextHolder (cleanup thread-local)
     * </pre>
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            // ch20: Locale lifecycle -- expose resolver and set thread-local locale
            if (localeResolver != null) {
                request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, localeResolver);
                Locale locale = localeResolver.resolveLocale(request);
                LocaleContextHolder.setLocale(locale);
            }

            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        } finally {
            // ch20: Reset locale context to prevent thread-local leaks.
            // Maps to: FrameworkServlet.resetContextHolders() (line 1084)
            LocaleContextHolder.resetLocale();
        }
    }

    /**
     * The central dispatch method -- the heart of the framework.
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
     *   3. mv = ha.handle(request, response, handler) -- now manages session attrs
     *   4. mappedHandler.applyPostHandle()
     *   5. mappedHandler.triggerAfterCompletion() (always)
     *   6. processHandlerException() if exception
     *   7. If view name starts with "redirect:" -> handleRedirect() + save flash map
     *      Otherwise -> render view (merge input flash attrs into model)
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

        // ch18: Step 0 -- Flash map lifecycle: retrieve input, create output
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

            // Step 2: Apply interceptor preHandle -- forward order
            if (!mappedHandler.applyPreHandle(request, response)) {
                return;
            }

            // Step 3: Find a HandlerAdapter and invoke the handler.
            // ch18: handle() now manages session attributes lifecycle internally.
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
            mv = adapter.handle(request, response, mappedHandler.getHandler());

            // Step 4: Apply interceptor postHandle -- reverse order
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

        // Step 5: afterCompletion -- always runs
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, dispatchException);
        }

        // Step 6: Exception handling
        if (dispatchException != null) {
            Object handler = (mappedHandler != null) ? mappedHandler.getHandler() : null;
            if (!processHandlerException(request, response, handler, dispatchException)) {
                throw dispatchException;
            }
            // Exception was handled -- don't render a view
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
     *   2. Resolve view name -> View object (via ViewResolver chain)
     *   3. Set response status if configured
     *   4. Call view.render(model, request, response)
     * </pre>
     *
     * <h3>ch20 Enhancement:</h3>
     * Now resolves the locale from the LocaleResolver (step 1), sets it on
     * the response (so response headers reflect the locale), and passes it
     * to the ViewResolver chain for locale-specific template selection.
     *
     * @param mv       the ModelAndView holding view name and model data
     * @param request  current HTTP request
     * @param response current HTTP response
     */
    private void render(ModelAndView mv, HttpServletRequest request,
                        HttpServletResponse response) throws Exception {
        String viewName = mv.getViewName();

        // ch20: Step 1 -- Determine locale from LocaleContextHolder
        // (set at request start in service(), possibly changed by LocaleChangeInterceptor)
        Locale locale = LocaleContextHolder.getLocale();
        response.setLocale(locale);

        // Step 2: Resolve view name via ViewResolver chain (now with locale)
        View view = resolveViewName(viewName, locale);

        if (view != null) {
            // Step 4: Delegate to the View object for rendering.
            view.render(mv.getModel(), request, response);
        } else {
            // Fallback: write the view name directly as text/plain.
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
     * -> iterates {@code viewResolvers} and returns the first non-null result.
     *
     * <h3>ch20 Enhancement:</h3>
     * Now passes the locale to each ViewResolver for locale-specific
     * template selection (e.g., hello_fr.html for French).
     *
     * @param viewName the logical view name to resolve
     * @param locale   the locale for locale-specific view selection
     * @return the View object, or null if no resolver can resolve it
     */
    private View resolveViewName(String viewName, Locale locale) throws Exception {
        for (ViewResolver resolver : viewResolvers) {
            View view = resolver.resolveViewName(viewName, locale);
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
     * -- specifically the CORS block at lines 569-588.
     */
    private HandlerExecutionChain getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        HandlerMethod handler = handlerMapping.lookupHandler(request);

        // ch19: Handle preflight requests that have no explicit OPTIONS handler
        if (handler == null && CorsUtils.isPreFlightRequest(request)) {
            HandlerMethod actualHandler = findPreflightHandler(request);
            if (actualHandler != null) {
                HandlerMethod noOpHandler = createNoOpHandler();
                HandlerExecutionChain chain = new HandlerExecutionChain(noOpHandler, interceptors);
                applyCorsProcessing(chain, actualHandler, request);
                return chain;
            }
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

    private HandlerMethod findPreflightHandler(HttpServletRequest request) {
        String actualMethod = request.getHeader("Access-Control-Request-Method");
        if (actualMethod == null) {
            return null;
        }
        return handlerMapping.lookupHandler(request.getRequestURI(), actualMethod);
    }

    private HandlerMethod createNoOpHandler() {
        try {
            return new HandlerMethod(this, getClass().getMethod("noOpPreflightHandler"));
        } catch (NoSuchMethodException e) {
            throw new IllegalStateException("Cannot create no-op preflight handler", e);
        }
    }

    public String noOpPreflightHandler() {
        return null;
    }

    private void applyCorsProcessing(HandlerExecutionChain chain,
                                      HandlerMethod handler,
                                      HttpServletRequest request) {
        CorsConfiguration handlerConfig = handlerMapping.getCorsConfiguration(handler);
        CorsConfiguration globalConfig = handlerMapping.getGlobalCorsConfiguration(
                request.getRequestURI());

        CorsConfiguration mergedConfig;
        if (globalConfig != null && handlerConfig != null) {
            mergedConfig = globalConfig.combine(handlerConfig);
        } else if (globalConfig != null) {
            mergedConfig = globalConfig;
        } else if (handlerConfig != null) {
            mergedConfig = handlerConfig;
        } else {
            return;
        }

        mergedConfig.validateAllowCredentials();
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

    // --- FlashMapManager registration ---

    public void setFlashMapManager(FlashMapManager flashMapManager) {
        this.flashMapManager = flashMapManager;
    }

    public FlashMapManager getFlashMapManager() {
        return flashMapManager;
    }

    // --- ViewResolver registration ---

    public void addViewResolver(ViewResolver viewResolver) {
        this.viewResolvers.add(viewResolver);
    }

    public List<ViewResolver> getViewResolvers() {
        return List.copyOf(viewResolvers);
    }

    // --- Interceptor registration ---

    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptors.add(interceptor);
    }

    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptors);
    }

    // --- CORS registration (ch19) ---

    public void setCorsRegistry(CorsRegistry corsRegistry) {
        if (handlerMapping != null) {
            handlerMapping.setCorsConfigurations(corsRegistry.getCorsConfigurations());
        }
    }

    public void setCorsProcessor(CorsProcessor corsProcessor) {
        this.corsProcessor = corsProcessor;
    }

    public CorsProcessor getCorsProcessor() {
        return corsProcessor;
    }

    // --- Locale resolver registration (ch20) ---

    public void setLocaleResolver(LocaleResolver localeResolver) {
        this.localeResolver = localeResolver;
    }

    public LocaleResolver getLocaleResolver() {
        return localeResolver;
    }

    // --- Accessors ---

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

#### `src/main/java/com/simplespringmvc/view/ViewResolver.java`

```java
package com.simplespringmvc.view;

import java.util.Locale;

/**
 * Strategy interface for resolving logical view names into View objects.
 *
 * Maps to: {@code org.springframework.web.servlet.ViewResolver}
 *
 * ViewResolvers are chained in the DispatcherServlet: the servlet iterates
 * all registered resolvers and uses the first one that returns a non-null
 * View. This allows mixing resolver strategies (e.g., Thymeleaf for HTML
 * templates + Jackson for JSON views).
 *
 * <h3>ch20 Enhancement:</h3>
 * The Locale parameter was added to support locale-specific view selection.
 * A ViewResolver can use the locale to choose different templates for
 * different languages (e.g., {@code hello_fr.html} for French).
 *
 * Simplifications:
 * <ul>
 *   <li>No view caching -- the real framework caches resolved views</li>
 * </ul>
 */
public interface ViewResolver {

    /**
     * Resolve the given view name into a View object, considering the locale.
     *
     * Maps to: {@code ViewResolver.resolveViewName(String, Locale)}
     *
     * Returns null if this resolver cannot resolve the view name,
     * allowing the DispatcherServlet to try the next resolver in the chain.
     *
     * @param viewName the logical view name (e.g., "hello", "user/profile")
     * @param locale   the locale for locale-specific view selection (may be null)
     * @return the View object, or null if not resolvable by this resolver
     * @throws Exception if the view cannot be resolved
     */
    View resolveViewName(String viewName, Locale locale) throws Exception;
}
```

#### `src/main/java/com/simplespringmvc/view/TemplateViewResolver.java`

```java
package com.simplespringmvc.view;

import com.simplespringmvc.i18n.MessageSource;

import java.io.InputStream;
import java.util.Locale;

/**
 * A ViewResolver that resolves view names to {@link TemplateView} instances
 * by applying a configurable prefix and suffix to the view name.
 *
 * Maps to: {@code org.springframework.web.servlet.view.InternalResourceViewResolver}
 * which extends {@code UrlBasedViewResolver}
 *
 * The real InternalResourceViewResolver:
 * <ul>
 *   <li>Extends UrlBasedViewResolver which provides prefix/suffix and caching</li>
 *   <li>Creates InternalResourceView (or JstlView if JSTL is present)</li>
 *   <li>Uses RequestDispatcher to forward to JSPs</li>
 *   <li>Always returns a view (never null) -- relies on being last in the chain</li>
 * </ul>
 *
 * <h3>ch20 Enhancement:</h3>
 * <ul>
 *   <li>Added Locale parameter for locale-specific template selection.
 *       When resolving "hello" with Locale "fr", tries "hello_fr.html" first,
 *       then falls back to "hello.html".</li>
 *   <li>Added MessageSource support -- if set, passes it to TemplateView instances
 *       so they can resolve {@code #{msg.code}} placeholders.</li>
 * </ul>
 *
 * <h3>Example:</h3>
 * <pre>
 *   prefix = "/templates/"
 *   suffix = ".html"
 *   viewName = "hello", locale = Locale.FRENCH
 *   -> tries "/templates/hello_fr.html" first, then "/templates/hello.html"
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No view caching -- the real framework caches resolved View objects</li>
 *   <li>No redirect: or forward: prefix handling</li>
 *   <li>Locale-specific selection by language only (no country variant templates)</li>
 * </ul>
 */
public class TemplateViewResolver implements ViewResolver {

    private String prefix = "";
    private String suffix = "";
    private MessageSource messageSource;

    public TemplateViewResolver() {
    }

    /**
     * Create a resolver with the given prefix and suffix.
     *
     * Maps to: {@code InternalResourceViewResolver(String prefix, String suffix)}
     *
     * @param prefix prepended to view names (e.g., "/templates/")
     * @param suffix appended to view names (e.g., ".html")
     */
    public TemplateViewResolver(String prefix, String suffix) {
        this.prefix = prefix;
        this.suffix = suffix;
    }

    /**
     * Resolve a view name to a TemplateView, trying locale-specific template
     * first, then falling back to the default template.
     *
     * Maps to: {@code UrlBasedViewResolver.createView()} (line 463)
     * -> {@code buildView()} (line 495) which applies prefix + suffix
     * -> {@code loadView()} which creates the View object
     *
     * <h3>ch20 Enhancement:</h3>
     * Resolution order for viewName="hello", locale=Locale.FRENCH:
     * <ol>
     *   <li>Try locale-specific: /templates/hello_fr.html</li>
     *   <li>Fall back to default: /templates/hello.html</li>
     * </ol>
     *
     * @param viewName the logical view name (e.g., "hello")
     * @param locale   the locale for template selection (may be null)
     * @return a TemplateView if the template exists, null otherwise
     */
    @Override
    public View resolveViewName(String viewName, Locale locale) {
        // ch20: Try locale-specific template first (e.g., hello_fr.html)
        if (locale != null) {
            String localizedPath = prefix + viewName + "_" + locale.getLanguage() + suffix;
            if (resourceExists(localizedPath)) {
                return createView(localizedPath);
            }
        }

        // Fall back to default template (e.g., hello.html)
        String path = prefix + viewName + suffix;
        if (resourceExists(path)) {
            return createView(path);
        }

        return null;
    }

    /**
     * Create a TemplateView for the given classpath path, optionally
     * configuring it with a MessageSource for i18n placeholder support.
     */
    private View createView(String path) {
        TemplateView view = new TemplateView(path);
        if (messageSource != null) {
            view.setMessageSource(messageSource);
        }
        return view;
    }

    /**
     * Check if a classpath resource exists at the given path.
     */
    private boolean resourceExists(String path) {
        InputStream resource = getClass().getResourceAsStream(path);
        if (resource != null) {
            try {
                resource.close();
            } catch (Exception ignored) {
            }
            return true;
        }
        return false;
    }

    /**
     * Set the MessageSource for resolving {@code #{msg.code}} placeholders
     * in templates.
     *
     * <h3>ch20 Enhancement:</h3>
     * When set, every TemplateView created by this resolver will have access
     * to the MessageSource for internationalized message resolution.
     *
     * @param messageSource the message source to use
     */
    public void setMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    public MessageSource getMessageSource() {
        return messageSource;
    }

    public void setPrefix(String prefix) {
        this.prefix = prefix;
    }

    public String getPrefix() {
        return prefix;
    }

    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }

    public String getSuffix() {
        return suffix;
    }
}
```

#### `src/main/java/com/simplespringmvc/view/TemplateView.java`

```java
package com.simplespringmvc.view;

import com.simplespringmvc.i18n.MessageSource;
import com.simplespringmvc.locale.LocaleContextHolder;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Locale;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * A simple View implementation that loads an HTML template from the classpath
 * and replaces placeholders with model values and i18n messages.
 *
 * Maps to: {@code org.springframework.web.servlet.view.InternalResourceView}
 * (conceptually -- InternalResourceView forwards to a JSP via RequestDispatcher,
 * while we do direct template rendering with placeholder substitution)
 *
 * <h3>Placeholder syntax:</h3>
 * <pre>
 *   ${name}          -> replaced with model.get("name").toString()
 *   #{greeting.hello} -> replaced with messageSource.getMessage("greeting.hello", ...)
 * </pre>
 *
 * <h3>ch20 Enhancement:</h3>
 * Added {@code #{msg.code}} placeholder support for internationalized messages.
 * When a MessageSource is configured, placeholders like {@code #{greeting.hello}}
 * are resolved using the current locale from {@code LocaleContextHolder}.
 * Placeholders that don't match any message code are left as-is.
 *
 * Simplifications:
 * <ul>
 *   <li>Simple regex-based substitution instead of a real template engine</li>
 *   <li>No expression language (no ${user.name} dot navigation)</li>
 *   <li>No conditional blocks, loops, or template inheritance</li>
 *   <li>No model attributes exposed as request attributes</li>
 * </ul>
 */
public class TemplateView implements View {

    /** Pattern matching ${key} placeholders (model values). */
    private static final Pattern PLACEHOLDER_PATTERN = Pattern.compile("\\$\\{([^}]+)}");

    /** Pattern matching #{msg.code} placeholders (i18n messages). */
    private static final Pattern MESSAGE_PATTERN = Pattern.compile("#\\{([^}]+)}");

    private final String templatePath;
    private MessageSource messageSource;

    /**
     * Create a TemplateView for the given classpath resource path.
     *
     * @param templatePath the classpath path to the HTML template
     *                     (e.g., "/templates/hello.html")
     */
    public TemplateView(String templatePath) {
        this.templatePath = templatePath;
    }

    @Override
    public String getContentType() {
        return "text/html";
    }

    /**
     * Load the template from the classpath, replace ${key} placeholders
     * with model values and #{msg.code} with i18n messages, and write
     * the result to the response.
     *
     * Maps to: {@code InternalResourceView.renderMergedOutputModel()}
     */
    @Override
    public void render(Map<String, ?> model, HttpServletRequest request,
                       HttpServletResponse response) throws Exception {
        // Step 1: Load the template from the classpath
        String template = loadTemplate();

        // Step 2: Replace ${key} placeholders with model values
        String rendered = replacePlaceholders(template, model);

        // Step 3 (ch20): Replace #{msg.code} placeholders with i18n messages
        rendered = replaceMessagePlaceholders(rendered);

        // Step 4: Write to the response
        response.setContentType("text/html");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(rendered);
        response.getWriter().flush();
    }

    /**
     * Load the template file from the classpath as a String.
     *
     * @return the template content
     * @throws Exception if the template cannot be found or read
     */
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

    /**
     * Replace all ${key} placeholders in the template with corresponding
     * model values. Unmatched placeholders are left as-is.
     */
    private String replacePlaceholders(String template, Map<String, ?> model) {
        if (model == null || model.isEmpty()) {
            return template;
        }

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

    /**
     * Replace all #{msg.code} placeholders with messages from the MessageSource.
     * Uses the current locale from {@link LocaleContextHolder}.
     *
     * <h3>ch20 Enhancement:</h3>
     * If no MessageSource is configured, this method is a no-op.
     * Unresolved message codes are left as-is (e.g., #{unknown.code} stays).
     */
    private String replaceMessagePlaceholders(String template) {
        if (messageSource == null) {
            return template;
        }

        Locale locale = LocaleContextHolder.getLocale();
        Matcher matcher = MESSAGE_PATTERN.matcher(template);
        StringBuilder result = new StringBuilder();

        while (matcher.find()) {
            String code = matcher.group(1);
            // Use the placeholder itself as default so unresolved codes are left as-is
            String message = messageSource.getMessage(code, null, matcher.group(0), locale);
            matcher.appendReplacement(result, Matcher.quoteReplacement(message));
        }
        matcher.appendTail(result);

        return result.toString();
    }

    /**
     * Set the MessageSource for resolving #{msg.code} placeholders.
     *
     * @param messageSource the message source, or null to disable i18n
     */
    public void setMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    /**
     * Returns the configured MessageSource, or null.
     */
    public MessageSource getMessageSource() {
        return messageSource;
    }

    /**
     * Returns the classpath path of this template (for testing/debugging).
     */
    public String getTemplatePath() {
        return templatePath;
    }

    @Override
    public String toString() {
        return "TemplateView [" + templatePath + "]";
    }
}
```

#### `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`

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
import com.simplespringmvc.session.DefaultSessionAttributeStore;
import com.simplespringmvc.session.SessionAttributeStore;
import com.simplespringmvc.session.SessionAttributesHandler;
import com.simplespringmvc.session.SimpleSessionStatus;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * The adapter that bridges the DispatcherServlet's generic handler invocation
 * to our specific HandlerMethod-based invocation pipeline.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 *
 * <h3>ch18 Enhancement:</h3>
 * <ul>
 *   <li>Manages the {@code @SessionAttributes} lifecycle around handler invocation:
 *       retrieves session attributes before invocation, stores matching model
 *       attributes after invocation, and cleans up on SessionStatus.setComplete()</li>
 *   <li>Caches {@link SessionAttributesHandler} per controller class for efficiency</li>
 *   <li>Registers three new argument resolvers: {@link SessionAttributeArgumentResolver},
 *       {@link SessionStatusArgumentResolver}, and {@link RedirectAttributesArgumentResolver}</li>
 *   <li>Sets up per-request {@code SimpleSessionStatus} before argument resolution</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Single adapter instance (not a chain of adapters)</li>
 *   <li>No ModelAndViewContainer -- return value handlers return ModelAndView directly</li>
 *   <li>No ModelFactory -- session attribute lifecycle is managed directly</li>
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

    /**
     * ch18: Cache of SessionAttributesHandler per controller class.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.sessionAttributesHandlerCache} (line 194)
     * The real version uses a ConcurrentHashMap keyed by controller type.
     */
    private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache =
            new ConcurrentHashMap<>(64);

    /**
     * ch18: Strategy for session attribute storage.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.sessionAttributeStore}
     */
    private final SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();

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

        // ch20: Servlet API + Locale resolver -- handles HttpServletRequest, HttpServletResponse,
        // HttpSession, and Locale as method parameters. Comes FIRST because it matches by type,
        // not annotation, and should be checked before annotation-based resolvers.
        argumentResolvers.addResolver(new ServletRequestArgumentResolver());

        // ch18: Session-aware resolvers come FIRST -- they match by annotation/type
        // and should take priority over generic resolvers.
        argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
        argumentResolvers.addResolver(new SessionStatusArgumentResolver());
        argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers -- they take priority
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

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
        // ch20: Servlet API + Locale resolver
        argumentResolvers.addResolver(new ServletRequestArgumentResolver());
        // ch18: Session-aware resolvers
        argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
        argumentResolvers.addResolver(new SessionStatusArgumentResolver());
        argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

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

    /**
     * Handle the request by invoking the handler method, managing session
     * attributes lifecycle around the invocation.
     *
     * <h3>ch18 Enhancement:</h3>
     * The handle() method now wraps invocation with session attribute management:
     * <pre>
     *   1. Get/create SessionAttributesHandler for this controller
     *   2. Retrieve session attributes -> store as request attribute
     *   3. Create SessionStatus -> store as request attribute
     *   4. Resolve arguments (resolvers can find session attrs via request attribute)
     *   5. Invoke handler method
     *   6. Process return value -> ModelAndView
     *   7. If SessionStatus.isComplete() -> cleanup session attributes
     *      Else -> store matching model attributes in session
     *   8. Return ModelAndView
     * </pre>
     *
     * Maps to: {@code RequestMappingHandlerAdapter.invokeHandlerMethod()} (line 885)
     * which creates ModelFactory, initializes the model (retrieves session attrs),
     * invokes the handler, and calls ModelFactory.updateModel() (stores/cleans up).
     */
    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                               Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // ch18: Step 1 -- Get or create SessionAttributesHandler for this controller
        SessionAttributesHandler sessionAttrHandler =
                getSessionAttributesHandler(handlerMethod);

        // ch18: Step 2 -- Retrieve session attributes and store as request attribute
        // so ModelAttributeArgumentResolver can find existing instances
        Map<String, Object> sessionAttrs = sessionAttrHandler.retrieveAttributes(request);
        request.setAttribute(ModelAttributeArgumentResolver.SESSION_ATTRIBUTES_KEY, sessionAttrs);

        // ch18: Step 3 -- Create per-request SessionStatus
        SimpleSessionStatus sessionStatus = new SimpleSessionStatus();
        request.setAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE, sessionStatus);

        // ch20: Store response as request attribute for ServletRequestArgumentResolver
        request.setAttribute(ServletRequestArgumentResolver.RESPONSE_ATTRIBUTE, response);

        // Steps 4-6: Resolve arguments, invoke, process return value
        Object[] args = resolveArguments(handlerMethod, request);
        Object result = invokeHandlerMethod(handlerMethod, args);
        ModelAndView mv = returnValueHandlers.handleReturnValue(
                result, handlerMethod, request, response);

        // ch18: Step 7 -- Session attributes lifecycle
        if (sessionAttrHandler.hasSessionAttributes()) {
            if (sessionStatus.isComplete()) {
                // SessionStatus.setComplete() was called -- cleanup
                sessionAttrHandler.cleanupAttributes(request);
            } else {
                // Store matching model attributes back in session.
                // Two sources of attributes to store:
                // 1. Session attributes retrieved at start (may have been modified by reference)
                sessionAttrHandler.storeAttributes(request, sessionAttrs);
                // 2. New attributes in the ModelAndView model
                if (mv != null) {
                    sessionAttrHandler.storeAttributes(request, mv.getModel());
                }
            }
        }

        return mv;
    }

    private SessionAttributesHandler getSessionAttributesHandler(HandlerMethod handlerMethod) {
        Class<?> handlerType = handlerMethod.getBeanType();
        return this.sessionAttributesHandlerCache.computeIfAbsent(
                handlerType,
                type -> new SessionAttributesHandler(type, this.sessionAttributeStore));
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

    public Map<Class<?>, SessionAttributesHandler> getSessionAttributesHandlerCache() {
        return sessionAttributesHandlerCache;
    }
}
```

### Test Files

#### `src/test/java/com/simplespringmvc/locale/LocaleContextHolderTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/locale/AcceptHeaderLocaleResolverTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/locale/SessionLocaleResolverTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/locale/CookieLocaleResolverTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/locale/LocaleChangeInterceptorTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/i18n/ResourceBundleMessageSourceTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/adapter/ServletRequestArgumentResolverTest.java`

(Full content shown in section 20.9 above)

#### `src/test/java/com/simplespringmvc/integration/LocaleIntegrationTest.java`

(Full content shown in section 20.9 above)

### Test Resources

#### `src/test/resources/messages.properties`

```properties
greeting.hello=Hello
greeting.welcome=Welcome to our site
greeting.with.name=Hello, {0}!
app.title=Simple Spring MVC
nav.home=Home
nav.about=About
```

#### `src/test/resources/messages_fr.properties`

```properties
greeting.hello=Bonjour
greeting.welcome=Bienvenue sur notre site
greeting.with.name=Bonjour, {0} !
app.title=Simple Spring MVC
nav.home=Accueil
nav.about=\u00C0 propos
```

#### `src/test/resources/messages_de.properties`

```properties
greeting.hello=Hallo
greeting.welcome=Willkommen auf unserer Seite
greeting.with.name=Hallo, {0}!
app.title=Simple Spring MVC
nav.home=Startseite
nav.about=\u00DCber uns
```

#### `src/test/resources/templates/hello_fr.html`

```html
<html>
<body>
<h1>Bonjour, ${name} !</h1>
<p>Bienvenue sur Simple Spring MVC.</p>
</body>
</html>
```

#### `src/test/resources/templates/i18n.html`

```html
<html>
<body>
<h1>#{greeting.hello}</h1>
<p>#{greeting.welcome}</p>
<nav>#{nav.home} | #{nav.about}</nav>
</body>
</html>
```

---

## Summary

| Metric | Value |
|--------|-------|
| Files created | 10 (6 locale, 3 i18n, 1 adapter) |
| Files modified | 5 (SimpleDispatcherServlet, ViewResolver, TemplateViewResolver, TemplateView, SimpleHandlerAdapter) |
| Test files | 8 (7 unit + 1 integration) |
| Test resources | 5 (3 message bundles + 2 templates) |
| Lines of production code | ~872 |
| Lines of test code | ~1,095 |
| Key concepts | Strategy pattern (LocaleResolver, MessageSource), Thread-local context (LocaleContextHolder), ResourceBundle-based i18n, Locale-specific view resolution, Request-attribute service locator |

**Next chapter:** [Chapter 21: WebSocket Support](ch21_websocket_support.md) -- Add WebSocket endpoints using the HTTP upgrade mechanism, message handling, and session management for real-time communication.
