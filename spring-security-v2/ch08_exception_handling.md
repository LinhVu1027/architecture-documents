# Chapter 8: Exception Handling

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure exception handling via the DSL:
SimpleHttpSecurity http = new SimpleHttpSecurity();
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((request, response, authException) ->
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Please log in"))
    .accessDeniedHandler((request, response, accessDeniedException) ->
        response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied")));
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());

SecurityFilterChain chain = http.build();

// An ExceptionTranslationFilter is added at position 500 in the filter chain.
// It wraps chain.doFilter() in a try-catch and handles:
//   1. AuthenticationException → delegates to AuthenticationEntryPoint (401/redirect)
//   2. AccessDeniedException + no Authentication → delegates to AuthenticationEntryPoint
//   3. AccessDeniedException + authenticated user → delegates to AccessDeniedHandler (403)
//   4. Other exceptions → re-thrown unchanged
```

Questions to guide your thinking:
- Why does the exception translation filter sit at position 500 (BEFORE the authorization filter at 600) even though it handles exceptions FROM the authorization filter?
- Why is `AccessDeniedException` for an unauthenticated user treated as an authentication issue rather than an authorization issue?
- How does the configurer discover the entry point registered by formLogin or httpBasic without a direct dependency on those configurers?
- Why are `AuthenticationEntryPoint` and `AccessDeniedHandler` both `@FunctionalInterface`? Where does this design pay off?

---

## The API Contract

Feature 8 adds exception handling — the ability to translate security exceptions thrown by
downstream filters into appropriate HTTP responses. Without this filter, an `AccessDeniedException`
from `SimpleAuthorizationFilter` would propagate as a raw servlet exception (500 Internal Server Error).
With it, unauthenticated users get redirected to login or receive a 401, and unauthorized users get a 403.

### The DSL Entry Point

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity exceptionHandling(
        Customizer<SimpleExceptionHandlingConfigurer> exceptionHandlingCustomizer) {
    exceptionHandlingCustomizer.customize(getOrApply(new SimpleExceptionHandlingConfigurer()));
    return this;
}
```

### What the client configures

| DSL Method | Purpose | Default |
|-----------|---------|---------|
| `authenticationEntryPoint(...)` | Handle unauthenticated access | Shared object from formLogin/httpBasic, or `SimpleHttp403ForbiddenEntryPoint` |
| `accessDeniedHandler(...)` | Handle authenticated-but-forbidden access | `SimpleAccessDeniedHandlerImpl` (sends 403) |

### The behavioral contract

1. **`AuthenticationException`** from downstream → clear SecurityContext, delegate to `AuthenticationEntryPoint`
2. **`AccessDeniedException`** with no `Authentication` in SecurityContext → treat as authentication issue → delegate to `AuthenticationEntryPoint`
3. **`AccessDeniedException`** with `Authentication` present → delegate to `AccessDeniedHandler`
4. **All other exceptions** → re-thrown unchanged

---

## Client Usage & Tests

### Test 1: Custom entry point for unauthenticated users

```java
@Test
void shouldUseCustomEntryPointForUnauthenticatedUser() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, authException) ->
                        response.sendError(401, "Please log in")));
        http.authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated());
    });

    // No authentication set → should trigger entry point
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/dashboard");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(401);
    assertThat(response.getErrorMessage()).isEqualTo("Please log in");
}
```

### Test 2: Custom access denied handler for forbidden users

```java
@Test
void shouldUseCustomAccessDeniedHandlerForAuthorizedButForbiddenUser() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.exceptionHandling(ex -> ex
                .accessDeniedHandler((request, response, accessDeniedException) ->
                        response.sendError(403, "You are not allowed here")));
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated());
    });

    authenticateAs("user", "ROLE_USER");

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/admin/users");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(403);
    assertThat(response.getErrorMessage()).isEqualTo("You are not allowed here");
}
```

### Test 3: Integration with formLogin entry point

```java
@Test
void shouldUseFormLoginEntryPointWhenFormLoginConfigured() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(Customizer.withDefaults());
        http.exceptionHandling(Customizer.withDefaults());
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/login").permitAll()
                .anyRequest().authenticated());
    });

    // Unauthenticated request to protected resource → should redirect to /login
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/protected");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getRedirectedUrl()).isEqualTo("/login");
}
```

### Test 4: Explicit entry point overrides formLogin

```java
@Test
void shouldPreferExplicitEntryPointOverFormLogin() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(Customizer.withDefaults());
        http.exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, authException) ->
                        response.sendError(401, "Custom entry point wins")));
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/login").permitAll()
                .anyRequest().authenticated());
    });

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/protected");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(401);
    assertThat(response.getErrorMessage()).isEqualTo("Custom entry point wins");
}
```

### Test 5: Filter ordering

```java
@Test
void shouldPlaceExceptionTranslationFilterBeforeAuthorizationFilter() throws Exception {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.exceptionHandling(Customizer.withDefaults());
    http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

    SecurityFilterChain chain = http.build();

    assertThat(chain.getFilters()).hasSize(2);
    assertThat(chain.getFilters().get(0)).isInstanceOf(SimpleExceptionTranslationFilter.class);
    assertThat(chain.getFilters().get(1)).isInstanceOf(SimpleAuthorizationFilter.class);
}
```

---

## Implementing the Call Chain

### How exceptions flow through the filter chain

```
Request →  [SecurityContextHolder] → [CSRF] → [FormLogin] → [BasicAuth]
                                                                   ↓
                                                     [ExceptionTranslation] (pos 500)
                                                           ↓ try {
                                                     [Authorization] (pos 600)
                                                           ↓ throws AccessDeniedException
                                                     [ExceptionTranslation] catches it
                                                           ↓
                                              ┌────────────┴────────────┐
                                              ↓                         ↓
                                     No Authentication?         Has Authentication?
                                              ↓                         ↓
                                   AuthenticationEntryPoint    AccessDeniedHandler
                                     (401 / redirect)              (403)
```

### Layer 1: Handler Interface — `AccessDeniedHandler`

The `AuthenticationEntryPoint` interface already exists from Feature 6. We need its counterpart
for the "authenticated but forbidden" case.

```java
// simple/security/web/access/AccessDeniedHandler.java
@FunctionalInterface
public interface AccessDeniedHandler {

    void handle(HttpServletRequest request, HttpServletResponse response,
            AccessDeniedException accessDeniedException) throws IOException, ServletException;
}
```

**Design parallel**: Both `AuthenticationEntryPoint` and `AccessDeniedHandler` are single-method
functional interfaces. They represent the two branches of the exception translation decision tree:
"who are you?" vs. "you can't do that."

### Layer 2: Default Implementations

```java
// simple/security/web/access/SimpleAccessDeniedHandlerImpl.java
public class SimpleAccessDeniedHandlerImpl implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
            AccessDeniedException accessDeniedException) throws IOException, ServletException {
        if (!response.isCommitted()) {
            response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access Denied");
        }
    }
}
```

```java
// simple/security/web/SimpleHttp403ForbiddenEntryPoint.java
public class SimpleHttp403ForbiddenEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException, ServletException {
        response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access Denied");
    }
}
```

The `isCommitted()` check in `SimpleAccessDeniedHandlerImpl` prevents writing to a response that
has already been flushed to the client. This is important because filters earlier in the chain
may have already started writing response data.

### Layer 3: The Core Filter — `SimpleExceptionTranslationFilter`

This is the heart of Feature 8. It wraps the downstream filter chain in a try-catch:

```java
// simple/security/web/access/SimpleExceptionTranslationFilter.java
public class SimpleExceptionTranslationFilter implements Filter {

    private final AuthenticationEntryPoint authenticationEntryPoint;
    private AccessDeniedHandler accessDeniedHandler = new SimpleAccessDeniedHandlerImpl();

    public SimpleExceptionTranslationFilter(AuthenticationEntryPoint authenticationEntryPoint) {
        if (authenticationEntryPoint == null) {
            throw new IllegalArgumentException("authenticationEntryPoint cannot be null");
        }
        this.authenticationEntryPoint = authenticationEntryPoint;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);
    }

    private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        try {
            chain.doFilter(request, response);
        }
        catch (AuthenticationException ex) {
            handleAuthenticationException(request, response, ex);
        }
        catch (AccessDeniedException ex) {
            handleAccessDeniedException(request, response, ex);
        }
    }
```

The two exception handlers implement the routing logic:

```java
    private void handleAuthenticationException(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException exception) throws IOException, ServletException {
        // Clear any stale authentication before commencing a new login
        SecurityContextHolder.clearContext();
        this.authenticationEntryPoint.commence(request, response, exception);
    }

    private void handleAccessDeniedException(HttpServletRequest request, HttpServletResponse response,
            AccessDeniedException exception) throws IOException, ServletException {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null) {
            // No authentication → treat as authentication issue
            AuthenticationException authException = new InsufficientAuthenticationException(
                    "Full authentication is required to access this resource", exception);
            handleAuthenticationException(request, response, authException);
        }
        else {
            // Authenticated but not authorized → 403
            this.accessDeniedHandler.handle(request, response, exception);
        }
    }
```

**Why clear the SecurityContext on `AuthenticationException`?** If the current session held a stale
or invalid authentication token, keeping it around while the entry point commences could cause
confusion. Clearing it ensures a clean slate for the new authentication attempt.

**Why wrap `AccessDeniedException` in `InsufficientAuthenticationException`?** The entry point
expects an `AuthenticationException`. By wrapping the original `AccessDeniedException`, we
preserve the cause chain while satisfying the type contract. The original exception is accessible
via `getCause()`.

### Layer 4: The Configurer — `SimpleExceptionHandlingConfigurer`

The configurer bridges the DSL to the filter:

```java
// simple/security/config/annotation/web/configurers/SimpleExceptionHandlingConfigurer.java
public class SimpleExceptionHandlingConfigurer
        extends SimpleAbstractHttpConfigurer<SimpleExceptionHandlingConfigurer> {

    private AuthenticationEntryPoint authenticationEntryPoint;
    private AccessDeniedHandler accessDeniedHandler;

    @Override
    public void configure(SimpleHttpSecurity builder) throws Exception {
        AuthenticationEntryPoint entryPoint = resolveEntryPoint(builder);
        SimpleExceptionTranslationFilter filter = new SimpleExceptionTranslationFilter(entryPoint);

        AccessDeniedHandler deniedHandler = resolveAccessDeniedHandler();
        filter.setAccessDeniedHandler(deniedHandler);

        builder.addFilter(filter);
    }
```

The entry point resolution has a **three-level fallback chain**:

```java
    private AuthenticationEntryPoint resolveEntryPoint(SimpleHttpSecurity builder) {
        // Priority 1: Explicitly configured via DSL
        if (this.authenticationEntryPoint != null) {
            return this.authenticationEntryPoint;
        }
        // Priority 2: Shared object from formLogin/httpBasic
        AuthenticationEntryPoint sharedEntryPoint =
                builder.getSharedObject(AuthenticationEntryPoint.class);
        if (sharedEntryPoint != null) {
            return sharedEntryPoint;
        }
        // Priority 3: Default fallback
        return new SimpleHttp403ForbiddenEntryPoint();
    }
```

**How shared objects enable loose coupling**: `SimpleFormLoginConfigurer.init()` registers a
`SimpleLoginUrlAuthenticationEntryPoint` as a shared object. `SimpleExceptionHandlingConfigurer.configure()`
reads it. Neither configurer imports or references the other — the `SimpleHttpSecurity` shared objects
map acts as a mediator. This mirrors the real Spring Security's shared object pattern.

---

## Try It Yourself

1. **JSON error body**: Create a custom `AuthenticationEntryPoint` that returns a JSON body
   `{"error": "unauthorized", "message": "..."}` with Content-Type `application/json`.

2. **Request cache**: The real framework saves the original request before redirecting to login,
   so it can replay the request after successful authentication. Try adding a simple request
   cache to `SimpleExceptionTranslationFilter`.

3. **Cause chain analysis**: The real framework uses `ThrowableAnalyzer` to walk the exception
   cause chain looking for security exceptions wrapped in `ServletException`. Try extending the
   filter to handle `ServletException` wrapping an `AccessDeniedException`.

---

## Why This Works

### Exception flow as call-stack unwinding

The filter chain is implemented as nested `doFilter()` calls:

```
ExceptionTranslation.doFilter()     ← catch block here
    └→ chain.doFilter()
        └→ AuthorizationFilter.doFilter()   ← throw here
```

When `AuthorizationFilter` throws, Java's exception mechanism unwinds the call stack, popping
frames until it finds a matching catch block — which is in `ExceptionTranslation.doFilter()`.
This is why position ordering matters: the exception filter must be an *outer* frame (lower
position number = called first = outer frame).

### Strategy pattern for extensibility

Both `AuthenticationEntryPoint` and `AccessDeniedHandler` are strategy interfaces. The filter
defines the *algorithm* (catch → classify → route), while the strategies define the *behavior*
(what HTTP response to send). This means the same filter works for:
- Form login (redirect to `/login`)
- HTTP Basic (send `WWW-Authenticate` header)
- REST APIs (send JSON error body)
- Custom scenarios (anything you can express in a lambda)

### Shared objects as a service locator

The three-level entry point resolution in the configurer (`explicit > shared > default`) solves
a real ordering problem: the exception handling configurer doesn't know which authentication
mechanisms are configured, but it needs their entry points. Shared objects provide late binding —
the entry point is resolved during `configure()`, after all configurers have run `init()`.

---

## What We Enhanced

| Layer | Before Feature 8 | After Feature 8 |
|-------|-------------------|-----------------|
| `SimpleHttpSecurity` FILTER_ORDER_MAP | `"ExceptionTranslationFilter"` reserved at 500 | Renamed to `"SimpleExceptionTranslationFilter"` — now actively used |
| `SimpleHttpSecurity` DSL | `formLogin()`, `httpBasic()`, `authorizeHttpRequests()` | + `exceptionHandling()` |
| Exception behavior | `AccessDeniedException` from AuthorizationFilter propagated as raw servlet error (500) | Caught and translated to 401 (entry point) or 403 (access denied handler) |
| `AuthenticationEntryPoint` shared object | Set by formLogin/httpBasic but unused | Now consumed by `SimpleExceptionHandlingConfigurer` to provide proper entry point |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplifications |
|------------------|----------------------|---------------------|
| `SimpleExceptionTranslationFilter` | `ExceptionTranslationFilter` | No `RequestCache` (no saving original request), no `ThrowableAnalyzer` (no cause chain walking), no anonymous/remember-me distinction (checks null only) |
| `AccessDeniedHandler` | `AccessDeniedHandler` | Identical — single-method functional interface |
| `SimpleAccessDeniedHandlerImpl` | `AccessDeniedHandlerImpl` | No error page forwarding |
| `SimpleHttp403ForbiddenEntryPoint` | `Http403ForbiddenEntryPoint` | Identical behavior |
| `SimpleExceptionHandlingConfigurer` | `ExceptionHandlingConfigurer` | No `defaultAuthenticationEntryPointFor` (request-matcher-based delegation), no `defaultAccessDeniedHandlerFor`, no `DelegatingAuthenticationEntryPoint` |

**Real source files** (Spring Security 6.x):
- `web/src/main/java/org/springframework/security/web/access/ExceptionTranslationFilter.java`
- `web/src/main/java/org/springframework/security/web/AuthenticationEntryPoint.java`
- `web/src/main/java/org/springframework/security/web/access/AccessDeniedHandler.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/ExceptionHandlingConfigurer.java`

---

## Complete Code

### New Files

#### `simple-security-web/.../web/access/AccessDeniedHandler.java` [NEW]

```java
package simple.security.web.access;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.access.AccessDeniedException;

/**
 * Strategy interface for handling {@link AccessDeniedException}.
 *
 * <p>Called by {@link SimpleExceptionTranslationFilter} when a fully authenticated
 * user attempts to access a resource they are not authorized for. Implementations
 * decide HOW to respond — for example, sending a 403 status, rendering a custom
 * error page, or returning a JSON error body.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.access.AccessDeniedHandler}.
 *
 * Mirrors: org.springframework.security.web.access.AccessDeniedHandler
 * Source:  web/src/main/java/org/springframework/security/web/access/AccessDeniedHandler.java
 */
@FunctionalInterface
public interface AccessDeniedHandler {

	/**
	 * Handles an access denied failure.
	 *
	 * @param request               the request that resulted in an AccessDeniedException
	 * @param response              the response to modify (set status, write body, etc.)
	 * @param accessDeniedException the exception that triggered the handler
	 */
	void handle(HttpServletRequest request, HttpServletResponse response,
			AccessDeniedException accessDeniedException) throws IOException, ServletException;

}
```

#### `simple-security-web/.../web/access/SimpleAccessDeniedHandlerImpl.java` [NEW]

```java
package simple.security.web.access;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.access.AccessDeniedException;

/**
 * Default implementation of {@link AccessDeniedHandler} that sends an
 * HTTP 403 Forbidden error response.
 *
 * <p>Simplifications vs. real {@code AccessDeniedHandlerImpl}:
 * <ul>
 *   <li>No error page forwarding — always sends a 403 error</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.web.access.AccessDeniedHandlerImpl
 * Source:  web/src/main/java/org/springframework/security/web/access/AccessDeniedHandlerImpl.java
 */
public class SimpleAccessDeniedHandlerImpl implements AccessDeniedHandler {

	@Override
	public void handle(HttpServletRequest request, HttpServletResponse response,
			AccessDeniedException accessDeniedException) throws IOException, ServletException {
		if (!response.isCommitted()) {
			response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access Denied");
		}
	}

}
```

#### `simple-security-web/.../web/SimpleHttp403ForbiddenEntryPoint.java` [NEW]

```java
package simple.security.web;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;

/**
 * An {@link AuthenticationEntryPoint} that sends an HTTP 403 Forbidden error.
 *
 * <p>Used as the fallback entry point when no authentication mechanism (form login,
 * HTTP Basic, etc.) has registered a more specific entry point. This signals to the
 * client that authentication is required but the server has no way to commence it.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.Http403ForbiddenEntryPoint}.
 *
 * Mirrors: org.springframework.security.web.authentication.Http403ForbiddenEntryPoint
 * Source:  web/src/main/java/org/springframework/security/web/authentication/Http403ForbiddenEntryPoint.java
 */
public class SimpleHttp403ForbiddenEntryPoint implements AuthenticationEntryPoint {

	@Override
	public void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException {
		response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access Denied");
	}

}
```

#### `simple-security-web/.../web/access/SimpleExceptionTranslationFilter.java` [NEW]

```java
package simple.security.web.access;

import java.io.IOException;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.access.AccessDeniedException;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.AuthenticationEntryPoint;

/**
 * A servlet filter that catches security exceptions thrown by downstream filters
 * and translates them into appropriate HTTP responses.
 *
 * <p>This filter wraps {@code chain.doFilter()} in a try-catch and handles two
 * types of exceptions:
 * <ul>
 *   <li>{@link AuthenticationException} — the user is not authenticated.
 *       Delegates to the configured {@link AuthenticationEntryPoint} (e.g.,
 *       redirect to login page or send 401 with WWW-Authenticate header).</li>
 *   <li>{@link AccessDeniedException} — two sub-cases:
 *       <ul>
 *         <li>If no {@link Authentication} exists in the SecurityContext,
 *             this is treated as an authentication issue — delegates to the
 *             {@link AuthenticationEntryPoint}.</li>
 *         <li>If the user IS authenticated but lacks authorization, delegates
 *             to the {@link AccessDeniedHandler} (typically sends 403).</li>
 *       </ul>
 *   </li>
 * </ul>
 *
 * <p>Non-security exceptions are re-thrown unchanged.
 *
 * <h2>Position in the filter chain</h2>
 * This filter sits at position 500, BEFORE the {@code SimpleAuthorizationFilter} (600).
 * This is critical: the exception translation filter must be an outer frame on the call
 * stack relative to the filter that throws, so it can catch the exceptions as they
 * propagate up.
 *
 * <p>Simplifications vs. real {@code ExceptionTranslationFilter}:
 * <ul>
 *   <li>No {@code RequestCache} — does not save the original request for post-login redirect</li>
 *   <li>No {@code ThrowableAnalyzer} — catches exceptions directly instead of walking cause chains</li>
 *   <li>No anonymous/remember-me distinction — checks only for null Authentication</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.web.access.ExceptionTranslationFilter
 * Source:  web/src/main/java/org/springframework/security/web/access/ExceptionTranslationFilter.java
 */
public class SimpleExceptionTranslationFilter implements Filter {

	private final AuthenticationEntryPoint authenticationEntryPoint;

	private AccessDeniedHandler accessDeniedHandler = new SimpleAccessDeniedHandlerImpl();

	public SimpleExceptionTranslationFilter(AuthenticationEntryPoint authenticationEntryPoint) {
		if (authenticationEntryPoint == null) {
			throw new IllegalArgumentException("authenticationEntryPoint cannot be null");
		}
		this.authenticationEntryPoint = authenticationEntryPoint;
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);
	}

	private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		try {
			chain.doFilter(request, response);
		}
		catch (AuthenticationException ex) {
			handleAuthenticationException(request, response, ex);
		}
		catch (AccessDeniedException ex) {
			handleAccessDeniedException(request, response, ex);
		}
	}

	private void handleAuthenticationException(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException exception) throws IOException, ServletException {
		SecurityContextHolder.clearContext();
		this.authenticationEntryPoint.commence(request, response, exception);
	}

	private void handleAccessDeniedException(HttpServletRequest request, HttpServletResponse response,
			AccessDeniedException exception) throws IOException, ServletException {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		if (authentication == null) {
			// No authentication → treat as an authentication issue
			AuthenticationException authException = new InsufficientAuthenticationException(
					"Full authentication is required to access this resource", exception);
			handleAuthenticationException(request, response, authException);
		}
		else {
			// User is authenticated but not authorized → 403
			this.accessDeniedHandler.handle(request, response, exception);
		}
	}

	public void setAccessDeniedHandler(AccessDeniedHandler accessDeniedHandler) {
		if (accessDeniedHandler == null) {
			throw new IllegalArgumentException("accessDeniedHandler cannot be null");
		}
		this.accessDeniedHandler = accessDeniedHandler;
	}

	public AuthenticationEntryPoint getAuthenticationEntryPoint() {
		return this.authenticationEntryPoint;
	}

	/**
	 * Thrown when an access decision is made for a user that is not fully authenticated.
	 * This wraps an {@link AccessDeniedException} and allows the
	 * {@link AuthenticationEntryPoint} to commence authentication.
	 */
	public static class InsufficientAuthenticationException extends AuthenticationException {

		public InsufficientAuthenticationException(String msg, Throwable cause) {
			super(msg, cause);
		}

	}

}
```

#### `simple-security-config/.../configurers/SimpleExceptionHandlingConfigurer.java` [NEW]

```java
package simple.security.config.annotation.web.configurers;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.AuthenticationEntryPoint;
import simple.security.web.SimpleHttp403ForbiddenEntryPoint;
import simple.security.web.access.AccessDeniedHandler;
import simple.security.web.access.SimpleAccessDeniedHandlerImpl;
import simple.security.web.access.SimpleExceptionTranslationFilter;

/**
 * Configures exception handling via the {@link SimpleHttpSecurity} DSL.
 *
 * <h2>Usage</h2>
 * <pre>
 * // Defaults (uses entry point from formLogin/httpBasic, or Http403ForbiddenEntryPoint)
 * http.exceptionHandling(Customizer.withDefaults());
 *
 * // Custom handlers
 * http.exceptionHandling(ex -> ex
 *     .authenticationEntryPoint((request, response, authException) ->
 *         response.sendError(401, "Please log in"))
 *     .accessDeniedHandler((request, response, accessDeniedException) ->
 *         response.sendError(403, "Access denied")));
 * </pre>
 *
 * <h2>What it does</h2>
 * <p>During {@code configure()}, this configurer:
 * <ol>
 *   <li>Resolves the {@link AuthenticationEntryPoint}: uses the explicitly configured
 *       one, or falls back to the shared object (set by formLogin/httpBasic), or
 *       falls back to {@link SimpleHttp403ForbiddenEntryPoint}.</li>
 *   <li>Creates a {@link SimpleExceptionTranslationFilter} with the resolved entry point.</li>
 *   <li>Sets the {@link AccessDeniedHandler} (defaults to {@link SimpleAccessDeniedHandlerImpl}).</li>
 *   <li>Adds the filter at position 500 in the filter chain.</li>
 * </ol>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.web.configurers.ExceptionHandlingConfigurer}.
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.ExceptionHandlingConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/ExceptionHandlingConfigurer.java
 */
public class SimpleExceptionHandlingConfigurer
		extends SimpleAbstractHttpConfigurer<SimpleExceptionHandlingConfigurer> {

	private AuthenticationEntryPoint authenticationEntryPoint;

	private AccessDeniedHandler accessDeniedHandler;

	// === Lifecycle ===

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		AuthenticationEntryPoint entryPoint = resolveEntryPoint(builder);
		SimpleExceptionTranslationFilter filter = new SimpleExceptionTranslationFilter(entryPoint);

		AccessDeniedHandler deniedHandler = resolveAccessDeniedHandler();
		filter.setAccessDeniedHandler(deniedHandler);

		builder.addFilter(filter);
	}

	// === DSL methods ===

	/**
	 * Sets the {@link AuthenticationEntryPoint} to be used when an unauthenticated
	 * user tries to access a protected resource.
	 *
	 * <p>When set explicitly, this takes precedence over any entry point registered
	 * by other configurers (e.g., formLogin, httpBasic) via shared objects.
	 *
	 * @param authenticationEntryPoint the entry point to use
	 * @return this configurer for chaining
	 */
	public SimpleExceptionHandlingConfigurer authenticationEntryPoint(
			AuthenticationEntryPoint authenticationEntryPoint) {
		this.authenticationEntryPoint = authenticationEntryPoint;
		return this;
	}

	/**
	 * Sets the {@link AccessDeniedHandler} to be used when an authenticated user
	 * tries to access a resource they are not authorized for.
	 *
	 * @param accessDeniedHandler the handler to use
	 * @return this configurer for chaining
	 */
	public SimpleExceptionHandlingConfigurer accessDeniedHandler(AccessDeniedHandler accessDeniedHandler) {
		this.accessDeniedHandler = accessDeniedHandler;
		return this;
	}

	// === Internal resolution ===

	/**
	 * Resolves the entry point with this priority:
	 * <ol>
	 *   <li>Explicitly configured via {@link #authenticationEntryPoint}</li>
	 *   <li>Shared object registered by another configurer (e.g., formLogin registers
	 *       a {@code SimpleLoginUrlAuthenticationEntryPoint})</li>
	 *   <li>Fallback: {@link SimpleHttp403ForbiddenEntryPoint}</li>
	 * </ol>
	 */
	private AuthenticationEntryPoint resolveEntryPoint(SimpleHttpSecurity builder) {
		if (this.authenticationEntryPoint != null) {
			return this.authenticationEntryPoint;
		}
		AuthenticationEntryPoint sharedEntryPoint = builder.getSharedObject(AuthenticationEntryPoint.class);
		if (sharedEntryPoint != null) {
			return sharedEntryPoint;
		}
		return new SimpleHttp403ForbiddenEntryPoint();
	}

	private AccessDeniedHandler resolveAccessDeniedHandler() {
		if (this.accessDeniedHandler != null) {
			return this.accessDeniedHandler;
		}
		return new SimpleAccessDeniedHandlerImpl();
	}

}
```

### Modified Files

#### `simple-security-config/.../builders/SimpleHttpSecurity.java` [MODIFIED]

**Change 1**: Filter order map — renamed `"ExceptionTranslationFilter"` to `"SimpleExceptionTranslationFilter"`:
```java
FILTER_ORDER_MAP.put("SimpleExceptionTranslationFilter", order);  // was "ExceptionTranslationFilter"
```

**Change 2**: Added import:
```java
import simple.security.config.annotation.web.configurers.SimpleExceptionHandlingConfigurer;
```

**Change 3**: Added DSL method:
```java
public SimpleHttpSecurity exceptionHandling(
        Customizer<SimpleExceptionHandlingConfigurer> exceptionHandlingCustomizer) {
    exceptionHandlingCustomizer.customize(getOrApply(new SimpleExceptionHandlingConfigurer()));
    return this;
}
```
