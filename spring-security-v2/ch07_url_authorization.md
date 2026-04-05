# Chapter 7: URL Authorization

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure authorization rules via the DSL:
SimpleHttpSecurity http = new SimpleHttpSecurity();
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/**").hasAnyAuthority("SCOPE_read", "ROLE_USER")
    .anyRequest().authenticated());

SecurityFilterChain chain = http.build();

// An AuthorizationFilter is added at position 600 in the filter chain.
// For each request, it:
//   1. Reads the current Authentication from the SecurityContext
//   2. Iterates authorization rules in order — first matching rule wins
//   3. Grants or denies access based on the matched rule
//   4. If no rule matches → denied by default (secure-by-default)

// Custom AuthorizationManager for advanced scenarios:
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/special/**").access(
        (authentication, request) -> {
            String header = request.getHeader("X-Allow");
            return new AuthorizationDecision("true".equals(header));
        })
    .anyRequest().authenticated());
```

Questions to guide your thinking:
- Why does `AuthorizationManager` use `Supplier<Authentication>` instead of `Authentication` directly? When does this lazy resolution matter?
- Why does `SimpleRequestMatcherDelegatingAuthorizationManager` deny by default when no matcher matches, rather than allowing?
- How does the configurer bridge the fluent DSL (`requestMatchers("/admin/**").hasRole("ADMIN")`) into a runtime `(RequestMatcher, AuthorizationManager)` pair?
- Why is `AuthorizationManager` a `@FunctionalInterface`? Where does this pay off?

---

## The API Contract

Feature 7 adds URL-level authorization — the ability to protect URL patterns with
rules like `permitAll()`, `hasRole("ADMIN")`, and `authenticated()`. After authentication
establishes *who* the user is, authorization decides *what they can access*.

### The DSL Entry Point

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity authorizeHttpRequests(
        Customizer<SimpleAuthorizeHttpRequestsConfigurer.AuthorizationManagerRequestMatcherRegistry>
            authorizeHttpRequestsCustomizer) {
    SimpleAuthorizeHttpRequestsConfigurer configurer =
            getOrApply(new SimpleAuthorizeHttpRequestsConfigurer());
    authorizeHttpRequestsCustomizer.customize(configurer.getRegistry());
    return this;
}
```

Notice that the `Customizer` receives the **registry** (not the configurer). The user
interacts with `AuthorizationManagerRequestMatcherRegistry`, which provides the fluent
`.requestMatchers(...).hasRole(...)` API.

### The Core Authorization Interface

```java
// simple/security/authorization/AuthorizationManager.java
@FunctionalInterface
public interface AuthorizationManager<T> {
    AuthorizationDecision authorize(Supplier<Authentication> authentication, T object);

    default void verify(Supplier<Authentication> authentication, T object) {
        AuthorizationDecision decision = authorize(authentication, object);
        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }
    }
}
```

### The Decision Value Object

```java
// simple/security/authorization/AuthorizationDecision.java
public class AuthorizationDecision {
    private final boolean granted;
    public AuthorizationDecision(boolean granted) { this.granted = granted; }
    public boolean isGranted() { return this.granted; }
}
```

---

## Client Usage & Tests

### Basic Authorization Rules

```java
// permitAll — no authentication required
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()
    .anyRequest().authenticated());

// hasRole — requires ROLE_ADMIN authority
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());

// hasAnyAuthority — requires any of the listed authorities
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/**").hasAnyAuthority("SCOPE_read", "ROLE_USER")
    .anyRequest().authenticated());
```

### Test: Permit All Bypasses Authentication

```java
@Test
void shouldPermitAllEvenWithoutAuthentication() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http ->
            http.authorizeHttpRequests(auth -> auth
                    .requestMatchers("/health").permitAll()
                    .anyRequest().authenticated()));

    // No authentication set — permitAll should not even check
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/health");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(request, response, chain);

    assertThat(chain.getRequest()).isNotNull(); // Chain continues
}
```

### Test: hasRole Denies Unauthorized User

```java
@Test
void shouldDenyUserWithoutRequiredRole() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http ->
            http.authorizeHttpRequests(auth -> auth
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()));

    authenticateAs("user", "ROLE_USER");

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/admin/users");
    MockHttpServletResponse response = new MockHttpServletResponse();

    assertThatThrownBy(() ->
            proxy.doFilter(request, response, new MockFilterChain()))
            .isInstanceOf(AccessDeniedException.class);
}
```

### Test: First-Match-Wins Ordering

```java
@Test
void shouldApplyFirstMatchingRule() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http ->
            http.authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/public/**").permitAll()
                    .requestMatchers("/api/**").hasRole("ADMIN")
                    .anyRequest().authenticated()));

    // /api/public/health matches the FIRST rule (permitAll)
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/public/health");
    proxy.doFilter(request, new MockHttpServletResponse(), new MockFilterChain());
    // No exception → access granted
}
```

### Test: Deny by Default

```java
@Test
void shouldDenyByDefaultWhenNoRuleMatches() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http ->
            http.authorizeHttpRequests(auth -> auth
                    .requestMatchers("/known/**").permitAll()));

    authenticateAs("user", "ROLE_USER");

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/unknown");

    assertThatThrownBy(() ->
            proxy.doFilter(request, new MockHttpServletResponse(), new MockFilterChain()))
            .isInstanceOf(AccessDeniedException.class);
}
```

---

## Implementing the Call Chain

The call chain for URL authorization flows through four layers:

```
Client calls: http.authorizeHttpRequests(auth -> auth
                    .requestMatchers("/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated())
  │
  ├─► [DSL Layer] SimpleAuthorizeHttpRequestsConfigurer
  │     Accumulates (RequestMatcher, AuthorizationManager) pairs
  │
  ├─► [Build Phase] configurer.configure(http)
  │     Builds RequestMatcherDelegatingAuthorizationManager from entries
  │     Creates SimpleAuthorizationFilter, adds at position 600
  │
  ├─► [Request Time] SimpleAuthorizationFilter.doFilter()
  │     Gets Authentication from SecurityContext (lazy Supplier)
  │     Delegates to AuthorizationManager
  │
  ├─► [Routing] SimpleRequestMatcherDelegatingAuthorizationManager.authorize()
  │     Iterates entries — first matching RequestMatcher wins
  │     Delegates to that entry's AuthorizationManager
  │
  └─► [Decision] SimpleAuthorityAuthorizationManager / SimpleAuthenticatedAuthorizationManager
        Checks authorities or authentication status
        Returns AuthorizationDecision(true/false)
```

### Layer 1: The API Core (simple-security-core)

#### AuthorizationManager — The Central Interface

```java
@FunctionalInterface
public interface AuthorizationManager<T> {
    AuthorizationDecision authorize(Supplier<Authentication> authentication, T object);
}
```

The `@FunctionalInterface` annotation means you can use lambdas:

```java
// permitAll as a lambda:
AuthorizationManager<HttpServletRequest> permitAll =
    (auth, request) -> new AuthorizationDecision(true);
```

The `Supplier<Authentication>` enables **lazy resolution** — the SecurityContext
ThreadLocal is only accessed if the manager actually calls `.get()`. This avoids
unnecessary work for `permitAll()` rules where authentication doesn't matter.

#### SimpleAuthorityAuthorizationManager — Role & Authority Checks

```java
public final class SimpleAuthorityAuthorizationManager<T> implements AuthorizationManager<T> {

    private static final String ROLE_PREFIX = "ROLE_";
    private final Set<String> authorities;

    // Private constructor — use static factories
    private SimpleAuthorityAuthorizationManager(String... authorities) {
        this.authorities = Set.of(authorities);
    }

    public static <T> SimpleAuthorityAuthorizationManager<T> hasRole(String role) {
        // Validates role doesn't start with ROLE_ (common mistake)
        return hasAuthority(ROLE_PREFIX + role);
    }

    public static <T> SimpleAuthorityAuthorizationManager<T> hasAuthority(String authority) {
        return new SimpleAuthorityAuthorizationManager<>(authority);
    }

    @Override
    public AuthorizationDecision authorize(Supplier<Authentication> authentication, T object) {
        boolean granted = isGranted(authentication.get());
        return new AuthorizationDecision(granted);
    }

    private boolean isGranted(Authentication authentication) {
        if (authentication == null || !authentication.isAuthenticated()) {
            return false;
        }
        for (GrantedAuthority ga : authentication.getAuthorities()) {
            if (this.authorities.contains(ga.getAuthority())) {
                return true;  // Set-intersection: any match grants access
            }
        }
        return false;
    }
}
```

Key design: `hasRole("ADMIN")` auto-prepends `ROLE_` and delegates to `hasAuthority("ROLE_ADMIN")`.
This is why Spring Security uses `@PreAuthorize("hasRole('ADMIN')")` but the user's
`GrantedAuthority` must be `"ROLE_ADMIN"`. The real framework validates this eagerly —
passing `"ROLE_ADMIN"` to `hasRole()` throws an `IllegalArgumentException`.

#### SimpleAuthenticatedAuthorizationManager — Authentication Check

```java
public final class SimpleAuthenticatedAuthorizationManager<T> implements AuthorizationManager<T> {

    public static <T> SimpleAuthenticatedAuthorizationManager<T> authenticated() {
        return new SimpleAuthenticatedAuthorizationManager<>();
    }

    @Override
    public AuthorizationDecision authorize(Supplier<Authentication> authentication, T object) {
        Authentication auth = authentication.get();
        boolean granted = auth != null && auth.isAuthenticated();
        return new AuthorizationDecision(granted);
    }
}
```

Our simplified version only checks `isAuthenticated()`. The real framework distinguishes
four levels: fully authenticated, remember-me, anonymous, and any-authenticated —
using a Strategy pattern with inner classes.

### Layer 2: The Routing Layer (simple-security-web)

#### SimpleRequestMatcherDelegatingAuthorizationManager — First-Match Routing

```java
public final class SimpleRequestMatcherDelegatingAuthorizationManager
        implements AuthorizationManager<HttpServletRequest> {

    private static final AuthorizationDecision DENY = new AuthorizationDecision(false);
    private final List<RequestMatcherEntry> mappings;

    @Override
    public AuthorizationDecision authorize(Supplier<Authentication> authentication,
            HttpServletRequest request) {
        for (RequestMatcherEntry entry : this.mappings) {
            if (entry.requestMatcher().matches(request)) {
                return entry.authorizationManager().authorize(authentication, request);
            }
        }
        return DENY;  // No matcher matched → deny by default
    }

    public record RequestMatcherEntry(
            RequestMatcher requestMatcher,
            AuthorizationManager<HttpServletRequest> authorizationManager) {
    }
}
```

The **deny-by-default** when no matcher matches is the secure-by-default posture.
If you configure `/public/**` as `permitAll()` but forget to cover `/admin/**`,
the admin endpoints are automatically protected.

#### SimpleAuthorizationFilter — The Servlet Filter

```java
public class SimpleAuthorizationFilter implements Filter {

    private final AuthorizationManager<HttpServletRequest> authorizationManager;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        AuthorizationDecision decision = this.authorizationManager.authorize(
                this::getAuthentication, httpRequest);

        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }

        chain.doFilter(request, response);
    }

    private Authentication getAuthentication() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null) {
            throw new AuthenticationCredentialsNotFoundException(...);
        }
        return authentication;
    }
}
```

The filter is intentionally simple — it delegates ALL authorization logic to the
`AuthorizationManager`. It doesn't know about URLs, roles, or authorities. This is
the **separation of concerns** that makes Spring Security extensible: you can replace
the entire authorization strategy by providing a different `AuthorizationManager`.

### Layer 3: The DSL Configurer (simple-security-config)

#### SimpleAuthorizeHttpRequestsConfigurer — Bridging DSL to Runtime

```java
public class SimpleAuthorizeHttpRequestsConfigurer
        extends SimpleAbstractHttpConfigurer<SimpleAuthorizeHttpRequestsConfigurer> {

    private final AuthorizationManagerRequestMatcherRegistry registry =
            new AuthorizationManagerRequestMatcherRegistry();

    @Override
    public void configure(SimpleHttpSecurity builder) throws Exception {
        SimpleRequestMatcherDelegatingAuthorizationManager authorizationManager =
                this.registry.createAuthorizationManager();
        SimpleAuthorizationFilter filter = new SimpleAuthorizationFilter(authorizationManager);
        builder.addFilter(filter);
    }
}
```

The configurer's `configure()` method is called during the build phase. It:
1. Builds the `RequestMatcherDelegatingAuthorizationManager` from accumulated entries
2. Wraps it in a `SimpleAuthorizationFilter`
3. Adds the filter at its well-known position (600)

#### AuthorizationManagerRequestMatcherRegistry — The Fluent Chain

```java
public static final class AuthorizationManagerRequestMatcherRegistry {

    private final SimpleRequestMatcherDelegatingAuthorizationManager.Builder managerBuilder;

    public AuthorizedUrl requestMatchers(String... patterns) {
        List<RequestMatcher> matchers = new ArrayList<>();
        for (String pattern : patterns) {
            matchers.add(new SimplePathPatternRequestMatcher(pattern));
        }
        return new AuthorizedUrl(matchers);
    }

    public AuthorizedUrl anyRequest() {
        return new AuthorizedUrl(List.of(SimpleAnyRequestMatcher.INSTANCE));
    }
}
```

#### AuthorizedUrl — The Terminal Methods

```java
public final class AuthorizedUrl {
    private final List<RequestMatcher> matchers;

    public AuthorizationManagerRequestMatcherRegistry permitAll() {
        return access((auth, request) -> new AuthorizationDecision(true));
    }

    public AuthorizationManagerRequestMatcherRegistry hasRole(String role) {
        return access(SimpleAuthorityAuthorizationManager.hasRole(role));
    }

    public AuthorizationManagerRequestMatcherRegistry authenticated() {
        return access(SimpleAuthenticatedAuthorizationManager.authenticated());
    }

    public AuthorizationManagerRequestMatcherRegistry access(
            AuthorizationManager<HttpServletRequest> manager) {
        for (RequestMatcher matcher : this.matchers) {
            managerBuilder.add(matcher, manager);
        }
        return AuthorizationManagerRequestMatcherRegistry.this;
    }
}
```

Each terminal method (`permitAll()`, `hasRole()`, `authenticated()`, etc.) creates
the appropriate `AuthorizationManager` and passes it to `access()`, which pairs
it with each `RequestMatcher` and adds it to the builder.

### Modification: SimpleHttpSecurity [MODIFIED]

Two changes were needed:
1. **Filter order map**: Renamed `"AuthorizationFilter"` to `"SimpleAuthorizationFilter"` to
   match the actual class name used by `addFilter()`
2. **New DSL method**: Added `authorizeHttpRequests()` following the same pattern as
   `formLogin()` and `httpBasic()`

---

## Try It Yourself

1. **Add `hasAllAuthorities()`**: Implement a method where the user must have ALL of
   the specified authorities (not just any). Hint: change the `isGranted` check from
   "any match" to "all match".

2. **Add a NOT operator**: Implement `.not().hasRole("GUEST")` to deny access to guests
   while allowing everyone else. The real framework added this in Spring Security 6.3.

3. **Add `denyAll()` to AuthorizationManager directly**: Create a static factory
   `AuthorizationManager.denyAll()` that returns a shared singleton instance.

---

## Why This Works

### Separation of Concerns at Every Layer

The authorization system demonstrates clean layering:
- **AuthorizationManager** — knows about authentication and decisions, nothing about HTTP
- **RequestMatcherDelegatingAuthorizationManager** — knows about URL routing, delegates actual checks
- **SimpleAuthorizationFilter** — knows about servlets, delegates everything to AuthorizationManager
- **SimpleAuthorizeHttpRequestsConfigurer** — knows about DSL, bridges to runtime objects

Each layer has a single responsibility and delegates to the next.

### The Supplier Pattern for Lazy Resolution

`Supplier<Authentication>` is not just a code style choice — it has real performance
implications. Consider `permitAll()`:

```java
// permitAll creates: (auth, request) -> new AuthorizationDecision(true)
// The 'auth' Supplier is NEVER called — SecurityContext is never read
```

For high-traffic public endpoints, avoiding the `ThreadLocal` lookup matters. The real
framework uses this same pattern across all authorization checks.

### Why @FunctionalInterface Matters

Making `AuthorizationManager` a functional interface enables inline lambdas for custom rules:

```java
.requestMatchers("/special/**").access((auth, request) -> {
    String timeHeader = request.getHeader("X-Business-Hours");
    return new AuthorizationDecision("true".equals(timeHeader));
})
```

This is far more flexible than the old `ConfigAttribute`-based system Spring Security
used before version 5.5.

---

## What We Enhanced

| Component | Before Feature 7 | After Feature 7 |
|-----------|-------------------|-----------------|
| `SimpleHttpSecurity` | `formLogin()`, `httpBasic()` only | Added `authorizeHttpRequests()` DSL |
| Filter order map | `"AuthorizationFilter"` placeholder | Renamed to `"SimpleAuthorizationFilter"` |
| Core module | No authorization API | `AuthorizationManager`, `AuthorizationDecision`, `SimpleAuthorityAuthorizationManager`, `SimpleAuthenticatedAuthorizationManager` |
| Web module | No authorization filter | `SimpleAuthorizationFilter`, `SimpleRequestMatcherDelegatingAuthorizationManager` |
| Config module | No authorization configurer | `SimpleAuthorizeHttpRequestsConfigurer` with `AuthorizationManagerRequestMatcherRegistry` and `AuthorizedUrl` |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | File:Line Reference |
|------------------|----------------------|---------------------|
| `AuthorizationManager<T>` | `o.s.s.authorization.AuthorizationManager` | `core/src/.../authorization/AuthorizationManager.java` |
| `AuthorizationDecision` | `o.s.s.authorization.AuthorizationDecision` | `core/src/.../authorization/AuthorizationDecision.java` |
| `SimpleAuthorityAuthorizationManager` | `o.s.s.authorization.AuthorityAuthorizationManager` | `core/src/.../authorization/AuthorityAuthorizationManager.java` |
| `SimpleAuthenticatedAuthorizationManager` | `o.s.s.authorization.AuthenticatedAuthorizationManager` | `core/src/.../authorization/AuthenticatedAuthorizationManager.java` |
| `SimpleAuthorizationFilter` | `o.s.s.web.access.intercept.AuthorizationFilter` | `web/src/.../access/intercept/AuthorizationFilter.java` |
| `SimpleRequestMatcherDelegatingAuthorizationManager` | `o.s.s.web.access.intercept.RequestMatcherDelegatingAuthorizationManager` | `web/src/.../access/intercept/RequestMatcherDelegatingAuthorizationManager.java` |
| `SimpleAuthorizeHttpRequestsConfigurer` | `o.s.s.config.annotation.web.configurers.AuthorizeHttpRequestsConfigurer` | `config/src/.../configurers/AuthorizeHttpRequestsConfigurer.java` |

### Key Simplifications

| Real Framework | Simplified Version | Why |
|---|---|---|
| `AuthorizationResult` interface + `AuthorizationDecision` class | `AuthorizationDecision` only | Added in Spring Security 6.4; unnecessary abstraction layer for learning |
| `AuthoritiesAuthorizationManager` delegate inside `AuthorityAuthorizationManager` | Direct authority matching in `SimpleAuthorityAuthorizationManager` | Eliminates an extra delegation layer; same behavior |
| 4 authentication levels (fully, remember-me, anonymous, any) | Only `isAuthenticated()` check | Remember-me and anonymous are future features |
| `AuthorizationManagerFactory` indirection for role prefix customization | Direct static factory calls | No need for configurable role prefix yet |
| `AuthorizationEventPublisher` for audit events | Omitted | Event publishing is infrastructure, not core authorization |
| `RequestAuthorizationContext` wrapping request + path variables | Pass `HttpServletRequest` directly | Path variable extraction is not needed yet |
| `RoleHierarchy` support | Flat authority matching only | Role hierarchy is a specialized feature |

---

## Complete Code

### New Files

#### `simple-security-core/.../authorization/AuthorizationManager.java` [NEW]

```java
package simple.security.authorization;

import java.util.function.Supplier;

import simple.security.core.Authentication;

@FunctionalInterface
public interface AuthorizationManager<T> {

	AuthorizationDecision authorize(Supplier<Authentication> authentication, T object);

	default void verify(Supplier<Authentication> authentication, T object) {
		AuthorizationDecision decision = authorize(authentication, object);
		if (decision != null && !decision.isGranted()) {
			throw new simple.security.access.AccessDeniedException("Access Denied");
		}
	}

}
```

#### `simple-security-core/.../authorization/AuthorizationDecision.java` [NEW]

```java
package simple.security.authorization;

public class AuthorizationDecision {

	private final boolean granted;

	public AuthorizationDecision(boolean granted) {
		this.granted = granted;
	}

	public boolean isGranted() {
		return this.granted;
	}

	@Override
	public String toString() {
		return "AuthorizationDecision [granted=" + this.granted + "]";
	}

}
```

#### `simple-security-core/.../authorization/SimpleAuthorityAuthorizationManager.java` [NEW]

```java
package simple.security.authorization;

import java.util.Collection;
import java.util.Set;
import java.util.function.Supplier;

import simple.security.core.Authentication;
import simple.security.core.GrantedAuthority;

public final class SimpleAuthorityAuthorizationManager<T> implements AuthorizationManager<T> {

	private static final String ROLE_PREFIX = "ROLE_";
	private final Set<String> authorities;

	private SimpleAuthorityAuthorizationManager(String... authorities) {
		this.authorities = Set.of(authorities);
	}

	public static <T> SimpleAuthorityAuthorizationManager<T> hasRole(String role) {
		if (role.startsWith(ROLE_PREFIX)) {
			throw new IllegalArgumentException(
					"role should not start with 'ROLE_' since it is automatically prepended. "
							+ "Got '" + role + "'");
		}
		return hasAuthority(ROLE_PREFIX + role);
	}

	public static <T> SimpleAuthorityAuthorizationManager<T> hasAuthority(String authority) {
		return new SimpleAuthorityAuthorizationManager<>(authority);
	}

	public static <T> SimpleAuthorityAuthorizationManager<T> hasAnyRole(String... roles) {
		String[] authorities = new String[roles.length];
		for (int i = 0; i < roles.length; i++) {
			if (roles[i].startsWith(ROLE_PREFIX)) {
				throw new IllegalArgumentException(
						"role should not start with 'ROLE_' since it is automatically prepended. "
								+ "Got '" + roles[i] + "'");
			}
			authorities[i] = ROLE_PREFIX + roles[i];
		}
		return hasAnyAuthority(authorities);
	}

	public static <T> SimpleAuthorityAuthorizationManager<T> hasAnyAuthority(String... authorities) {
		return new SimpleAuthorityAuthorizationManager<>(authorities);
	}

	@Override
	public AuthorizationDecision authorize(Supplier<Authentication> authentication, T object) {
		boolean granted = isGranted(authentication.get());
		return new AuthorizationDecision(granted);
	}

	private boolean isGranted(Authentication authentication) {
		if (authentication == null || !authentication.isAuthenticated()) {
			return false;
		}
		Collection<? extends GrantedAuthority> userAuthorities = authentication.getAuthorities();
		for (GrantedAuthority ga : userAuthorities) {
			if (this.authorities.contains(ga.getAuthority())) {
				return true;
			}
		}
		return false;
	}

}
```

#### `simple-security-core/.../authorization/SimpleAuthenticatedAuthorizationManager.java` [NEW]

```java
package simple.security.authorization;

import java.util.function.Supplier;

import simple.security.core.Authentication;

public final class SimpleAuthenticatedAuthorizationManager<T> implements AuthorizationManager<T> {

	private SimpleAuthenticatedAuthorizationManager() {
	}

	public static <T> SimpleAuthenticatedAuthorizationManager<T> authenticated() {
		return new SimpleAuthenticatedAuthorizationManager<>();
	}

	@Override
	public AuthorizationDecision authorize(Supplier<Authentication> authentication, T object) {
		Authentication auth = authentication.get();
		boolean granted = auth != null && auth.isAuthenticated();
		return new AuthorizationDecision(granted);
	}

}
```

#### `simple-security-web/.../web/access/intercept/SimpleAuthorizationFilter.java` [NEW]

```java
package simple.security.web.access.intercept;

import java.io.IOException;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;

import simple.security.access.AccessDeniedException;
import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.core.context.SecurityContextHolder;

public class SimpleAuthorizationFilter implements Filter {

	private final AuthorizationManager<HttpServletRequest> authorizationManager;

	public SimpleAuthorizationFilter(AuthorizationManager<HttpServletRequest> authorizationManager) {
		if (authorizationManager == null) {
			throw new IllegalArgumentException("authorizationManager cannot be null");
		}
		this.authorizationManager = authorizationManager;
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest httpRequest = (HttpServletRequest) request;

		AuthorizationDecision decision = this.authorizationManager.authorize(
				this::getAuthentication, httpRequest);

		if (decision != null && !decision.isGranted()) {
			throw new AccessDeniedException("Access Denied");
		}

		chain.doFilter(request, response);
	}

	private Authentication getAuthentication() {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		if (authentication == null) {
			throw new AuthenticationCredentialsNotFoundException(
					"An Authentication object was not found in the SecurityContext");
		}
		return authentication;
	}

	public AuthorizationManager<HttpServletRequest> getAuthorizationManager() {
		return this.authorizationManager;
	}

	public static class AuthenticationCredentialsNotFoundException extends AuthenticationException {
		public AuthenticationCredentialsNotFoundException(String msg) {
			super(msg);
		}
	}

}
```

#### `simple-security-web/.../web/access/intercept/SimpleRequestMatcherDelegatingAuthorizationManager.java` [NEW]

```java
package simple.security.web.access.intercept;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Supplier;

import jakarta.servlet.http.HttpServletRequest;

import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.core.Authentication;
import simple.security.web.util.matcher.RequestMatcher;

public final class SimpleRequestMatcherDelegatingAuthorizationManager
		implements AuthorizationManager<HttpServletRequest> {

	private static final AuthorizationDecision DENY = new AuthorizationDecision(false);
	private final List<RequestMatcherEntry> mappings;

	private SimpleRequestMatcherDelegatingAuthorizationManager(List<RequestMatcherEntry> mappings) {
		this.mappings = List.copyOf(mappings);
	}

	@Override
	public AuthorizationDecision authorize(Supplier<Authentication> authentication,
			HttpServletRequest request) {
		for (RequestMatcherEntry entry : this.mappings) {
			if (entry.requestMatcher().matches(request)) {
				return entry.authorizationManager().authorize(authentication, request);
			}
		}
		return DENY;
	}

	public static Builder builder() {
		return new Builder();
	}

	public record RequestMatcherEntry(
			RequestMatcher requestMatcher,
			AuthorizationManager<HttpServletRequest> authorizationManager) {
	}

	public static final class Builder {
		private final List<RequestMatcherEntry> mappings = new ArrayList<>();

		private Builder() {
		}

		public Builder add(RequestMatcher matcher,
				AuthorizationManager<HttpServletRequest> manager) {
			this.mappings.add(new RequestMatcherEntry(matcher, manager));
			return this;
		}

		public SimpleRequestMatcherDelegatingAuthorizationManager build() {
			if (this.mappings.isEmpty()) {
				throw new IllegalStateException(
						"At least one RequestMatcher -> AuthorizationManager mapping is required");
			}
			return new SimpleRequestMatcherDelegatingAuthorizationManager(this.mappings);
		}
	}

}
```

#### `simple-security-config/.../configurers/SimpleAuthorizeHttpRequestsConfigurer.java` [NEW]

```java
package simple.security.config.annotation.web.configurers;

import java.util.ArrayList;
import java.util.List;

import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.authorization.SimpleAuthenticatedAuthorizationManager;
import simple.security.authorization.SimpleAuthorityAuthorizationManager;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.access.intercept.SimpleAuthorizationFilter;
import simple.security.web.access.intercept.SimpleRequestMatcherDelegatingAuthorizationManager;
import simple.security.web.util.matcher.RequestMatcher;
import simple.security.web.util.matcher.SimpleAnyRequestMatcher;
import simple.security.web.util.matcher.SimplePathPatternRequestMatcher;

public class SimpleAuthorizeHttpRequestsConfigurer
		extends SimpleAbstractHttpConfigurer<SimpleAuthorizeHttpRequestsConfigurer> {

	private final AuthorizationManagerRequestMatcherRegistry registry =
			new AuthorizationManagerRequestMatcherRegistry();

	public AuthorizationManagerRequestMatcherRegistry getRegistry() {
		return this.registry;
	}

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		SimpleRequestMatcherDelegatingAuthorizationManager authorizationManager =
				this.registry.createAuthorizationManager();
		SimpleAuthorizationFilter filter = new SimpleAuthorizationFilter(authorizationManager);
		builder.addFilter(filter);
	}

	public static final class AuthorizationManagerRequestMatcherRegistry {

		private final SimpleRequestMatcherDelegatingAuthorizationManager.Builder managerBuilder =
				SimpleRequestMatcherDelegatingAuthorizationManager.builder();
		private int mappingCount = 0;

		private AuthorizationManagerRequestMatcherRegistry() {
		}

		public AuthorizedUrl requestMatchers(String... patterns) {
			List<RequestMatcher> matchers = new ArrayList<>(patterns.length);
			for (String pattern : patterns) {
				matchers.add(new SimplePathPatternRequestMatcher(pattern));
			}
			return new AuthorizedUrl(matchers);
		}

		public AuthorizedUrl requestMatchers(RequestMatcher... matchers) {
			return new AuthorizedUrl(List.of(matchers));
		}

		public AuthorizedUrl anyRequest() {
			return new AuthorizedUrl(List.of(SimpleAnyRequestMatcher.INSTANCE));
		}

		SimpleRequestMatcherDelegatingAuthorizationManager createAuthorizationManager() {
			if (this.mappingCount == 0) {
				throw new IllegalStateException(
						"At least one authorization rule must be configured via "
								+ "authorizeHttpRequests().");
			}
			return this.managerBuilder.build();
		}

		public final class AuthorizedUrl {
			private final List<RequestMatcher> matchers;

			private AuthorizedUrl(List<RequestMatcher> matchers) {
				this.matchers = matchers;
			}

			public AuthorizationManagerRequestMatcherRegistry permitAll() {
				return access((auth, request) -> new AuthorizationDecision(true));
			}

			public AuthorizationManagerRequestMatcherRegistry denyAll() {
				return access((auth, request) -> new AuthorizationDecision(false));
			}

			public AuthorizationManagerRequestMatcherRegistry hasRole(String role) {
				return access(SimpleAuthorityAuthorizationManager.hasRole(role));
			}

			public AuthorizationManagerRequestMatcherRegistry hasAnyRole(String... roles) {
				return access(SimpleAuthorityAuthorizationManager.hasAnyRole(roles));
			}

			public AuthorizationManagerRequestMatcherRegistry hasAuthority(String authority) {
				return access(SimpleAuthorityAuthorizationManager.hasAuthority(authority));
			}

			public AuthorizationManagerRequestMatcherRegistry hasAnyAuthority(String... authorities) {
				return access(SimpleAuthorityAuthorizationManager.hasAnyAuthority(authorities));
			}

			public AuthorizationManagerRequestMatcherRegistry authenticated() {
				return access(SimpleAuthenticatedAuthorizationManager.authenticated());
			}

			public AuthorizationManagerRequestMatcherRegistry access(
					AuthorizationManager<jakarta.servlet.http.HttpServletRequest> manager) {
				for (RequestMatcher matcher : this.matchers) {
					managerBuilder.add(matcher, manager);
					mappingCount++;
				}
				return AuthorizationManagerRequestMatcherRegistry.this;
			}
		}
	}

}
```

### Modified Files

#### `simple-security-config/.../builders/SimpleHttpSecurity.java` [MODIFIED]

**Change 1**: Filter order map — renamed `"AuthorizationFilter"` to `"SimpleAuthorizationFilter"`:
```java
FILTER_ORDER_MAP.put("SimpleAuthorizationFilter", order);  // was "AuthorizationFilter"
```

**Change 2**: Added import:
```java
import simple.security.config.annotation.web.configurers.SimpleAuthorizeHttpRequestsConfigurer;
```

**Change 3**: Added DSL method:
```java
public SimpleHttpSecurity authorizeHttpRequests(
        Customizer<SimpleAuthorizeHttpRequestsConfigurer.AuthorizationManagerRequestMatcherRegistry>
            authorizeHttpRequestsCustomizer) {
    SimpleAuthorizeHttpRequestsConfigurer configurer =
            getOrApply(new SimpleAuthorizeHttpRequestsConfigurer());
    authorizeHttpRequestsCustomizer.customize(configurer.getRegistry());
    return this;
}
```
