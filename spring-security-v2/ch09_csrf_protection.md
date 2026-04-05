# Chapter 9: CSRF Protection

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure CSRF protection via the DSL:
SimpleHttpSecurity http = new SimpleHttpSecurity();
http.csrf(Customizer.withDefaults());
http.authorizeHttpRequests(auth -> auth
    .anyRequest().authenticated());

SecurityFilterChain chain = http.build();

// A SimpleCsrfFilter is added at position 200 in the filter chain.
// It intercepts every request and:
//   1. Loads a CSRF token from the repository (or generates one if missing)
//   2. Makes the token available as a request attribute
//   3. For GET/HEAD/TRACE/OPTIONS → passes through (no validation)
//   4. For POST/PUT/DELETE/PATCH → extracts the actual token from header or parameter
//   5. Compares expected vs. actual using constant-time comparison
//   6. On mismatch → delegates to AccessDeniedHandler (403)
//   7. On match → continues the filter chain
```

Questions to guide your thinking:
- Why does CSRF protection only validate state-changing methods (POST/PUT/DELETE) and not GET requests?
- Why does the token comparison use `MessageDigest.isEqual()` instead of `String.equals()`?
- What's the difference between session-based and cookie-based CSRF token storage, and when would you use each?
- Why must the cookie-based repository set `httpOnly=false` for SPA usage?
- How does the `ignoringRequestMatchers` feature compose with the default method-based matcher?

---

## The API Contract

Feature 9 adds CSRF (Cross-Site Request Forgery) protection — the ability to require a secret
token on state-changing requests that matches a server-stored value. Without CSRF protection,
an attacker's website could submit forged forms or AJAX requests to your application, exploiting
the user's active session.

### The DSL Entry Point

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity csrf(Customizer<SimpleCsrfConfigurer> csrfCustomizer) {
    csrfCustomizer.customize(getOrApply(new SimpleCsrfConfigurer()));
    return this;
}
```

### What the client configures

| DSL Method | Purpose | Default |
|-----------|---------|---------|
| `csrfTokenRepository(...)` | Where to store/load CSRF tokens | `SimpleHttpSessionCsrfTokenRepository` (session-based) |
| `ignoringRequestMatchers(...)` | Paths to skip CSRF validation | None (all state-changing requests validated) |
| `accessDeniedHandler(...)` | Handle CSRF validation failures | `SimpleAccessDeniedHandlerImpl` (sends 403) |
| `disable()` | Turn off CSRF entirely | Enabled by default |

### The behavioral contract

1. **Every request** → load or generate a CSRF token, save it, set it as a request attribute
2. **GET/HEAD/TRACE/OPTIONS** → pass through without validation
3. **POST/PUT/DELETE/PATCH** → extract token from `X-CSRF-TOKEN` header or `_csrf` parameter
4. **Token matches** → continue the filter chain
5. **Token missing or mismatched** → delegate to `AccessDeniedHandler` (403 Forbidden)

---

## Client Usage & Tests

### Test 1: GET requests pass through without CSRF token

```java
@Test
void shouldAllowGetRequestWithoutCsrfToken() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.csrf(Customizer.withDefaults());
    });

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/data");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(request, response, chain);

    // GET should pass through — no CSRF validation
    assertThat(chain.getRequest()).isNotNull();
    assertThat(response.getStatus()).isEqualTo(200);
}
```

### Test 2: POST without token is rejected

```java
@Test
void shouldRejectPostRequestWithoutCsrfToken() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.csrf(Customizer.withDefaults());
    });

    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/submit");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // POST without token → 403
    assertThat(response.getStatus()).isEqualTo(403);
}
```

### Test 3: POST with valid CSRF token passes through

```java
@Test
void shouldAllowPostRequestWithValidCsrfTokenInParameter() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.csrf(Customizer.withDefaults());
    });

    // Step 1: GET to obtain the CSRF token
    MockHttpServletRequest getRequest = new MockHttpServletRequest("GET", "/form");
    MockHttpServletResponse getResponse = new MockHttpServletResponse();
    proxy.doFilter(getRequest, getResponse, new MockFilterChain());

    CsrfToken token = (CsrfToken) getRequest.getAttribute(CsrfToken.class.getName());
    assertThat(token).isNotNull();

    // Step 2: POST with the token as a parameter
    MockHttpServletRequest postRequest = new MockHttpServletRequest("POST", "/submit");
    postRequest.setSession(getRequest.getSession()); // share session
    postRequest.setParameter(token.getParameterName(), token.getToken());
    MockHttpServletResponse postResponse = new MockHttpServletResponse();
    MockFilterChain postChain = new MockFilterChain();
    proxy.doFilter(postRequest, postResponse, postChain);

    // POST with valid token → should pass through
    assertThat(postChain.getRequest()).isNotNull();
    assertThat(postResponse.getStatus()).isEqualTo(200);
}
```

### Test 4: Cookie-based CSRF for SPAs

```java
@Test
void shouldUseCookieBasedCsrfRepository() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.csrf(csrf -> csrf
                .csrfTokenRepository(SimpleCookieCsrfTokenRepository.withHttpOnlyFalse()));
    });

    // GET → should set XSRF-TOKEN cookie
    MockHttpServletRequest getRequest = new MockHttpServletRequest("GET", "/form");
    MockHttpServletResponse getResponse = new MockHttpServletResponse();
    proxy.doFilter(getRequest, getResponse, new MockFilterChain());

    jakarta.servlet.http.Cookie csrfCookie = getResponse.getCookie("XSRF-TOKEN");
    assertThat(csrfCookie).isNotNull();
    assertThat(csrfCookie.getValue()).isNotEmpty();

    // POST with X-XSRF-TOKEN header → should pass
    MockHttpServletRequest postRequest = new MockHttpServletRequest("POST", "/submit");
    postRequest.setCookies(csrfCookie);
    postRequest.addHeader("X-XSRF-TOKEN", csrfCookie.getValue());
    MockHttpServletResponse postResponse = new MockHttpServletResponse();
    MockFilterChain postChain = new MockFilterChain();
    proxy.doFilter(postRequest, postResponse, postChain);

    assertThat(postChain.getRequest()).isNotNull();
    assertThat(postResponse.getStatus()).isEqualTo(200);
}
```

### Test 5: Ignoring specific paths

```java
@Test
void shouldIgnoreCsrfForSpecificPaths() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/webhooks/**"));
    });

    // POST to ignored path → should pass without token
    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/api/webhooks/github");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(request, response, chain);

    assertThat(chain.getRequest()).isNotNull();
    assertThat(response.getStatus()).isEqualTo(200);
}
```

### Test 6: Disable CSRF entirely

```java
@Test
void shouldDisableCsrfWhenRequested() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.csrf(csrf -> csrf.disable());
    });

    // POST without CSRF token → should pass because CSRF is disabled
    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/submit");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(request, response, chain);

    assertThat(chain.getRequest()).isNotNull();
    assertThat(response.getStatus()).isEqualTo(200);
}
```

---

## Implementing the Call Chain

### Layer 1: The Token Model (API surface)

The first thing we need is a way to represent a CSRF token. The token has three components:
- **Header name** — the HTTP header that can carry the token (e.g., `X-CSRF-TOKEN`)
- **Parameter name** — the HTTP parameter that can carry the token (e.g., `_csrf`)
- **Token value** — the secret random value

```java
// CsrfToken.java — the API contract
public interface CsrfToken extends Serializable {
    String getHeaderName();
    String getParameterName();
    String getToken();
}
```

And a simple implementation:

```java
// DefaultCsrfToken.java — holds the three values
public final class DefaultCsrfToken implements CsrfToken {
    private final String headerName;
    private final String parameterName;
    private final String token;

    public DefaultCsrfToken(String headerName, String parameterName, String token) {
        // validate non-null, non-empty
        this.headerName = headerName;
        this.parameterName = parameterName;
        this.token = token;
    }
    // getters...
}
```

### Layer 2: Token Storage (CsrfTokenRepository)

The repository interface defines how tokens are generated, saved, and loaded:

```java
public interface CsrfTokenRepository {
    CsrfToken generateToken(HttpServletRequest request);
    void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response);
    CsrfToken loadToken(HttpServletRequest request);
}
```

Two implementations:

**Session-based** (default, more secure):
```java
public final class SimpleHttpSessionCsrfTokenRepository implements CsrfTokenRepository {
    @Override
    public CsrfToken generateToken(HttpServletRequest request) {
        return new DefaultCsrfToken("X-CSRF-TOKEN", "_csrf", UUID.randomUUID().toString());
    }

    @Override
    public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
        if (token == null) {
            // Delete from session
            HttpSession session = request.getSession(false);
            if (session != null) session.removeAttribute(SESSION_ATTR);
        } else {
            request.getSession().setAttribute(SESSION_ATTR, token);
        }
    }

    @Override
    public CsrfToken loadToken(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        return (session != null) ? (CsrfToken) session.getAttribute(SESSION_ATTR) : null;
    }
}
```

**Cookie-based** (for SPAs):
```java
public final class SimpleCookieCsrfTokenRepository implements CsrfTokenRepository {
    // Uses "XSRF-TOKEN" cookie and "X-XSRF-TOKEN" header (AngularJS conventions)

    @Override
    public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
        Cookie cookie = new Cookie("XSRF-TOKEN", token != null ? token.getToken() : "");
        cookie.setMaxAge(token != null ? -1 : 0); // session cookie or expire
        cookie.setHttpOnly(this.cookieHttpOnly);
        response.addCookie(cookie);
    }

    public static SimpleCookieCsrfTokenRepository withHttpOnlyFalse() {
        // JavaScript needs to read the cookie → httpOnly must be false
        SimpleCookieCsrfTokenRepository repo = new SimpleCookieCsrfTokenRepository();
        repo.cookieHttpOnly = false;
        return repo;
    }
}
```

### Layer 3: The Filter (SimpleCsrfFilter)

The filter is the core — it intercepts every request:

```java
public final class SimpleCsrfFilter extends HttpFilter {
    private static final Set<String> ALLOWED_METHODS = Set.of("GET", "HEAD", "TRACE", "OPTIONS");

    @Override
    protected void doFilter(HttpServletRequest request, HttpServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        // 1. Load or generate token
        CsrfToken csrfToken = tokenRepository.loadToken(request);
        boolean missingToken = (csrfToken == null);
        if (missingToken) {
            csrfToken = tokenRepository.generateToken(request);
            tokenRepository.saveToken(csrfToken, request, response);
        }

        // 2. Make token available as request attribute
        request.setAttribute(CsrfToken.class.getName(), csrfToken);
        request.setAttribute(csrfToken.getParameterName(), csrfToken);

        // 3. Skip validation for safe methods
        if (!requireCsrfProtectionMatcher.matches(request)) {
            chain.doFilter(request, response);
            return;
        }

        // 4. Extract actual token from header or parameter
        String actualToken = request.getHeader(csrfToken.getHeaderName());
        if (actualToken == null) {
            actualToken = request.getParameter(csrfToken.getParameterName());
        }

        // 5. Constant-time comparison
        if (!equalsConstantTime(csrfToken.getToken(), actualToken)) {
            accessDeniedHandler.handle(request, response, new AccessDeniedException("..."));
            return;
        }

        // 6. Token valid → continue
        chain.doFilter(request, response);
    }
}
```

### Layer 4: The Configurer (SimpleCsrfConfigurer)

The configurer wires everything together and adds the filter to the chain:

```java
public class SimpleCsrfConfigurer extends SimpleAbstractHttpConfigurer<SimpleCsrfConfigurer> {
    private CsrfTokenRepository csrfTokenRepository = new SimpleHttpSessionCsrfTokenRepository();
    private List<RequestMatcher> ignoredCsrfProtectionMatchers = new ArrayList<>();

    @Override
    public void configure(SimpleHttpSecurity builder) throws Exception {
        SimpleCsrfFilter filter = new SimpleCsrfFilter(csrfTokenRepository);

        // Combine default matcher with ignore matchers
        if (!ignoredCsrfProtectionMatchers.isEmpty()) {
            filter.setRequireCsrfProtectionMatcher(request -> {
                if (isSafeMethod(request)) return false;
                for (RequestMatcher m : ignoredCsrfProtectionMatchers)
                    if (m.matches(request)) return false;
                return true;
            });
        }

        filter.setAccessDeniedHandler(resolveAccessDeniedHandler());
        builder.addFilter(filter);
    }

    // DSL methods: csrfTokenRepository(), ignoringRequestMatchers(), disable()
}
```

### Modification: SimpleHttpSecurity

Added the `csrf()` DSL method and updated the filter order map entry from `"CsrfFilter"` to
`"SimpleCsrfFilter"` to match the actual class name:

```java
// In SimpleHttpSecurity:
FILTER_ORDER_MAP.put("SimpleCsrfFilter", order); // was "CsrfFilter"

public SimpleHttpSecurity csrf(Customizer<SimpleCsrfConfigurer> csrfCustomizer) {
    csrfCustomizer.customize(getOrApply(new SimpleCsrfConfigurer()));
    return this;
}
```

---

## Try It Yourself

1. **CSRF + Form Login**: Configure both `formLogin()` and `csrf()`, then verify that the
   login form POST requires a CSRF token. How does a real Thymeleaf template access the
   token to include it as a hidden field?

2. **Multiple ignore patterns**: Add several `ignoringRequestMatchers()` calls for different
   API endpoints. Verify that POST requests to those paths succeed without tokens, while
   other paths still require tokens.

3. **Per-request token rotation**: Modify `SimpleCsrfFilter` to generate a new token after
   each successful POST request (invalidating the old one). This is a defense-in-depth
   technique. What test would break?

---

## Why This Works

### The synchronizer token pattern

CSRF protection relies on one simple fact: the same-origin policy prevents a foreign website
from reading content from your domain. So the server generates a random token and gives it
to the client (via session or cookie). On subsequent requests, the client must send the token
back. An attacker's website cannot obtain this token (it can't read your cookies cross-origin),
so it cannot forge a valid request.

### Why two storage strategies?

**Session-based** (default): The token lives only in server-side memory. The client never sees
the raw expected value — it only sees the token rendered into an HTML form. This is more secure
because even if an attacker finds an XSS vulnerability, they can't extract the expected token
from the session.

**Cookie-based** (for SPAs): Single-page applications use JavaScript to make API calls. They
can't render hidden form fields. Instead, the server sets the token in a cookie, JavaScript
reads the cookie and sends the value in a custom header (`X-XSRF-TOKEN`). The attacker's site
can't read cookies from your domain (same-origin policy), so it can't forge the header.

### Constant-time comparison

Why `MessageDigest.isEqual()` instead of `String.equals()`? `String.equals()` returns false
as soon as it finds a mismatched character — the response time reveals how many characters
matched. An attacker making millions of requests could measure these timing differences and
guess the token one byte at a time. `MessageDigest.isEqual()` always compares all bytes,
regardless of where the mismatch occurs.

### Why the filter sits at position 200

The CSRF filter runs *before* authentication filters (300-400) and authorization (600). This
is intentional — CSRF is about the request's origin, not the user's identity. A forged POST
from an attacker should be rejected immediately, before expensive authentication processing.

---

## What We Enhanced

| Layer | Before Feature 9 | After Feature 9 |
|-------|-------------------|-----------------|
| `SimpleHttpSecurity` FILTER_ORDER_MAP | `"CsrfFilter"` reserved at 200 (placeholder) | Renamed to `"SimpleCsrfFilter"` — now actively used |
| `SimpleHttpSecurity` DSL | `authorizeHttpRequests()`, `formLogin()`, `httpBasic()`, `exceptionHandling()` | + `csrf()` |
| CSRF behavior | None — POST/PUT/DELETE requests had no forgery protection | State-changing requests validated against stored CSRF token |
| Token storage | N/A | Session-based (default) + cookie-based (SPA-friendly) |
| Request attributes | N/A | `CsrfToken` available as `_csrf` attribute for templates |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplifications |
|------------------|----------------------|---------------------|
| `CsrfToken` | `CsrfToken` | Identical interface |
| `DefaultCsrfToken` | `DefaultCsrfToken` | Identical |
| `CsrfTokenRepository` | `CsrfTokenRepository` | No `loadDeferredToken()` (no lazy loading) |
| `SimpleHttpSessionCsrfTokenRepository` | `HttpSessionCsrfTokenRepository` | No customizable parameter/header/session-attribute names |
| `SimpleCookieCsrfTokenRepository` | `CookieCsrfTokenRepository` | No `ResponseCookie`/`SameSite` support, no customizable cookie name/path/domain |
| `SimpleCsrfFilter` | `CsrfFilter` | No `DeferredCsrfToken` (eager loading), no `CsrfTokenRequestHandler` (simple header/parameter resolution), no `OncePerRequestFilter` (extends `HttpFilter`) |
| `SimpleCsrfConfigurer` | `CsrfConfigurer` | No `SessionAuthenticationStrategy` integration, no `LogoutConfigurer` integration, no `ObservationRegistry`, no `ApplicationContext` |

**Real source files** (Spring Security 6.x):
- `web/src/main/java/org/springframework/security/web/csrf/CsrfFilter.java`
- `web/src/main/java/org/springframework/security/web/csrf/CsrfToken.java`
- `web/src/main/java/org/springframework/security/web/csrf/CsrfTokenRepository.java`
- `web/src/main/java/org/springframework/security/web/csrf/HttpSessionCsrfTokenRepository.java`
- `web/src/main/java/org/springframework/security/web/csrf/CookieCsrfTokenRepository.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/CsrfConfigurer.java`

---

## Complete Code

### New Files

#### `simple-security-web/.../web/csrf/CsrfToken.java` [NEW]

```java
package simple.security.web.csrf;

import java.io.Serializable;

/**
 * Provides the information about an expected CSRF token.
 *
 * <p>A CSRF token has three components:
 * <ul>
 *   <li>{@link #getHeaderName()} — the HTTP header that can carry the token</li>
 *   <li>{@link #getParameterName()} — the HTTP request parameter that can carry the token</li>
 *   <li>{@link #getToken()} — the actual secret token value</li>
 * </ul>
 *
 * <p>The filter checks BOTH the header and the parameter — either one can carry
 * the token value. Headers are typically used by JavaScript SPAs, while parameters
 * are used by traditional form submissions (hidden {@code <input>} field).
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.csrf.CsrfToken}.
 *
 * Mirrors: org.springframework.security.web.csrf.CsrfToken
 * Source:  web/src/main/java/org/springframework/security/web/csrf/CsrfToken.java
 */
public interface CsrfToken extends Serializable {

	/**
	 * Gets the HTTP header name that the CSRF token is populated on the response
	 * and can be placed on requests instead of the parameter.
	 * @return the HTTP header name, never null
	 */
	String getHeaderName();

	/**
	 * Gets the HTTP parameter name that should contain the token.
	 * @return the HTTP parameter name, never null
	 */
	String getParameterName();

	/**
	 * Gets the token value.
	 * @return the token value, never null
	 */
	String getToken();

}
```

#### `simple-security-web/.../web/csrf/DefaultCsrfToken.java` [NEW]

```java
package simple.security.web.csrf;

/**
 * A CSRF token that holds the header name, parameter name, and token value.
 *
 * <p>Instances are created by {@link CsrfTokenRepository} implementations
 * and consumed by {@link SimpleCsrfFilter} for validation.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.csrf.DefaultCsrfToken}.
 *
 * Mirrors: org.springframework.security.web.csrf.DefaultCsrfToken
 * Source:  web/src/main/java/org/springframework/security/web/csrf/DefaultCsrfToken.java
 */
public final class DefaultCsrfToken implements CsrfToken {

	private static final long serialVersionUID = 1L;

	private final String headerName;

	private final String parameterName;

	private final String token;

	public DefaultCsrfToken(String headerName, String parameterName, String token) {
		if (headerName == null || headerName.isEmpty()) {
			throw new IllegalArgumentException("headerName cannot be null or empty");
		}
		if (parameterName == null || parameterName.isEmpty()) {
			throw new IllegalArgumentException("parameterName cannot be null or empty");
		}
		if (token == null || token.isEmpty()) {
			throw new IllegalArgumentException("token cannot be null or empty");
		}
		this.headerName = headerName;
		this.parameterName = parameterName;
		this.token = token;
	}

	@Override
	public String getHeaderName() {
		return this.headerName;
	}

	@Override
	public String getParameterName() {
		return this.parameterName;
	}

	@Override
	public String getToken() {
		return this.token;
	}

}
```

#### `simple-security-web/.../web/csrf/CsrfTokenRepository.java` [NEW]

```java
package simple.security.web.csrf;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

/**
 * Strategy interface for storing and loading {@link CsrfToken}s.
 *
 * <p>Two built-in implementations:
 * <ul>
 *   <li>{@link SimpleHttpSessionCsrfTokenRepository} — stores the token in the
 *       {@link HttpSession} (default, more secure)</li>
 *   <li>{@link SimpleCookieCsrfTokenRepository} — stores the token in a cookie
 *       (needed for SPAs where JavaScript must read the token)</li>
 * </ul>
 *
 * <p>Simplifications vs. real {@code CsrfTokenRepository}:
 * <ul>
 *   <li>No {@code loadDeferredToken()} — tokens are loaded eagerly</li>
 * </ul>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.csrf.CsrfTokenRepository}.
 *
 * Mirrors: org.springframework.security.web.csrf.CsrfTokenRepository
 * Source:  web/src/main/java/org/springframework/security/web/csrf/CsrfTokenRepository.java
 */
public interface CsrfTokenRepository {

	CsrfToken generateToken(HttpServletRequest request);

	void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response);

	CsrfToken loadToken(HttpServletRequest request);

}
```

#### `simple-security-web/.../web/csrf/SimpleHttpSessionCsrfTokenRepository.java` [NEW]

```java
package simple.security.web.csrf;

import java.util.UUID;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

/**
 * A {@link CsrfTokenRepository} that stores the {@link CsrfToken} in the
 * {@link HttpSession}.
 *
 * <p>This is the default repository. Session-based storage is more secure than
 * cookie-based because the token value is never sent to the browser — it lives
 * only in server-side session memory. The browser must submit the token via a
 * hidden form field or a custom header, but can never read the expected value.
 *
 * <p>Simplifications vs. real {@code HttpSessionCsrfTokenRepository}:
 * <ul>
 *   <li>No customizable parameter/header/session-attribute names</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository
 * Source:  web/src/main/java/org/springframework/security/web/csrf/HttpSessionCsrfTokenRepository.java
 */
public final class SimpleHttpSessionCsrfTokenRepository implements CsrfTokenRepository {

	static final String DEFAULT_CSRF_PARAMETER_NAME = "_csrf";

	static final String DEFAULT_CSRF_HEADER_NAME = "X-CSRF-TOKEN";

	private static final String DEFAULT_CSRF_TOKEN_ATTR_NAME =
			SimpleHttpSessionCsrfTokenRepository.class.getName().concat(".CSRF_TOKEN");

	@Override
	public CsrfToken generateToken(HttpServletRequest request) {
		return new DefaultCsrfToken(DEFAULT_CSRF_HEADER_NAME, DEFAULT_CSRF_PARAMETER_NAME,
				UUID.randomUUID().toString());
	}

	@Override
	public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
		if (token == null) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				session.removeAttribute(DEFAULT_CSRF_TOKEN_ATTR_NAME);
			}
		}
		else {
			HttpSession session = request.getSession();
			session.setAttribute(DEFAULT_CSRF_TOKEN_ATTR_NAME, token);
		}
	}

	@Override
	public CsrfToken loadToken(HttpServletRequest request) {
		HttpSession session = request.getSession(false);
		if (session == null) {
			return null;
		}
		return (CsrfToken) session.getAttribute(DEFAULT_CSRF_TOKEN_ATTR_NAME);
	}

}
```

#### `simple-security-web/.../web/csrf/SimpleCookieCsrfTokenRepository.java` [NEW]

```java
package simple.security.web.csrf;

import java.util.UUID;

import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * A {@link CsrfTokenRepository} that persists the CSRF token in a cookie named
 * "XSRF-TOKEN" and reads from the header "X-XSRF-TOKEN", following the conventions
 * of AngularJS and other SPA frameworks.
 *
 * <p>Cookie-based CSRF works because of the same-origin policy:
 * <ul>
 *   <li>The server sets a cookie with the CSRF token value</li>
 *   <li>JavaScript on the same origin reads the cookie and sends the value back
 *       in a custom header ({@code X-XSRF-TOKEN})</li>
 *   <li>An attacker site cannot read cookies from a different origin, so it
 *       cannot forge the custom header</li>
 * </ul>
 *
 * <p>Use {@link #withHttpOnlyFalse()} for SPAs — JavaScript needs to read the
 * cookie value, which requires {@code httpOnly=false}.
 *
 * <p>Simplifications vs. real {@code CookieCsrfTokenRepository}:
 * <ul>
 *   <li>No {@code ResponseCookie} / {@code SameSite} support — uses plain {@code Cookie} API</li>
 *   <li>No customizable cookie name/path/domain/maxAge</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.web.csrf.CookieCsrfTokenRepository
 * Source:  web/src/main/java/org/springframework/security/web/csrf/CookieCsrfTokenRepository.java
 */
public final class SimpleCookieCsrfTokenRepository implements CsrfTokenRepository {

	static final String DEFAULT_CSRF_COOKIE_NAME = "XSRF-TOKEN";

	static final String DEFAULT_CSRF_PARAMETER_NAME = "_csrf";

	static final String DEFAULT_CSRF_HEADER_NAME = "X-XSRF-TOKEN";

	private boolean cookieHttpOnly = true;

	@Override
	public CsrfToken generateToken(HttpServletRequest request) {
		return new DefaultCsrfToken(DEFAULT_CSRF_HEADER_NAME, DEFAULT_CSRF_PARAMETER_NAME,
				UUID.randomUUID().toString());
	}

	@Override
	public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
		String tokenValue = (token != null) ? token.getToken() : "";
		Cookie cookie = new Cookie(DEFAULT_CSRF_COOKIE_NAME, tokenValue);
		cookie.setSecure(request.isSecure());
		cookie.setPath(getRequestContext(request));
		cookie.setMaxAge((token != null) ? -1 : 0);
		cookie.setHttpOnly(this.cookieHttpOnly);
		response.addCookie(cookie);
	}

	@Override
	public CsrfToken loadToken(HttpServletRequest request) {
		Cookie[] cookies = request.getCookies();
		if (cookies == null) {
			return null;
		}
		for (Cookie cookie : cookies) {
			if (DEFAULT_CSRF_COOKIE_NAME.equals(cookie.getName())) {
				String token = cookie.getValue();
				if (token != null && !token.isEmpty()) {
					return new DefaultCsrfToken(DEFAULT_CSRF_HEADER_NAME, DEFAULT_CSRF_PARAMETER_NAME, token);
				}
			}
		}
		return null;
	}

	public static SimpleCookieCsrfTokenRepository withHttpOnlyFalse() {
		SimpleCookieCsrfTokenRepository result = new SimpleCookieCsrfTokenRepository();
		result.cookieHttpOnly = false;
		return result;
	}

	private String getRequestContext(HttpServletRequest request) {
		String contextPath = request.getContextPath();
		return (contextPath != null && !contextPath.isEmpty()) ? contextPath : "/";
	}

}
```

#### `simple-security-web/.../web/csrf/SimpleCsrfFilter.java` [NEW]

```java
package simple.security.web.csrf;

import java.io.IOException;
import java.security.MessageDigest;
import java.util.Set;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpFilter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.access.AccessDeniedException;
import simple.security.web.access.AccessDeniedHandler;
import simple.security.web.access.SimpleAccessDeniedHandlerImpl;
import simple.security.web.util.matcher.RequestMatcher;

/**
 * Applies CSRF protection using the synchronizer token pattern.
 *
 * <h2>How it works</h2>
 * <ol>
 *   <li>Load the expected CSRF token from the {@link CsrfTokenRepository}</li>
 *   <li>If no stored token exists, generate and save a new one</li>
 *   <li>Make the token available as a request attribute (for Thymeleaf, JSP, etc.)</li>
 *   <li>If the request method is GET, HEAD, TRACE, or OPTIONS → pass through</li>
 *   <li>Otherwise, extract the actual token from the request header or parameter</li>
 *   <li>Compare expected vs. actual using constant-time comparison</li>
 *   <li>If mismatch → delegate to {@link AccessDeniedHandler}</li>
 *   <li>If match → continue the filter chain</li>
 * </ol>
 *
 * <p>Simplifications vs. real {@code CsrfFilter}:
 * <ul>
 *   <li>No {@code DeferredCsrfToken} — token is loaded eagerly</li>
 *   <li>No {@code CsrfTokenRequestHandler} — simple header/parameter resolution</li>
 *   <li>No {@code OncePerRequestFilter} — extends {@code HttpFilter} directly</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.web.csrf.CsrfFilter
 * Source:  web/src/main/java/org/springframework/security/web/csrf/CsrfFilter.java
 */
public final class SimpleCsrfFilter extends HttpFilter {

	private static final Set<String> ALLOWED_METHODS = Set.of("GET", "HEAD", "TRACE", "OPTIONS");

	private final CsrfTokenRepository tokenRepository;

	private RequestMatcher requireCsrfProtectionMatcher = request -> !ALLOWED_METHODS.contains(request.getMethod());

	private AccessDeniedHandler accessDeniedHandler = new SimpleAccessDeniedHandlerImpl();

	public SimpleCsrfFilter(CsrfTokenRepository tokenRepository) {
		if (tokenRepository == null) {
			throw new IllegalArgumentException("tokenRepository cannot be null");
		}
		this.tokenRepository = tokenRepository;
	}

	@Override
	protected void doFilter(HttpServletRequest request, HttpServletResponse response,
			FilterChain chain) throws IOException, ServletException {

		// 1. Load existing token or generate a new one
		CsrfToken csrfToken = this.tokenRepository.loadToken(request);
		boolean missingToken = (csrfToken == null);
		if (missingToken) {
			csrfToken = this.tokenRepository.generateToken(request);
			this.tokenRepository.saveToken(csrfToken, request, response);
		}

		// 2. Make token available as request attribute
		request.setAttribute(CsrfToken.class.getName(), csrfToken);
		request.setAttribute(csrfToken.getParameterName(), csrfToken);

		// 3. Skip validation for safe methods
		if (!this.requireCsrfProtectionMatcher.matches(request)) {
			chain.doFilter(request, response);
			return;
		}

		// 4. Extract the actual token from the request
		String actualToken = request.getHeader(csrfToken.getHeaderName());
		if (actualToken == null) {
			actualToken = request.getParameter(csrfToken.getParameterName());
		}

		// 5. Compare using constant-time comparison
		if (!equalsConstantTime(csrfToken.getToken(), actualToken)) {
			AccessDeniedException exception = missingToken
					? new AccessDeniedException("CSRF token missing")
					: new AccessDeniedException("Invalid CSRF token");
			this.accessDeniedHandler.handle(request, response, exception);
			return;
		}

		// 6. Token valid — continue
		chain.doFilter(request, response);
	}

	public void setRequireCsrfProtectionMatcher(RequestMatcher requireCsrfProtectionMatcher) {
		if (requireCsrfProtectionMatcher == null) {
			throw new IllegalArgumentException("requireCsrfProtectionMatcher cannot be null");
		}
		this.requireCsrfProtectionMatcher = requireCsrfProtectionMatcher;
	}

	public void setAccessDeniedHandler(AccessDeniedHandler accessDeniedHandler) {
		if (accessDeniedHandler == null) {
			throw new IllegalArgumentException("accessDeniedHandler cannot be null");
		}
		this.accessDeniedHandler = accessDeniedHandler;
	}

	private static boolean equalsConstantTime(String expected, String actual) {
		if (expected == actual) {
			return true;
		}
		if (expected == null || actual == null) {
			return false;
		}
		byte[] expectedBytes = expected.getBytes(java.nio.charset.StandardCharsets.UTF_8);
		byte[] actualBytes = actual.getBytes(java.nio.charset.StandardCharsets.UTF_8);
		return MessageDigest.isEqual(expectedBytes, actualBytes);
	}

}
```

#### `simple-security-config/.../configurers/SimpleCsrfConfigurer.java` [NEW]

```java
package simple.security.config.annotation.web.configurers;

import java.util.ArrayList;
import java.util.List;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.access.AccessDeniedHandler;
import simple.security.web.access.SimpleAccessDeniedHandlerImpl;
import simple.security.web.csrf.CsrfTokenRepository;
import simple.security.web.csrf.SimpleCsrfFilter;
import simple.security.web.csrf.SimpleHttpSessionCsrfTokenRepository;
import simple.security.web.util.matcher.RequestMatcher;

/**
 * Configures CSRF protection via the {@link SimpleHttpSecurity} DSL.
 *
 * <p>During {@code configure()}, this configurer:
 * <ol>
 *   <li>Creates a {@link SimpleCsrfFilter} with the configured
 *       {@link CsrfTokenRepository}</li>
 *   <li>Combines the default CSRF matcher with any ignore matchers</li>
 *   <li>Sets the {@link AccessDeniedHandler} for token validation failures</li>
 *   <li>Adds the filter at position 200 in the filter chain</li>
 * </ol>
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.CsrfConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/CsrfConfigurer.java
 */
public class SimpleCsrfConfigurer extends SimpleAbstractHttpConfigurer<SimpleCsrfConfigurer> {

	private CsrfTokenRepository csrfTokenRepository = new SimpleHttpSessionCsrfTokenRepository();

	private AccessDeniedHandler accessDeniedHandler;

	private final List<RequestMatcher> ignoredCsrfProtectionMatchers = new ArrayList<>();

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		SimpleCsrfFilter filter = new SimpleCsrfFilter(this.csrfTokenRepository);

		if (!this.ignoredCsrfProtectionMatchers.isEmpty()) {
			filter.setRequireCsrfProtectionMatcher(request -> {
				String method = request.getMethod();
				if ("GET".equals(method) || "HEAD".equals(method)
						|| "TRACE".equals(method) || "OPTIONS".equals(method)) {
					return false;
				}
				for (RequestMatcher ignoreMatcher : this.ignoredCsrfProtectionMatchers) {
					if (ignoreMatcher.matches(request)) {
						return false;
					}
				}
				return true;
			});
		}

		AccessDeniedHandler deniedHandler = resolveAccessDeniedHandler();
		filter.setAccessDeniedHandler(deniedHandler);

		builder.addFilter(filter);
	}

	public SimpleCsrfConfigurer csrfTokenRepository(CsrfTokenRepository csrfTokenRepository) {
		if (csrfTokenRepository == null) {
			throw new IllegalArgumentException("csrfTokenRepository cannot be null");
		}
		this.csrfTokenRepository = csrfTokenRepository;
		return this;
	}

	public SimpleCsrfConfigurer accessDeniedHandler(AccessDeniedHandler accessDeniedHandler) {
		this.accessDeniedHandler = accessDeniedHandler;
		return this;
	}

	public SimpleCsrfConfigurer ignoringRequestMatchers(String... patterns) {
		for (String pattern : patterns) {
			this.ignoredCsrfProtectionMatchers.add(
					new simple.security.web.util.matcher.SimplePathPatternRequestMatcher(pattern));
		}
		return this;
	}

	public SimpleCsrfConfigurer ignoringRequestMatchers(RequestMatcher... requestMatchers) {
		for (RequestMatcher matcher : requestMatchers) {
			this.ignoredCsrfProtectionMatchers.add(matcher);
		}
		return this;
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

Changes:
1. Filter order map: `"CsrfFilter"` → `"SimpleCsrfFilter"` at position 200
2. Added import for `SimpleCsrfConfigurer`
3. Added `csrf()` DSL method

```java
// Change 1: Filter order map entry
FILTER_ORDER_MAP.put("SimpleCsrfFilter", order); // was "CsrfFilter"

// Change 2: New import
import simple.security.config.annotation.web.configurers.SimpleCsrfConfigurer;

// Change 3: New DSL method (added between authorizeHttpRequests and formLogin)
public SimpleHttpSecurity csrf(Customizer<SimpleCsrfConfigurer> csrfCustomizer) {
    csrfCustomizer.customize(getOrApply(new SimpleCsrfConfigurer()));
    return this;
}
```
