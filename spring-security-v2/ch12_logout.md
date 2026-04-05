# Chapter 12: Logout

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure logout via the DSL:
SimpleHttpSecurity http = new SimpleHttpSecurity();

// Defaults (POST /logout -> redirect to /login?logout):
http.sessionManagement(Customizer.withDefaults());
http.logout(Customizer.withDefaults());

// Custom:
http.logout(logout -> logout
    .logoutUrl("/signout")
    .logoutSuccessUrl("/goodbye")
    .invalidateHttpSession(true)
    .deleteCookies("JSESSIONID", "remember-me")
    .addLogoutHandler((req, res, auth) ->
        auditLog.add("User logged out: " + auth.getName()))
    .logoutSuccessHandler((req, res, auth) -> {
        res.setStatus(200);
        res.getWriter().write("Logged out");
    }));

SecurityFilterChain chain = http.build();

// A SimpleLogoutFilter is added at position 250 (after CsrfFilter at 200,
// before UsernamePasswordAuthenticationFilter at 300).
// When a request matches the logout URL:
//   1. Retrieves the current Authentication from SecurityContextHolder
//   2. Invokes each LogoutHandler in order (custom handlers first, then
//      the built-in SecurityContextLogoutHandler last)
//   3. Invokes the LogoutSuccessHandler to send the response
//   4. Returns without continuing the filter chain
```

Questions to guide your thinking:
- Why is logout a two-phase operation (cleanup handlers, then success handler)? Why not combine them into one interface?
- Why does the built-in `SecurityContextLogoutHandler` always run last? What would break if a custom handler ran after it?
- Why does `LogoutHandler.logout()` not throw checked exceptions, while `LogoutSuccessHandler.onLogoutSuccess()` does?
- When CSRF is enabled, only POST triggers logout. Why is this important? What attack does it prevent?
- Why does the logout handler save an empty `SecurityContext` to the repository instead of just invalidating the session?

---

## The API Contract

Feature 12 adds logout handling -- the ability to terminate an authenticated session, clean up security state, and send a response. Without logout, users have no way to end their session, and the `SecurityContext` persists until the session expires naturally.

### The DSL Entry Point

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity logout(Customizer<SimpleLogoutConfigurer> logoutCustomizer) {
    logoutCustomizer.customize(getOrApply(new SimpleLogoutConfigurer()));
    return this;
}
```

### What the client configures

| DSL Method | Purpose | Default |
|-----------|---------|---------|
| `logoutUrl(...)` | URL that triggers logout | `/logout` |
| `logoutSuccessUrl(...)` | Redirect target after logout | `/login?logout` |
| `logoutSuccessHandler(...)` | Custom response handler (overrides success URL) | Redirect to success URL |
| `invalidateHttpSession(...)` | Whether to invalidate the session | `true` |
| `deleteCookies(...)` | Cookie names to clear on logout | None |
| `addLogoutHandler(...)` | Custom cleanup handler (runs before built-in) | None |
| `logoutRequestMatcher(...)` | Custom request matcher (overrides URL+method logic) | Path + POST (with CSRF) or any method (without CSRF) |
| `disable()` | Turn off logout entirely | Enabled |

### The behavioral contract

1. **Request matches logout URL** -> `SimpleLogoutFilter` intercepts
2. **Retrieve authentication** -> reads current `Authentication` from `SecurityContextHolder`
3. **Run cleanup handlers** -> each `LogoutHandler` runs in order (custom handlers first, `SecurityContextLogoutHandler` last)
4. **Send response** -> `LogoutSuccessHandler` sends the HTTP response (redirect or custom)
5. **Filter chain stops** -> the request does NOT continue to downstream filters
6. **Request does not match** -> filter passes through to the next filter in the chain

### CSRF interaction

| CSRF State | Allowed Methods | Why |
|:---:|:---:|---|
| Enabled (configurer present) | POST only | Prevents CSRF attacks that trick a logged-in browser into visiting a logout link |
| Disabled (no configurer) | Any method | GET, POST, DELETE, etc. all trigger logout |

---

## Client Usage & Tests

### Test 1: Default logout -- POST /logout with session invalidation

```java
@Test
void shouldLogoutWithDefaults_PostLogout() throws Exception {
    // Arrange -- build a chain with logout defaults
    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults())
            .logout(Customizer.withDefaults())
            .authenticationManager(acceptingAuthManager()));

    // Authenticate first to establish a session
    MockHttpServletRequest authRequest = new MockHttpServletRequest("GET", "/resource");
    authRequest.addHeader("Authorization", basicAuth("user", "password"));
    MockHttpServletResponse authResponse = new MockHttpServletResponse();
    proxy.doFilter(authRequest, authResponse, new MockFilterChain());
    MockHttpSession session = (MockHttpSession) authRequest.getSession(false);
    assertThat(session).isNotNull();

    // Act -- POST /logout with the session
    MockHttpServletRequest logoutRequest = new MockHttpServletRequest("POST", "/logout");
    logoutRequest.setSession(session);
    MockHttpServletResponse logoutResponse = new MockHttpServletResponse();
    proxy.doFilter(logoutRequest, logoutResponse, new MockFilterChain());

    // Assert -- redirected to /login?logout, session invalidated, context cleared
    assertThat(logoutResponse.getRedirectedUrl()).isEqualTo("/login?logout");
    assertThat(logoutRequest.getSession(false)).isNull(); // Session invalidated
    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
}
```

### Test 2: Non-logout request passes through

```java
@Test
void shouldPassThrough_WhenUrlDoesNotMatchLogout() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .logout(Customizer.withDefaults())
            .authenticationManager(acceptingAuthManager()));

    // Act -- request a normal URL, not /logout
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/home");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(request, response, chain);

    // Assert -- request passed through
    assertThat(chain.getRequest()).isNotNull();
    assertThat(response.getRedirectedUrl()).isNull();
}
```

### Test 3: Custom logout URL and success URL

```java
@Test
void shouldUseCustomLogoutUrlAndSuccessUrl() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .logout(logout -> logout
                    .logoutUrl("/signout")
                    .logoutSuccessUrl("/goodbye"))
            .csrf(csrf -> csrf.disable()) // Disable CSRF so GET works
            .authenticationManager(acceptingAuthManager()));

    // Act -- GET /signout (allowed since CSRF is disabled)
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/signout");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Assert -- redirected to custom success URL
    assertThat(response.getRedirectedUrl()).isEqualTo("/goodbye");
}
```

### Test 4: Custom success handler (REST API style)

```java
@Test
void shouldUseCustomLogoutSuccessHandler() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .logout(logout -> logout
                    .logoutSuccessHandler((request, response, auth) -> {
                        response.setStatus(200);
                        response.getWriter().write("Logged out");
                    }))
            .csrf(csrf -> csrf.disable())
            .authenticationManager(acceptingAuthManager()));

    // Act
    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Assert -- custom handler was invoked
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(response.getContentAsString()).isEqualTo("Logged out");
}
```

### Test 5: Cookie clearing on logout

```java
@Test
void shouldDeleteCookiesOnLogout() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .logout(logout -> logout
                    .deleteCookies("JSESSIONID", "remember-me"))
            .csrf(csrf -> csrf.disable())
            .authenticationManager(acceptingAuthManager()));

    // Act
    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Assert -- cookies cleared (maxAge=0)
    Cookie[] cookies = response.getCookies();
    assertThat(cookies).hasSize(2);
    assertThat(cookies[0].getName()).isEqualTo("JSESSIONID");
    assertThat(cookies[0].getMaxAge()).isZero();
    assertThat(cookies[1].getName()).isEqualTo("remember-me");
    assertThat(cookies[1].getMaxAge()).isZero();
}
```

### Test 6: Custom logout handler (audit logging)

```java
@Test
void shouldInvokeCustomLogoutHandler() throws Exception {
    List<String> auditLog = new ArrayList<>();

    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .logout(logout -> logout
                    .addLogoutHandler((req, res, auth) ->
                            auditLog.add("User logged out: " + (auth != null ? auth.getName() : "anonymous"))))
            .csrf(csrf -> csrf.disable())
            .authenticationManager(acceptingAuthManager()));

    // Set up authentication
    SecurityContextHolder.getContext().setAuthentication(
            UsernamePasswordAuthenticationToken.authenticated(
                    "admin", null, List.of(new SimpleGrantedAuthority("ROLE_ADMIN"))));

    // Act
    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Assert -- custom handler ran with the authentication
    assertThat(auditLog).containsExactly("User logged out: admin");
}
```

### Test 7: CSRF-enabled logout requires POST

```java
@Test
void shouldRequirePost_WhenCsrfIsEnabled() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> http
            .sessionManagement(Customizer.withDefaults())
            .csrf(Customizer.withDefaults())
            .logout(Customizer.withDefaults())
            .authenticationManager(acceptingAuthManager()));

    // GET /logout should pass through (not trigger logout) when CSRF is enabled
    MockHttpServletRequest getRequest = new MockHttpServletRequest("GET", "/logout");
    MockHttpServletResponse getResponse = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(getRequest, getResponse, chain);

    assertThat(getResponse.getRedirectedUrl()).isNull();
    assertThat(chain.getRequest()).isNotNull(); // Chain continued
}
```

---

## Implementing the Call Chain

Logout is built as a layered pipeline. Let's trace through each layer from the strategy interfaces down to the DSL wiring.

### Layer 1: The Strategy Interfaces

Logout is a **two-phase operation** with two separate interfaces:

**Phase 1 -- Cleanup** (`LogoutHandler`): Performs silent cleanup. Must not throw. All handlers run regardless of earlier failures.

```java
// LogoutHandler.java
@FunctionalInterface
public interface LogoutHandler {
    void logout(HttpServletRequest request, HttpServletResponse response,
                Authentication authentication);
}
```

**Phase 2 -- Response** (`LogoutSuccessHandler`): Sends the HTTP response. May throw `IOException`/`ServletException` because it performs I/O. Exactly one success handler runs per logout.

```java
// LogoutSuccessHandler.java
@FunctionalInterface
public interface LogoutSuccessHandler {
    void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response,
                         Authentication authentication) throws IOException, ServletException;
}
```

**Why two interfaces?** Cleanup and response are fundamentally different operations. Cleanup handlers should be resilient -- if cookie clearing fails, session invalidation should still happen. The success handler is the final step that commits the HTTP response. Mixing them would force a choice: should a failed redirect prevent session invalidation? The two-phase design avoids this dilemma.

### Layer 2: SimpleSecurityContextLogoutHandler (core cleanup)

The built-in handler that always runs last. It performs three actions:

```java
// SimpleSecurityContextLogoutHandler.java
public class SimpleSecurityContextLogoutHandler implements LogoutHandler {

    private boolean invalidateHttpSession = true;
    private boolean clearAuthentication = true;
    private SecurityContextRepository securityContextRepository;

    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response,
                       Authentication authentication) {
        // Step 1: Invalidate the HTTP session if configured
        if (this.invalidateHttpSession) {
            HttpSession session = request.getSession(false);
            if (session != null) {
                session.invalidate();
            }
        }

        // Step 2: Clear the SecurityContext from the ThreadLocal holder
        SecurityContext context = SecurityContextHolder.getContext();
        SecurityContextHolder.clearContext();

        // Step 3: Clear authentication from the context object itself
        if (this.clearAuthentication) {
            context.setAuthentication(null);
        }

        // Step 4: Persist an empty context (clears from repository/session attribute)
        if (this.securityContextRepository != null) {
            SecurityContext emptyContext = new SecurityContextImpl();
            this.securityContextRepository.saveContext(emptyContext, request, response);
        }
    }
}
```

**Why save an empty context?** Simply invalidating the session is not enough if a custom `SecurityContextRepository` is in use (e.g., one that persists to a database or Redis). Saving an empty context ensures the persistent store is also cleaned up.

**Why clear both the ThreadLocal and the context object?** The ThreadLocal (`SecurityContextHolder`) holds a reference to the `SecurityContext` object. Other code may also hold a reference to that same object. Setting the authentication to `null` on the context object itself ensures no lingering references can access the old credentials.

### Layer 3: SimpleCookieClearingLogoutHandler (cookie cleanup)

Clears specified cookies by adding cookies with `maxAge=0` to the response:

```java
// SimpleCookieClearingLogoutHandler.java
public class SimpleCookieClearingLogoutHandler implements LogoutHandler {

    private final String[] cookieNames;

    public SimpleCookieClearingLogoutHandler(String... cookieNames) {
        if (cookieNames == null || cookieNames.length == 0) {
            throw new IllegalArgumentException("At least one cookie name must be provided");
        }
        this.cookieNames = cookieNames;
    }

    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response,
                       Authentication authentication) {
        String contextPath = request.getContextPath();
        String cookiePath = (contextPath != null && !contextPath.isEmpty())
                ? contextPath : "/";

        for (String cookieName : this.cookieNames) {
            Cookie cookie = new Cookie(cookieName, null);
            cookie.setPath(cookiePath);
            cookie.setMaxAge(0);          // Tells browser to delete
            cookie.setSecure(request.isSecure());
            response.addCookie(cookie);
        }
    }
}
```

**Why use the context path?** Cookies are scoped by path. A cookie set at `/myapp` won't be cleared by a cookie deletion at `/`. Using the request's context path ensures the deletion targets the correct scope.

### Layer 4: SimpleLogoutFilter (the orchestrator)

The filter that ties everything together. It sits at position 250 in the filter chain:

```java
// SimpleLogoutFilter.java
public class SimpleLogoutFilter implements Filter {

    private final LogoutSuccessHandler logoutSuccessHandler;
    private final List<LogoutHandler> handlers;
    private RequestMatcher logoutRequestMatcher;

    public SimpleLogoutFilter(LogoutSuccessHandler logoutSuccessHandler,
                              List<LogoutHandler> handlers) {
        // Validation omitted for brevity
        this.logoutSuccessHandler = logoutSuccessHandler;
        this.handlers = List.copyOf(handlers);  // Defensive copy
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        if (requiresLogout(request)) {
            // Phase 1: Get the current authentication
            Authentication authentication =
                    SecurityContextHolder.getContext().getAuthentication();

            // Phase 2: Run all cleanup handlers
            for (LogoutHandler handler : this.handlers) {
                handler.logout(request, response, authentication);
            }

            // Phase 3: Send the response
            this.logoutSuccessHandler.onLogoutSuccess(request, response, authentication);
            return; // Do NOT continue the filter chain
        }

        chain.doFilter(request, response);
    }

    private boolean requiresLogout(HttpServletRequest request) {
        return this.logoutRequestMatcher != null
                && this.logoutRequestMatcher.matches(request);
    }
}
```

**Key design decisions:**
- The authentication is captured *before* handlers run, because handlers will clear it from the `SecurityContextHolder`. The success handler receives the original authentication so it can log or display the username.
- `List.copyOf()` creates a defensive copy -- handlers cannot be modified after construction.
- The filter returns immediately after the success handler -- the request never reaches downstream filters like authorization.

### Layer 5: SimpleLogoutConfigurer (DSL wiring)

The configurer assembles all the pieces during the build lifecycle:

```java
// SimpleLogoutConfigurer.java
public class SimpleLogoutConfigurer
        extends SimpleAbstractHttpConfigurer<SimpleLogoutConfigurer> {

    private final List<LogoutHandler> logoutHandlers = new ArrayList<>();
    private final SimpleSecurityContextLogoutHandler contextLogoutHandler =
            new SimpleSecurityContextLogoutHandler();
    private String logoutUrl = "/logout";
    private String logoutSuccessUrl = "/login?logout";
    private LogoutSuccessHandler logoutSuccessHandler;
    private RequestMatcher logoutRequestMatcher;

    @Override
    public void configure(SimpleHttpSecurity builder) throws Exception {
        // Wire the SecurityContextRepository into the context logout handler
        SecurityContextRepository repository =
                builder.getSharedObject(SecurityContextRepository.class);
        if (repository != null) {
            this.contextLogoutHandler.setSecurityContextRepository(repository);
        }

        // Build the handler list: user handlers first, built-in last
        List<LogoutHandler> handlers = new ArrayList<>(this.logoutHandlers);
        handlers.add(this.contextLogoutHandler);

        // Build the success handler (custom or default redirect)
        LogoutSuccessHandler successHandler = resolveLogoutSuccessHandler();

        // Build the request matcher
        RequestMatcher matcher = resolveLogoutRequestMatcher(builder);

        // Assemble and add the filter
        SimpleLogoutFilter filter = new SimpleLogoutFilter(successHandler, handlers);
        filter.setLogoutRequestMatcher(matcher);
        builder.addFilter(filter);
    }
}
```

**The CSRF-aware request matcher:**

```java
private RequestMatcher resolveLogoutRequestMatcher(SimpleHttpSecurity builder) {
    if (this.logoutRequestMatcher != null) {
        return this.logoutRequestMatcher;
    }
    RequestMatcher pathMatcher = new SimplePathPatternRequestMatcher(this.logoutUrl);
    boolean csrfEnabled = builder.getConfigurer(SimpleCsrfConfigurer.class) != null;
    if (csrfEnabled) {
        // When CSRF is enabled, only POST is allowed
        return request -> pathMatcher.matches(request)
                && "POST".equalsIgnoreCase(request.getMethod());
    }
    // When CSRF is disabled, accept any HTTP method
    return pathMatcher;
}
```

The configurer checks whether a `SimpleCsrfConfigurer` is registered by calling `builder.getConfigurer(SimpleCsrfConfigurer.class)`. This is a loose coupling -- the logout configurer never imports or depends on the CSRF filter itself, only checks for the configurer's presence.

### Layer 6: SimpleHttpSecurity modifications

Two changes to the builder:

**1. Filter order registration** -- `SimpleLogoutFilter` is registered at position 250:

```java
static {
    int order = 100;
    FILTER_ORDER_MAP.put("SimpleSecurityContextHolderFilter", order);
    order += 100;
    FILTER_ORDER_MAP.put("SimpleCsrfFilter", order);            // 200
    order += 50;
    FILTER_ORDER_MAP.put("SimpleLogoutFilter", order);           // 250
    order += 50;
    FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", order); // 300
    // ...
}
```

**2. DSL method** -- the `logout()` convenience method:

```java
public SimpleHttpSecurity logout(Customizer<SimpleLogoutConfigurer> logoutCustomizer) {
    logoutCustomizer.customize(getOrApply(new SimpleLogoutConfigurer()));
    return this;
}
```

---

## Try It Yourself

1. **Remember-me cleanup**: When remember-me authentication is implemented, logout needs to clear the remember-me token (both the cookie and the persistent token in the database). Implement a `RememberMeLogoutHandler` that takes a `PersistentTokenRepository` and deletes the user's tokens on logout.

2. **Event publishing**: Add a logout event that is published after all handlers complete but before the success handler runs. You'll need a `LogoutSuccessEvent` class and an `ApplicationEventPublisher`. Where in the filter would you add the publish call?

3. **Conditional logout URL**: Implement a logout configurer that supports different logout URLs for different paths (e.g., `/api/logout` for REST APIs and `/web/logout` for browser sessions). How would you compose multiple request matchers?

4. **Rate limiting**: Add a logout handler that rate-limits logout requests per session to prevent abuse (e.g., a script that hammers the logout endpoint). What data structure would you use to track request counts?

---

## Why This Works

### The Two-Phase Design
Separating cleanup (`LogoutHandler`) from response (`LogoutSuccessHandler`) is the key architectural decision. Cleanup handlers are fail-safe -- they must not throw exceptions, and all handlers run regardless of earlier failures. The success handler is fail-last -- it runs only after all cleanup completes, and its failure is the only one that propagates. This guarantees that security state is always cleaned up, even if the redirect fails.

### The Composite Handler Pattern
The `SimpleLogoutFilter` holds a `List<LogoutHandler>`, not a single handler. This is the **Composite pattern** applied to cleanup -- the filter doesn't know or care whether there's one handler or ten. Adding cookie clearing, audit logging, or token revocation is just adding another handler to the list. The configurer ensures the built-in `SecurityContextLogoutHandler` always runs last, so custom handlers can still access the session and context.

### Cross-Configurer Communication via Shared Objects
The logout configurer reads the `SecurityContextRepository` from shared objects (set by `SessionManagementConfigurer` during `init()`). Neither configurer references the other directly. This is the same pattern used throughout the framework -- shared objects are Spring Security's internal service locator, enabling loose coupling between features.

### CSRF-Aware Method Restriction
When CSRF protection is enabled, logout requires POST. This prevents a common attack: an attacker embeds `<img src="https://bank.com/logout">` in a page, and the browser automatically sends a GET request that logs the user out. Requiring POST means the attacker would need a form submission with a valid CSRF token, which they cannot forge.

---

## What We Enhanced

| Component | Previous State | Enhancement in Feature 12 |
|-----------|---------------|--------------------------|
| `SimpleHttpSecurity` | Filter order map had entries up to `SimpleAuthorizationFilter`; DSL had methods for session, CSRF, form login, HTTP basic, exception handling | Added `SimpleLogoutFilter` at position 250 in filter order map; added `logout(Customizer)` DSL method |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | File |
|------------------|----------------------|------|
| `LogoutHandler` | `LogoutHandler` | `web/src/main/java/.../web/authentication/logout/LogoutHandler.java` |
| `LogoutSuccessHandler` | `LogoutSuccessHandler` | `web/src/main/java/.../web/authentication/logout/LogoutSuccessHandler.java` |
| `SimpleSecurityContextLogoutHandler` | `SecurityContextLogoutHandler` | `web/src/main/java/.../web/authentication/logout/SecurityContextLogoutHandler.java` |
| `SimpleCookieClearingLogoutHandler` | `CookieClearingLogoutHandler` | `web/src/main/java/.../web/authentication/logout/CookieClearingLogoutHandler.java` |
| `SimpleLogoutFilter` | `LogoutFilter` | `web/src/main/java/.../web/authentication/logout/LogoutFilter.java` |
| `SimpleLogoutConfigurer` | `LogoutConfigurer` | `config/src/main/java/.../configurers/LogoutConfigurer.java` |

### What the real framework adds

- **`CompositeLogoutHandler`**: Wraps multiple `LogoutHandler` instances and calls them in sequence. Our `SimpleLogoutFilter` handles this directly with a `List<LogoutHandler>`, but the real framework extracts it into a separate class for reuse outside the filter
- **`SimpleUrlLogoutSuccessHandler`**: A dedicated class for redirect-based success handling with URL encoding, context-relative redirects, and target URL parameter support. Our version uses an inline lambda
- **`HttpStatusReturningLogoutSuccessHandler`**: Returns a configurable HTTP status code (default 200) with no body -- designed for REST APIs
- **`HeaderWriterLogoutHandler`**: Clears security-sensitive response headers on logout (e.g., `Clear-Site-Data` header that instructs the browser to clear cookies, storage, and cache)
- **`DelegatingLogoutSuccessHandler`**: Routes to different success handlers based on `RequestMatcher` -- enables different logout responses for browser vs. API clients
- **`LogoutConfigurer.defaultLogoutSuccessHandlerFor()`**: Registers success handlers per request matcher pattern, enabling content-negotiated logout responses
- **CSRF token clearing**: The real `LogoutConfigurer` automatically adds a `CsrfLogoutHandler` that clears the CSRF token on logout, forcing a new token for the next session
- **Logout success URL validation**: The real configurer validates the success URL and supports URL encoding and context-relative paths
- **`LogoutHandler` ordering guarantees**: The real `SecurityContextLogoutHandler` has explicit ordering support to ensure it always runs last, even when handlers are added in arbitrary order

---

## Complete Code

### New Files

#### `[NEW] simple-security-web/src/main/java/simple/security/web/authentication/logout/LogoutHandler.java`

```java
package simple.security.web.authentication.logout;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;

/**
 * Strategy interface for components that perform cleanup during logout.
 *
 * <p>Implementations handle tasks like invalidating sessions, clearing the
 * {@link simple.security.core.context.SecurityContext SecurityContext}, or
 * removing cookies. The {@link SimpleLogoutFilter} invokes all registered
 * handlers before delegating to the {@link LogoutSuccessHandler}.
 *
 * <h2>Contract</h2>
 * <ul>
 *   <li>Implementations should complete silently — do NOT throw exceptions</li>
 *   <li>All handlers run regardless of failures in earlier ones</li>
 *   <li>The {@code authentication} parameter may be {@code null} if no user
 *       is currently authenticated</li>
 * </ul>
 *
 * <p>This is a functional interface whose functional method is
 * {@link #logout(HttpServletRequest, HttpServletResponse, Authentication)}.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.logout.LogoutHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.logout.LogoutHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/logout/LogoutHandler.java
 */
@FunctionalInterface
public interface LogoutHandler {

	/**
	 * Performs a logout action (e.g., invalidate session, clear context).
	 *
	 * @param request        the HTTP request
	 * @param response       the HTTP response
	 * @param authentication the current authentication, or {@code null}
	 */
	void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication);

}
```

#### `[NEW] simple-security-web/src/main/java/simple/security/web/authentication/logout/LogoutSuccessHandler.java`

```java
package simple.security.web.authentication.logout;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;

/**
 * Strategy interface for handling the response after a successful logout.
 *
 * <p>Called by {@link SimpleLogoutFilter} after all {@link LogoutHandler}
 * instances have completed their cleanup. The success handler is responsible
 * for sending the HTTP response — typically a redirect to a login page or a
 * JSON response for REST APIs.
 *
 * <h2>Contract</h2>
 * <ul>
 *   <li>May throw {@link IOException} or {@link ServletException} (unlike
 *       {@link LogoutHandler}, this interface performs I/O)</li>
 *   <li>The {@code authentication} parameter may be {@code null} if no user
 *       was authenticated when the logout URL was accessed</li>
 *   <li>Exactly one success handler is invoked per logout request</li>
 * </ul>
 *
 * <p>This is a functional interface whose functional method is
 * {@link #onLogoutSuccess(HttpServletRequest, HttpServletResponse, Authentication)}.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.logout.LogoutSuccessHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.logout.LogoutSuccessHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/logout/LogoutSuccessHandler.java
 */
@FunctionalInterface
public interface LogoutSuccessHandler {

	/**
	 * Handles the response after a successful logout.
	 *
	 * @param request        the HTTP request
	 * @param response       the HTTP response
	 * @param authentication the authentication that was active before logout, or {@code null}
	 * @throws IOException      if an I/O error occurs
	 * @throws ServletException if a servlet error occurs
	 */
	void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) throws IOException, ServletException;

}
```

#### `[NEW] simple-security-web/src/main/java/simple/security/web/authentication/logout/SimpleSecurityContextLogoutHandler.java`

```java
package simple.security.web.authentication.logout;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

import simple.security.core.Authentication;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.context.SecurityContextImpl;
import simple.security.web.context.SecurityContextRepository;

/**
 * A {@link LogoutHandler} that clears the {@link SecurityContext} and optionally
 * invalidates the HTTP session.
 *
 * <p>This is the default logout handler and always runs as part of the logout
 * flow. It performs three actions:
 * <ol>
 *   <li>Invalidates the {@link HttpSession} (if configured and if one exists)</li>
 *   <li>Clears the {@link SecurityContextHolder} (removes the ThreadLocal)</li>
 *   <li>Saves an empty {@link SecurityContext} to the
 *       {@link SecurityContextRepository} (removes the session attribute)</li>
 * </ol>
 *
 * <h2>Why save an empty context?</h2>
 * <p>Simply invalidating the session is not enough if a custom
 * {@link SecurityContextRepository} is in use (e.g., one that persists to a
 * database). Saving an empty context ensures the persistent store is also cleaned up.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.logout.SecurityContextLogoutHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.logout.SecurityContextLogoutHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/logout/SecurityContextLogoutHandler.java
 */
public class SimpleSecurityContextLogoutHandler implements LogoutHandler {

	private boolean invalidateHttpSession = true;

	private boolean clearAuthentication = true;

	private SecurityContextRepository securityContextRepository;

	@Override
	public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
		// Step 1: Invalidate the HTTP session if configured
		if (this.invalidateHttpSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				session.invalidate();
			}
		}

		// Step 2: Clear the SecurityContext from the ThreadLocal holder
		SecurityContext context = SecurityContextHolder.getContext();
		SecurityContextHolder.clearContext();

		// Step 3: Clear authentication from the context object itself
		if (this.clearAuthentication) {
			context.setAuthentication(null);
		}

		// Step 4: Persist an empty context (clears from repository/session attribute)
		if (this.securityContextRepository != null) {
			SecurityContext emptyContext = new SecurityContextImpl();
			this.securityContextRepository.saveContext(emptyContext, request, response);
		}
	}

	// --- Configuration setters ---

	/**
	 * Sets whether the {@link HttpSession} should be invalidated during logout.
	 * Defaults to {@code true}.
	 *
	 * @param invalidateHttpSession {@code true} to invalidate the session
	 */
	public void setInvalidateHttpSession(boolean invalidateHttpSession) {
		this.invalidateHttpSession = invalidateHttpSession;
	}

	/**
	 * Sets whether the {@link Authentication} should be cleared from the
	 * {@link SecurityContext}. Defaults to {@code true}.
	 *
	 * @param clearAuthentication {@code true} to clear authentication
	 */
	public void setClearAuthentication(boolean clearAuthentication) {
		this.clearAuthentication = clearAuthentication;
	}

	/**
	 * Sets the {@link SecurityContextRepository} used to persist the empty context
	 * after logout. This is typically set by the {@link simple.security.config.annotation.web.configurers.SimpleLogoutConfigurer}
	 * during the configure phase.
	 *
	 * @param securityContextRepository the repository to use
	 */
	public void setSecurityContextRepository(SecurityContextRepository securityContextRepository) {
		this.securityContextRepository = securityContextRepository;
	}

	public boolean isInvalidateHttpSession() {
		return this.invalidateHttpSession;
	}

}
```

#### `[NEW] simple-security-web/src/main/java/simple/security/web/authentication/logout/SimpleCookieClearingLogoutHandler.java`

```java
package simple.security.web.authentication.logout;

import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;

/**
 * A {@link LogoutHandler} that clears specified cookies on logout.
 *
 * <p>For each cookie name, adds a cookie with {@code maxAge=0} to the response,
 * which instructs the browser to delete it. The cookie path is set to the
 * request's context path (or "/" if the context path is empty).
 *
 * <h2>Usage</h2>
 * <pre>
 * // Via the DSL:
 * http.logout(logout -> logout
 *     .deleteCookies("JSESSIONID", "remember-me"));
 *
 * // Or directly:
 * LogoutHandler handler = new SimpleCookieClearingLogoutHandler("JSESSIONID");
 * </pre>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.logout.CookieClearingLogoutHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.logout.CookieClearingLogoutHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/logout/CookieClearingLogoutHandler.java
 */
public class SimpleCookieClearingLogoutHandler implements LogoutHandler {

	private final String[] cookieNames;

	/**
	 * Creates a handler that will clear the specified cookies on logout.
	 *
	 * @param cookieNames the names of cookies to clear
	 */
	public SimpleCookieClearingLogoutHandler(String... cookieNames) {
		if (cookieNames == null || cookieNames.length == 0) {
			throw new IllegalArgumentException("At least one cookie name must be provided");
		}
		this.cookieNames = cookieNames;
	}

	@Override
	public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
		String contextPath = request.getContextPath();
		String cookiePath = (contextPath != null && !contextPath.isEmpty()) ? contextPath : "/";

		for (String cookieName : this.cookieNames) {
			Cookie cookie = new Cookie(cookieName, null);
			cookie.setPath(cookiePath);
			cookie.setMaxAge(0);
			cookie.setSecure(request.isSecure());
			response.addCookie(cookie);
		}
	}

}
```

#### `[NEW] simple-security-web/src/main/java/simple/security/web/authentication/logout/SimpleLogoutFilter.java`

```java
package simple.security.web.authentication.logout;

import java.io.IOException;
import java.util.List;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.util.matcher.RequestMatcher;

/**
 * Intercepts logout requests and performs the logout operation.
 *
 * <p>When a request matches the configured logout URL (default: {@code POST /logout}),
 * this filter:
 * <ol>
 *   <li>Retrieves the current {@link Authentication} from the {@link SecurityContextHolder}</li>
 *   <li>Invokes each {@link LogoutHandler} in order (session invalidation, context clearing, cookie removal, etc.)</li>
 *   <li>Invokes the {@link LogoutSuccessHandler} to send the response (typically a redirect)</li>
 *   <li>Returns without continuing the filter chain</li>
 * </ol>
 *
 * <p>If the request does NOT match the logout URL, the filter passes it through
 * to the next filter in the chain.
 *
 * <h2>Filter ordering</h2>
 * <p>Positioned after {@code SimpleCsrfFilter} (200) and before
 * {@code SimpleUsernamePasswordAuthenticationFilter} (300), at position 250.
 * This ensures CSRF validation has already occurred when CSRF protection is enabled.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.logout.LogoutFilter}.
 *
 * Mirrors: org.springframework.security.web.authentication.logout.LogoutFilter
 * Source:  web/src/main/java/org/springframework/security/web/authentication/logout/LogoutFilter.java
 */
public class SimpleLogoutFilter implements Filter {

	private final LogoutSuccessHandler logoutSuccessHandler;

	private final List<LogoutHandler> handlers;

	private RequestMatcher logoutRequestMatcher;

	/**
	 * Creates a logout filter with the given success handler and logout handlers.
	 *
	 * @param logoutSuccessHandler invoked after all handlers complete
	 * @param handlers             the logout handlers to invoke in order
	 */
	public SimpleLogoutFilter(LogoutSuccessHandler logoutSuccessHandler, List<LogoutHandler> handlers) {
		if (logoutSuccessHandler == null) {
			throw new IllegalArgumentException("logoutSuccessHandler cannot be null");
		}
		if (handlers == null || handlers.isEmpty()) {
			throw new IllegalArgumentException("At least one LogoutHandler must be provided");
		}
		this.logoutSuccessHandler = logoutSuccessHandler;
		this.handlers = List.copyOf(handlers);
	}

	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
			FilterChain chain) throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) servletRequest;
		HttpServletResponse response = (HttpServletResponse) servletResponse;

		if (requiresLogout(request)) {
			Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
			for (LogoutHandler handler : this.handlers) {
				handler.logout(request, response, authentication);
			}
			this.logoutSuccessHandler.onLogoutSuccess(request, response, authentication);
			return; // Do NOT continue the filter chain
		}

		chain.doFilter(request, response);
	}

	/**
	 * Checks if the request matches the logout URL.
	 */
	private boolean requiresLogout(HttpServletRequest request) {
		return this.logoutRequestMatcher != null && this.logoutRequestMatcher.matches(request);
	}

	/**
	 * Sets the {@link RequestMatcher} used to determine if a request is a logout request.
	 *
	 * @param logoutRequestMatcher the matcher
	 */
	public void setLogoutRequestMatcher(RequestMatcher logoutRequestMatcher) {
		this.logoutRequestMatcher = logoutRequestMatcher;
	}

}
```

#### `[NEW] simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleLogoutConfigurer.java`

```java
package simple.security.config.annotation.web.configurers;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.authentication.logout.LogoutHandler;
import simple.security.web.authentication.logout.LogoutSuccessHandler;
import simple.security.web.authentication.logout.SimpleCookieClearingLogoutHandler;
import simple.security.web.authentication.logout.SimpleLogoutFilter;
import simple.security.web.authentication.logout.SimpleSecurityContextLogoutHandler;
import simple.security.web.context.SecurityContextRepository;
import simple.security.web.util.matcher.RequestMatcher;
import simple.security.web.util.matcher.SimplePathPatternRequestMatcher;

/**
 * Configures logout handling via the {@link SimpleHttpSecurity} DSL.
 *
 * <h2>Usage</h2>
 * <pre>
 * http.logout(logout -> logout
 *     .logoutUrl("/logout")
 *     .logoutSuccessUrl("/login?logout")
 *     .invalidateHttpSession(true)
 *     .deleteCookies("JSESSIONID")
 *     .addLogoutHandler(customHandler)
 *     .logoutSuccessHandler(customSuccessHandler));
 * </pre>
 *
 * <h2>What it does</h2>
 * <p>During the build lifecycle:
 * <ol>
 *   <li><b>init()</b> — no-op (logout does not need to set up shared state)</li>
 *   <li><b>configure()</b> — assembles the {@link SimpleLogoutFilter} with:
 *       <ul>
 *         <li>A request matcher for the logout URL (POST-only when CSRF is enabled)</li>
 *         <li>User-added {@link LogoutHandler}s + the built-in
 *             {@link SimpleSecurityContextLogoutHandler} (always last)</li>
 *         <li>The configured {@link LogoutSuccessHandler} (redirect by default)</li>
 *       </ul>
 *   </li>
 * </ol>
 *
 * <h2>Default behavior</h2>
 * <ul>
 *   <li>Logout URL: {@code /logout}</li>
 *   <li>Success URL: {@code /login?logout} (redirect after logout)</li>
 *   <li>Session invalidation: enabled</li>
 *   <li>SecurityContext clearing: enabled</li>
 * </ul>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.web.configurers.LogoutConfigurer}.
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.LogoutConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/LogoutConfigurer.java
 */
public class SimpleLogoutConfigurer extends SimpleAbstractHttpConfigurer<SimpleLogoutConfigurer> {

	private final List<LogoutHandler> logoutHandlers = new ArrayList<>();

	private final SimpleSecurityContextLogoutHandler contextLogoutHandler =
			new SimpleSecurityContextLogoutHandler();

	private String logoutUrl = "/logout";

	private String logoutSuccessUrl = "/login?logout";

	private LogoutSuccessHandler logoutSuccessHandler;

	private RequestMatcher logoutRequestMatcher;

	// === Lifecycle ===

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		// Wire the SecurityContextRepository into the context logout handler
		SecurityContextRepository repository = builder.getSharedObject(SecurityContextRepository.class);
		if (repository != null) {
			this.contextLogoutHandler.setSecurityContextRepository(repository);
		}

		// Build the handler list: user handlers first, then the built-in context handler last
		List<LogoutHandler> handlers = new ArrayList<>(this.logoutHandlers);
		handlers.add(this.contextLogoutHandler);

		// Build the success handler (custom or default redirect)
		LogoutSuccessHandler successHandler = resolveLogoutSuccessHandler();

		// Build the request matcher
		RequestMatcher matcher = resolveLogoutRequestMatcher(builder);

		// Assemble and add the filter
		SimpleLogoutFilter filter = new SimpleLogoutFilter(successHandler, handlers);
		filter.setLogoutRequestMatcher(matcher);
		builder.addFilter(filter);
	}

	// === DSL methods ===

	/**
	 * Sets the URL that triggers logout. Defaults to {@code /logout}.
	 *
	 * @param logoutUrl the logout URL
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer logoutUrl(String logoutUrl) {
		this.logoutUrl = logoutUrl;
		this.logoutRequestMatcher = null; // Clear custom matcher when URL changes
		return this;
	}

	/**
	 * Sets the URL to redirect to after successful logout. Defaults to {@code /login?logout}.
	 *
	 * @param logoutSuccessUrl the redirect URL
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer logoutSuccessUrl(String logoutSuccessUrl) {
		this.logoutSuccessUrl = logoutSuccessUrl;
		this.logoutSuccessHandler = null; // Clear custom handler when URL changes
		return this;
	}

	/**
	 * Sets a custom {@link LogoutSuccessHandler}. When set, this takes precedence
	 * over {@link #logoutSuccessUrl}.
	 *
	 * @param logoutSuccessHandler the handler
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer logoutSuccessHandler(LogoutSuccessHandler logoutSuccessHandler) {
		this.logoutSuccessHandler = logoutSuccessHandler;
		return this;
	}

	/**
	 * Sets whether the HTTP session should be invalidated during logout.
	 * Defaults to {@code true}.
	 *
	 * @param invalidateHttpSession {@code true} to invalidate the session
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer invalidateHttpSession(boolean invalidateHttpSession) {
		this.contextLogoutHandler.setInvalidateHttpSession(invalidateHttpSession);
		return this;
	}

	/**
	 * Adds cookie names that should be cleared on logout.
	 *
	 * @param cookieNames the cookie names to delete
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer deleteCookies(String... cookieNames) {
		this.logoutHandlers.add(new SimpleCookieClearingLogoutHandler(cookieNames));
		return this;
	}

	/**
	 * Adds a custom {@link LogoutHandler} to the logout flow.
	 *
	 * <p>Custom handlers execute before the built-in
	 * {@link SimpleSecurityContextLogoutHandler}. This allows custom handlers
	 * to access the session and security context before they are cleared.
	 *
	 * @param logoutHandler the handler to add
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer addLogoutHandler(LogoutHandler logoutHandler) {
		if (logoutHandler == null) {
			throw new IllegalArgumentException("logoutHandler cannot be null");
		}
		this.logoutHandlers.add(logoutHandler);
		return this;
	}

	/**
	 * Sets a custom {@link RequestMatcher} to determine which requests trigger logout.
	 * When set, this takes precedence over the matcher derived from {@link #logoutUrl}.
	 *
	 * @param logoutRequestMatcher the matcher
	 * @return this configurer for chaining
	 */
	public SimpleLogoutConfigurer logoutRequestMatcher(RequestMatcher logoutRequestMatcher) {
		this.logoutRequestMatcher = logoutRequestMatcher;
		return this;
	}

	// === Internal ===

	/**
	 * Returns the configured or default {@link LogoutSuccessHandler}.
	 *
	 * <p>If no custom handler is set, creates a handler that redirects to
	 * the configured {@link #logoutSuccessUrl}.
	 */
	private LogoutSuccessHandler resolveLogoutSuccessHandler() {
		if (this.logoutSuccessHandler != null) {
			return this.logoutSuccessHandler;
		}
		// Default: redirect to the success URL
		String url = this.logoutSuccessUrl;
		return (request, response, authentication) -> response.sendRedirect(url);
	}

	/**
	 * Builds the {@link RequestMatcher} for the logout URL.
	 *
	 * <p>When CSRF protection is enabled (a {@link SimpleCsrfConfigurer} is registered),
	 * only POST requests are accepted for logout. When CSRF is disabled, any HTTP method
	 * can trigger logout.
	 *
	 * @param builder the HttpSecurity builder (used to check for CSRF configurer)
	 * @return the request matcher for the logout URL
	 */
	private RequestMatcher resolveLogoutRequestMatcher(SimpleHttpSecurity builder) {
		if (this.logoutRequestMatcher != null) {
			return this.logoutRequestMatcher;
		}
		RequestMatcher pathMatcher = new SimplePathPatternRequestMatcher(this.logoutUrl);
		boolean csrfEnabled = builder.getConfigurer(SimpleCsrfConfigurer.class) != null;
		if (csrfEnabled) {
			// When CSRF is enabled, only POST is allowed (matches real Spring Security behavior)
			return request -> pathMatcher.matches(request) && "POST".equalsIgnoreCase(request.getMethod());
		}
		// When CSRF is disabled, accept any HTTP method
		return pathMatcher;
	}

}
```

### Modified Files

#### `[MODIFIED] simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

Changes:
1. Added `SimpleLogoutFilter` at position 250 in `FILTER_ORDER_MAP` (between `SimpleCsrfFilter` at 200 and `SimpleUsernamePasswordAuthenticationFilter` at 300)
2. Added import for `SimpleLogoutConfigurer`
3. Added `logout(Customizer<SimpleLogoutConfigurer>)` DSL method

### Test Files

#### `[NEW] simple-security-web/src/test/java/simple/security/web/authentication/logout/SimpleLogoutFilterTest.java`

```java
package simple.security.web.authentication.logout;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.util.matcher.SimplePathPatternRequestMatcher;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;

/**
 * Client-perspective tests for {@link SimpleLogoutFilter}.
 *
 * <p>These tests verify the logout flow from the perspective of a client
 * configuring and using the logout filter — the same way a Spring Security
 * user would experience it.
 */
class SimpleLogoutFilterTest {

	private MockHttpServletResponse response;

	private FilterChain chain;

	private List<String> handlerLog;

	@BeforeEach
	void setUp() {
		response = new MockHttpServletResponse();
		chain = mock(FilterChain.class);
		handlerLog = new ArrayList<>();
	}

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	// --- Matching behavior ---

	@Test
	void shouldInvokeHandlersAndSuccessHandler_WhenRequestMatchesLogoutUrl()
			throws IOException, ServletException {
		// Arrange — set up an authenticated user
		Authentication auth = authenticatedUser();
		SecurityContextHolder.getContext().setAuthentication(auth);

		SimpleLogoutFilter filter = createFilter("/logout", handlerLog);
		MockHttpServletRequest request = postRequest("/logout");

		// Act
		filter.doFilter(request, response, chain);

		// Assert — handlers ran, chain did NOT continue
		assertThat(handlerLog).containsExactly("handler1", "handler2");
		assertThat(response.getRedirectedUrl()).isEqualTo("/login?logout");
		verifyNoInteractions(chain);
	}

	@Test
	void shouldPassThrough_WhenRequestDoesNotMatchLogoutUrl()
			throws IOException, ServletException {
		// Arrange
		SimpleLogoutFilter filter = createFilter("/logout", handlerLog);
		MockHttpServletRequest request = getRequest("/home");

		// Act
		filter.doFilter(request, response, chain);

		// Assert — no handlers ran, chain continued
		assertThat(handlerLog).isEmpty();
		verify(chain).doFilter(request, response);
	}

	@Test
	void shouldStillLogout_WhenNoAuthenticationIsPresent()
			throws IOException, ServletException {
		// Arrange — no user logged in (SecurityContext is empty)
		SimpleLogoutFilter filter = createFilter("/logout", handlerLog);
		MockHttpServletRequest request = postRequest("/logout");

		// Act
		filter.doFilter(request, response, chain);

		// Assert — handlers still ran (with null authentication), redirect happened
		assertThat(handlerLog).containsExactly("handler1", "handler2");
		assertThat(response.getRedirectedUrl()).isEqualTo("/login?logout");
	}

	@Test
	void shouldPassAuthenticationToHandlers()
			throws IOException, ServletException {
		// Arrange
		Authentication auth = authenticatedUser();
		SecurityContextHolder.getContext().setAuthentication(auth);

		List<Authentication> capturedAuth = new ArrayList<>();
		LogoutHandler capturingHandler = (req, res, authentication) ->
				capturedAuth.add(authentication);

		SimpleLogoutFilter filter = new SimpleLogoutFilter(
				(req, res, authentication) -> res.sendRedirect("/done"),
				List.of(capturingHandler));
		filter.setLogoutRequestMatcher(new SimplePathPatternRequestMatcher("/logout"));
		MockHttpServletRequest request = postRequest("/logout");

		// Act
		filter.doFilter(request, response, chain);

		// Assert
		assertThat(capturedAuth).hasSize(1);
		assertThat(capturedAuth.get(0).getName()).isEqualTo("user");
	}

	// --- Helper methods ---

	private SimpleLogoutFilter createFilter(String logoutUrl, List<String> log) {
		LogoutHandler handler1 = (req, res, auth) -> log.add("handler1");
		LogoutHandler handler2 = (req, res, auth) -> log.add("handler2");
		LogoutSuccessHandler successHandler = (req, res, auth) ->
				res.sendRedirect("/login?logout");

		SimpleLogoutFilter filter = new SimpleLogoutFilter(successHandler, List.of(handler1, handler2));
		filter.setLogoutRequestMatcher(new SimplePathPatternRequestMatcher(logoutUrl));
		return filter;
	}

	private MockHttpServletRequest postRequest(String path) {
		MockHttpServletRequest request = new MockHttpServletRequest("POST", path);
		request.setServletPath(path);
		return request;
	}

	private MockHttpServletRequest getRequest(String path) {
		MockHttpServletRequest request = new MockHttpServletRequest("GET", path);
		request.setServletPath(path);
		return request;
	}

	private Authentication authenticatedUser() {
		return UsernamePasswordAuthenticationToken.authenticated(
				"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
	}

}
```

#### `[NEW] simple-security-web/src/test/java/simple/security/web/authentication/logout/SimpleSecurityContextLogoutHandlerTest.java`

```java
package simple.security.web.authentication.logout;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.context.SimpleHttpSessionSecurityContextRepository;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link SimpleSecurityContextLogoutHandler}.
 */
class SimpleSecurityContextLogoutHandlerTest {

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	@Test
	void shouldClearSecurityContextAndInvalidateSession() {
		// Arrange — authenticated user with an active session
		Authentication auth = authenticatedUser();
		SecurityContextHolder.getContext().setAuthentication(auth);

		MockHttpServletRequest request = new MockHttpServletRequest();
		request.getSession(true); // Create session
		MockHttpServletResponse response = new MockHttpServletResponse();

		SimpleSecurityContextLogoutHandler handler = new SimpleSecurityContextLogoutHandler();

		// Act
		handler.logout(request, response, auth);

		// Assert — context cleared, session invalidated
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
		assertThat(request.getSession(false)).isNull(); // Session was invalidated
	}

	@Test
	void shouldNotInvalidateSession_WhenDisabled() {
		// Arrange
		Authentication auth = authenticatedUser();
		SecurityContextHolder.getContext().setAuthentication(auth);

		MockHttpServletRequest request = new MockHttpServletRequest();
		request.getSession(true);
		MockHttpServletResponse response = new MockHttpServletResponse();

		SimpleSecurityContextLogoutHandler handler = new SimpleSecurityContextLogoutHandler();
		handler.setInvalidateHttpSession(false);

		// Act
		handler.logout(request, response, auth);

		// Assert — context cleared but session still alive
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
		assertThat(request.getSession(false)).isNotNull();
	}

	@Test
	void shouldHandleNoExistingSession() {
		// Arrange — no session created yet
		Authentication auth = authenticatedUser();
		SecurityContextHolder.getContext().setAuthentication(auth);

		MockHttpServletRequest request = new MockHttpServletRequest();
		MockHttpServletResponse response = new MockHttpServletResponse();

		SimpleSecurityContextLogoutHandler handler = new SimpleSecurityContextLogoutHandler();

		// Act — should not throw
		handler.logout(request, response, auth);

		// Assert — context still cleared
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldSaveEmptyContextToRepository() {
		// Arrange
		Authentication auth = authenticatedUser();
		SecurityContextHolder.getContext().setAuthentication(auth);

		MockHttpServletRequest request = new MockHttpServletRequest();
		request.getSession(true);
		MockHttpServletResponse response = new MockHttpServletResponse();

		SimpleHttpSessionSecurityContextRepository repository =
				new SimpleHttpSessionSecurityContextRepository();
		// First, save a context so the repository has something to clear
		repository.saveContext(SecurityContextHolder.getContext(), request, response);
		assertThat(repository.containsContext(request)).isTrue();

		SimpleSecurityContextLogoutHandler handler = new SimpleSecurityContextLogoutHandler();
		handler.setSecurityContextRepository(repository);
		handler.setInvalidateHttpSession(false); // Don't invalidate so we can check repository

		// Act
		handler.logout(request, response, auth);

		// Assert — repository no longer contains a context (empty context was saved)
		assertThat(repository.containsContext(request)).isFalse();
	}

	@Test
	void shouldHandleNullAuthentication() {
		// Arrange — logout when nobody is authenticated
		MockHttpServletRequest request = new MockHttpServletRequest();
		MockHttpServletResponse response = new MockHttpServletResponse();

		SimpleSecurityContextLogoutHandler handler = new SimpleSecurityContextLogoutHandler();

		// Act — should not throw
		handler.logout(request, response, null);

		// Assert
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	private Authentication authenticatedUser() {
		return UsernamePasswordAuthenticationToken.authenticated(
				"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
	}

}
```

#### `[NEW] simple-security-web/src/test/java/simple/security/web/authentication/logout/SimpleCookieClearingLogoutHandlerTest.java`

```java
package simple.security.web.authentication.logout;

import jakarta.servlet.http.Cookie;

import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for {@link SimpleCookieClearingLogoutHandler}.
 */
class SimpleCookieClearingLogoutHandlerTest {

	@Test
	void shouldClearSpecifiedCookies() {
		// Arrange
		MockHttpServletRequest request = new MockHttpServletRequest();
		MockHttpServletResponse response = new MockHttpServletResponse();
		SimpleCookieClearingLogoutHandler handler =
				new SimpleCookieClearingLogoutHandler("JSESSIONID", "remember-me");

		// Act
		handler.logout(request, response, null);

		// Assert
		Cookie[] cookies = response.getCookies();
		assertThat(cookies).hasSize(2);

		assertThat(cookies[0].getName()).isEqualTo("JSESSIONID");
		assertThat(cookies[0].getMaxAge()).isZero();
		assertThat(cookies[0].getPath()).isEqualTo("/");

		assertThat(cookies[1].getName()).isEqualTo("remember-me");
		assertThat(cookies[1].getMaxAge()).isZero();
	}

	@Test
	void shouldUseContextPathAsCookiePath() {
		// Arrange
		MockHttpServletRequest request = new MockHttpServletRequest();
		request.setContextPath("/myapp");
		MockHttpServletResponse response = new MockHttpServletResponse();
		SimpleCookieClearingLogoutHandler handler =
				new SimpleCookieClearingLogoutHandler("JSESSIONID");

		// Act
		handler.logout(request, response, null);

		// Assert
		Cookie cookie = response.getCookies()[0];
		assertThat(cookie.getPath()).isEqualTo("/myapp");
	}

	@Test
	void shouldRejectEmptyCookieNames() {
		assertThatThrownBy(() -> new SimpleCookieClearingLogoutHandler())
				.isInstanceOf(IllegalArgumentException.class);
	}

}
```

#### `[NEW] simple-security-config/src/test/java/simple/security/config/LogoutApiTest.java`

```java
package simple.security.config;

import java.util.ArrayList;
import java.util.Base64;
import java.util.List;

import jakarta.servlet.http.Cookie;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockFilterChain;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.mock.web.MockHttpSession;

import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.config.Customizer;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.SecurityFilterChain;
import simple.security.web.SimpleFilterChainProxy;
import simple.security.web.context.SimpleHttpSessionSecurityContextRepository;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for Logout (Feature 12).
 *
 * <p>These tests verify the full logout flow as a client would experience it:
 * configuring logout via the DSL, sending requests through the filter chain,
 * and checking that sessions are invalidated and redirects happen.
 */
class LogoutApiTest {

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	// ========== Default logout behavior ==========

	@Test
	void shouldLogoutWithDefaults_PostLogout() throws Exception {
		// Arrange — build a chain with logout defaults
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.httpBasic(Customizer.withDefaults())
				.logout(Customizer.withDefaults())
				.authenticationManager(acceptingAuthManager()));

		// Authenticate first to establish a session
		MockHttpServletRequest authRequest = new MockHttpServletRequest("GET", "/resource");
		authRequest.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse authResponse = new MockHttpServletResponse();
		proxy.doFilter(authRequest, authResponse, new MockFilterChain());
		MockHttpSession session = (MockHttpSession) authRequest.getSession(false);
		assertThat(session).isNotNull();

		// Act — POST /logout with the session
		MockHttpServletRequest logoutRequest = new MockHttpServletRequest("POST", "/logout");
		logoutRequest.setSession(session);
		MockHttpServletResponse logoutResponse = new MockHttpServletResponse();
		proxy.doFilter(logoutRequest, logoutResponse, new MockFilterChain());

		// Assert — redirected to /login?logout, session invalidated, context cleared
		assertThat(logoutResponse.getRedirectedUrl()).isEqualTo("/login?logout");
		assertThat(logoutRequest.getSession(false)).isNull(); // Session invalidated
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldPassThrough_WhenUrlDoesNotMatchLogout() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.logout(Customizer.withDefaults())
				.authenticationManager(acceptingAuthManager()));

		// Act — request a normal URL, not /logout
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/home");
		MockHttpServletResponse response = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();
		proxy.doFilter(request, response, chain);

		// Assert — request passed through
		assertThat(chain.getRequest()).isNotNull();
		assertThat(response.getRedirectedUrl()).isNull();
	}

	// ========== Custom logout URL and success URL ==========

	@Test
	void shouldUseCustomLogoutUrlAndSuccessUrl() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.logout(logout -> logout
						.logoutUrl("/signout")
						.logoutSuccessUrl("/goodbye"))
				.csrf(csrf -> csrf.disable()) // Disable CSRF so GET works
				.authenticationManager(acceptingAuthManager()));

		// Act — GET /signout (allowed since CSRF is disabled)
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/signout");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Assert — redirected to custom success URL
		assertThat(response.getRedirectedUrl()).isEqualTo("/goodbye");
	}

	// ========== Custom success handler ==========

	@Test
	void shouldUseCustomLogoutSuccessHandler() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.logout(logout -> logout
						.logoutSuccessHandler((request, response, auth) -> {
							response.setStatus(200);
							response.getWriter().write("Logged out");
						}))
				.csrf(csrf -> csrf.disable())
				.authenticationManager(acceptingAuthManager()));

		// Act
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Assert — custom handler was invoked
		assertThat(response.getStatus()).isEqualTo(200);
		assertThat(response.getContentAsString()).isEqualTo("Logged out");
	}

	// ========== Cookie clearing ==========

	@Test
	void shouldDeleteCookiesOnLogout() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.logout(logout -> logout
						.deleteCookies("JSESSIONID", "remember-me"))
				.csrf(csrf -> csrf.disable())
				.authenticationManager(acceptingAuthManager()));

		// Act
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Assert — cookies cleared (maxAge=0)
		Cookie[] cookies = response.getCookies();
		assertThat(cookies).hasSize(2);
		assertThat(cookies[0].getName()).isEqualTo("JSESSIONID");
		assertThat(cookies[0].getMaxAge()).isZero();
		assertThat(cookies[1].getName()).isEqualTo("remember-me");
		assertThat(cookies[1].getMaxAge()).isZero();
	}

	// ========== Custom logout handler ==========

	@Test
	void shouldInvokeCustomLogoutHandler() throws Exception {
		List<String> auditLog = new ArrayList<>();

		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.logout(logout -> logout
						.addLogoutHandler((req, res, auth) ->
								auditLog.add("User logged out: " + (auth != null ? auth.getName() : "anonymous"))))
				.csrf(csrf -> csrf.disable())
				.authenticationManager(acceptingAuthManager()));

		// Set up authentication
		SecurityContextHolder.getContext().setAuthentication(
				UsernamePasswordAuthenticationToken.authenticated(
						"admin", null, List.of(new SimpleGrantedAuthority("ROLE_ADMIN"))));

		// Act
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Assert — custom handler ran with the authentication
		assertThat(auditLog).containsExactly("User logged out: admin");
	}

	// ========== Session invalidation control ==========

	@Test
	void shouldNotInvalidateSession_WhenDisabled() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.logout(logout -> logout.invalidateHttpSession(false))
				.csrf(csrf -> csrf.disable())
				.authenticationManager(acceptingAuthManager()));

		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/logout");
		MockHttpSession session = new MockHttpSession();
		request.setSession(session);
		MockHttpServletResponse response = new MockHttpServletResponse();

		// Act
		proxy.doFilter(request, response, new MockFilterChain());

		// Assert — session is still valid
		assertThat(session.isInvalid()).isFalse();
	}

	// ========== CSRF interaction ==========

	@Test
	void shouldRequirePost_WhenCsrfIsEnabled() throws Exception {
		// Default: CSRF is NOT added, so any method works.
		// When CSRF configurer is present, only POST triggers logout.
		SimpleFilterChainProxy proxy = buildProxy(http -> http
				.sessionManagement(Customizer.withDefaults())
				.csrf(Customizer.withDefaults())
				.logout(Customizer.withDefaults())
				.authenticationManager(acceptingAuthManager()));

		// GET /logout should pass through (not trigger logout) when CSRF is enabled
		MockHttpServletRequest getRequest = new MockHttpServletRequest("GET", "/logout");
		MockHttpServletResponse getResponse = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();
		proxy.doFilter(getRequest, getResponse, chain);

		assertThat(getResponse.getRedirectedUrl()).isNull();
		assertThat(chain.getRequest()).isNotNull(); // Chain continued
	}

	// ========== Helpers ==========

	@FunctionalInterface
	interface HttpSecurityCustomizer {

		void customize(SimpleHttpSecurity http) throws Exception;

	}

	private SimpleFilterChainProxy buildProxy(HttpSecurityCustomizer customizer) throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		customizer.customize(http);
		SecurityFilterChain chain = http.build();
		return new SimpleFilterChainProxy(List.of(chain));
	}

	private simple.security.authentication.AuthenticationManager acceptingAuthManager() {
		return auth -> {
			UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) auth;
			if ("user".equals(token.getName()) && "password".equals(token.getCredentials())) {
				return UsernamePasswordAuthenticationToken.authenticated(
						"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
			}
			throw new simple.security.authentication.BadCredentialsException("Bad credentials");
		};
	}

	private String basicAuth(String username, String password) {
		return "Basic " + Base64.getEncoder().encodeToString((username + ":" + password).getBytes());
	}

}
```
