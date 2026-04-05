# Chapter 4: Security Filter Chain

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Create request matchers:
RequestMatcher apiMatcher = new SimplePathPatternRequestMatcher("/api/**");
RequestMatcher anyMatcher = SimpleAnyRequestMatcher.INSTANCE;

// Build security filter chains with their filters:
SecurityFilterChain apiChain = new SimpleDefaultSecurityFilterChain(
    apiMatcher, List.of(authFilter, authorizationFilter));
SecurityFilterChain defaultChain = new SimpleDefaultSecurityFilterChain(
    anyMatcher, List.of(loginFilter));

// FilterChainProxy dispatches to the first matching chain:
SimpleFilterChainProxy proxy = new SimpleFilterChainProxy(List.of(apiChain, defaultChain));

// Wrap with DelegatingFilterProxy (the only filter the servlet container sees):
SimpleDelegatingFilterProxy delegating = new SimpleDelegatingFilterProxy(proxy);

// Now every HTTP request goes through:
// servlet container → DelegatingFilterProxy → FilterChainProxy → matching chain's filters
```

Questions to guide your thinking:
- Why does `FilterChainProxy` select only the *first* matching chain rather than running all chains that match?
- Why is there a `DelegatingFilterProxy` at all — why not register `FilterChainProxy` directly with the servlet container?
- Why does `FilterChainProxy` clear the `SecurityContext` after every request?
- What happens to the request if no `SecurityFilterChain` matches?

---

## The API Contract

Feature 4 introduces the **security filter chain** — the mechanism that bridges the
servlet container to Spring Security by intercepting every HTTP request and routing it
through the correct set of security filters.

There are three layers in the public API:

### Layer 1: Request matching (`RequestMatcher`)

```java
// simple/security/web/util/matcher/RequestMatcher.java
@FunctionalInterface
public interface RequestMatcher {
    boolean matches(HttpServletRequest request);
}
```

`RequestMatcher` is the strategy for deciding whether a security chain applies to a
given request. Different implementations match by URL pattern, HTTP method, or any
other request attribute.

### Layer 2: The filter chain (`SecurityFilterChain`)

```java
// simple/security/web/SecurityFilterChain.java
public interface SecurityFilterChain {
    boolean matches(HttpServletRequest request);
    List<Filter> getFilters();
}
```

`SecurityFilterChain` pairs a `RequestMatcher` with an ordered list of servlet `Filter`s.
`FilterChainProxy` uses this interface to decide which filters to apply.

### Layer 3: The dispatcher (`SimpleFilterChainProxy`)

```java
// simple/security/web/SimpleFilterChainProxy.java
public class SimpleFilterChainProxy implements Filter {
    public SimpleFilterChainProxy(List<SecurityFilterChain> filterChains) { ... }
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) { ... }
}
```

`SimpleFilterChainProxy` is a servlet `Filter` that holds a list of `SecurityFilterChain`s.
On each request, it iterates the chains, picks the first match, and executes that chain's
security filters via an internal `VirtualFilterChain`.

### Client usage example

```java
// Build a filter chain for API endpoints
SecurityFilterChain apiChain = new SimpleDefaultSecurityFilterChain(
    new SimplePathPatternRequestMatcher("/api/**"),
    List.of(bearerTokenFilter, authorizationFilter));

// Build a catch-all chain for everything else
SecurityFilterChain defaultChain = new SimpleDefaultSecurityFilterChain(
    SimpleAnyRequestMatcher.INSTANCE,
    List.of(formLoginFilter, csrfFilter, authorizationFilter));

// Wire into FilterChainProxy
SimpleFilterChainProxy filterChainProxy = new SimpleFilterChainProxy(
    List.of(apiChain, defaultChain));

// Register with the servlet container through DelegatingFilterProxy
SimpleDelegatingFilterProxy proxy = new SimpleDelegatingFilterProxy(filterChainProxy);
```

---

## Client Usage & Tests

The tests in `SecurityFilterChainApiTest` define the full behavioral contract.
Key scenarios:

```java
@Test
void shouldMatchRequest_WhenPathMatches() {
    SecurityFilterChain chain = new SimpleDefaultSecurityFilterChain(
        new SimplePathPatternRequestMatcher("/api/**"),
        List.of(new NoOpFilter()));
    assertThat(chain.matches(new MockHttpServletRequest("GET", "/api/users"))).isTrue();
    assertThat(chain.matches(new MockHttpServletRequest("GET", "/web/home"))).isFalse();
}

@Test
void shouldSelectFirstMatchingChain_WhenMultipleChainsConfigured() {
    SecurityFilterChain apiChain = new SimpleDefaultSecurityFilterChain(
        new SimplePathPatternRequestMatcher("/api/**"),
        List.of(new LoggingFilter("api-filter", log)));
    SecurityFilterChain webChain = new SimpleDefaultSecurityFilterChain(
        new SimplePathPatternRequestMatcher("/web/**"),
        List.of(new LoggingFilter("web-filter", log)));
    SimpleFilterChainProxy proxy = new SimpleFilterChainProxy(List.of(apiChain, webChain));

    proxy.doFilter(new MockHttpServletRequest("GET", "/api/users"), response, chain);
    assertThat(log).containsExactly("api-filter"); // Only the API chain ran
}

@Test
void shouldClearSecurityContext_AfterRequestProcessing() {
    // A filter sets SecurityContext during processing...
    proxy.doFilter(request, response, chain);
    // After the proxy finishes, SecurityContext is cleared
    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
}

@Test
void shouldProcessFullPipeline_WhenEndToEnd() {
    // DelegatingFilterProxy → FilterChainProxy → security filters → servlet
    delegatingProxy.doFilter(request, response, servletChain);
    assertThat(log).containsExactly("auth-filter", "authz-filter", "servlet");
}
```

---

## Implementing the Call Chain

### Call chain recap

```
HTTP Request: POST /api/orders
  │
  ├─► [Bridge] SimpleDelegatingFilterProxy.doFilter()
  │     Delegates to the target Filter bean (FilterChainProxy)
  │
  ├─► [Dispatch] SimpleFilterChainProxy.doFilter()
  │     Sets FILTER_APPLIED attribute (prevents re-entrant processing)
  │     Iterates SecurityFilterChain list, selects first match
  │
  ├─► [Virtual Chain] VirtualFilterChain.doFilter()
  │     Iterates through matched security filters in order:
  │     ├─► authFilter.doFilter(req, res, virtualChain)
  │     ├─► authorizationFilter.doFilter(req, res, virtualChain)
  │     └─► (all filters done) → originalChain.doFilter(req, res)
  │
  └─► [Cleanup] FilterChainProxy finally block
        SecurityContextHolder.clearContext()
        Remove FILTER_APPLIED attribute
```

### 4a. Matching Layer: `RequestMatcher`

This is a `@FunctionalInterface` — in tests or simple setups, you can match with a lambda:

```java
RequestMatcher adminOnly = req -> req.getRequestURI().startsWith("/admin");
```

Two implementations cover the common cases:

**`SimpleAnyRequestMatcher`** — the catch-all. A singleton that always returns `true`.
Used as the last chain to ensure every request is processed by some security filter set.

**`SimplePathPatternRequestMatcher`** — matches URL patterns:

```java
public boolean matches(HttpServletRequest request) {
    String requestPath = getRequestPath(request);
    if (pattern.endsWith("/**")) {
        String prefix = pattern.substring(0, pattern.length() - 3);
        return requestPath.equals(prefix) || requestPath.startsWith(prefix + "/");
    }
    return pattern.equals(requestPath);
}
```

The path is extracted from `getRequestURI()` with the context path stripped, so
`/myapp/api/users` in a context `/myapp` matches the pattern `/api/**`.

The real `PathPatternRequestMatcher` uses Spring's `PathPattern` parser (supporting
`{variables}`, `*`, `**`, regex segments), and also supports HTTP method constraints.
Our simplified version supports exact paths and `/**` prefix wildcards, which covers
the vast majority of security configuration patterns.

### 4b. API Layer: `SecurityFilterChain` and `SimpleDefaultSecurityFilterChain`

`SecurityFilterChain` is a simple interface — `matches()` and `getFilters()`.

`SimpleDefaultSecurityFilterChain` pairs a `RequestMatcher` with a defensive copy of
the filter list:

```java
public SimpleDefaultSecurityFilterChain(RequestMatcher requestMatcher, List<Filter> filters) {
    this.requestMatcher = requestMatcher;
    this.filters = new ArrayList<>(filters); // defensive copy
}
```

The defensive copy ensures that modifying the original list after construction doesn't
affect the chain. The real `DefaultSecurityFilterChain` does the same thing.

### 4c. Dispatch Layer: `SimpleFilterChainProxy`

The heart of the feature. `doFilter()` follows a three-step pattern:

**Step 1: Guard against re-entrant processing.** A request attribute `FILTER_APPLIED`
tracks whether the proxy is already running. Forward dispatches (e.g., Spring MVC error
handling) would re-enter `doFilter()` — the guard ensures security filters run only once.

```java
if (httpRequest.getAttribute(FILTER_APPLIED) != null) {
    doFilterInternal(httpRequest, httpResponse, chain);
    return;  // No cleanup — the outer invocation handles it
}
```

**Step 2: Find and execute the matching chain.** `doFilterInternal()` iterates the
`SecurityFilterChain` list and returns the filters from the first match:

```java
for (SecurityFilterChain filterChain : this.filterChains) {
    if (filterChain.matches(request)) {
        return filterChain.getFilters();
    }
}
return null; // no match
```

If no chain matches, the request bypasses all security filters and continues directly
to the original servlet filter chain.

**Step 3: Clean up.** The `finally` block clears the `SecurityContext` and removes the
`FILTER_APPLIED` attribute. This is critical — without clearing, a `ThreadLocal` leak
would cause one user's authentication to bleed into another user's request when threads
are reused (servlet container thread pools).

### 4d. The `VirtualFilterChain`

The inner `VirtualFilterChain` is how security filters are chained together. It implements
`FilterChain` and maintains a position counter:

```java
public void doFilter(ServletRequest request, ServletResponse response) {
    if (currentPosition == filters.size()) {
        originalChain.doFilter(request, response);  // Fall through to servlet
        return;
    }
    Filter nextFilter = filters.get(currentPosition);
    currentPosition++;
    nextFilter.doFilter(request, response, this);   // Pass `this` as the chain
}
```

Each security filter calls `chain.doFilter(request, response)` to invoke the next filter.
When all security filters have executed, the `VirtualFilterChain` falls through to the
original servlet `FilterChain`. This is the same pattern the real `FilterChainProxy` uses.

### 4e. Bridge Layer: `SimpleDelegatingFilterProxy`

`SimpleDelegatingFilterProxy` is the only filter registered with the servlet container.
It simply delegates to its target `Filter` (a `FilterChainProxy`):

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    this.delegate.doFilter(request, response, chain);
}
```

**Why this indirection?** In a real application, the servlet container initializes filters
during startup, before the Spring `ApplicationContext` is ready. The `DelegatingFilterProxy`
is created early (by the container) and lazily looks up the actual security filter bean
(the `FilterChainProxy`) from the Spring context on the first request. Our simplified
version skips the lazy lookup and accepts the delegate directly.

---

## Try It Yourself

**Challenge 1**: Add a third `SecurityFilterChain` that matches only `POST` requests
to `/api/**`. You'll need to implement a custom `RequestMatcher` that checks both the
HTTP method and the URL pattern.

**Challenge 2**: Write a filter that counts how many requests pass through it (use an
`AtomicInteger`). Register it in a chain and verify the counter increments after each
request through `FilterChainProxy`.

**Challenge 3**: What happens if a security filter does *not* call `chain.doFilter()`?
Write a test with a "blocking" filter (one that writes a 403 response and stops). Verify
that subsequent filters and the original chain are never invoked.

**Challenge 4**: Implement a `SimpleOrRequestMatcher` that matches if *any* of its
child matchers match, and a `SimpleAndRequestMatcher` that matches only if *all* match.

---

## Why This Works

**The Composite pattern**: `SecurityFilterChain` composes a `RequestMatcher` with a
`List<Filter>`. `SimpleFilterChainProxy` composes multiple `SecurityFilterChain`s. This
nesting lets you declare completely different security configurations for different
URL spaces (e.g., stateless JWT for `/api/**`, session-based for `/web/**`) — something
that's impossible with a flat list of filters.

**The Proxy pattern**: `SimpleDelegatingFilterProxy` is a classic proxy — it has the
same interface as the delegate (`Filter`), but adds the bridge between the servlet
container's lifecycle and Spring's bean lifecycle. The servlet container never knows
that Spring Security exists.

**The Chain of Responsibility pattern**: `VirtualFilterChain` is the same pattern we saw
in `SimpleProviderManager` (Feature 3) — each filter in the chain decides whether to
continue by calling `chain.doFilter()`, or halt by writing a response directly.

**Thread safety via cleanup**: The `finally` block in `doFilter()` ensures
`SecurityContextHolder.clearContext()` runs even if a filter throws an exception. This
prevents ThreadLocal contamination in servlet container thread pools, which would be a
critical security vulnerability.

---

## What We Enhanced

| Layer | Feature 1 | Feature 2 | Feature 3 | Feature 4 (This Chapter) |
|-------|-----------|-----------|-----------|--------------------------|
| Crypto | `PasswordEncoder` / BCrypt | — | Used by `SimpleDaoAuthenticationProvider` | — |
| Identity model | — | `Authentication`, `UserDetails`, `SecurityContext` | Used as pipeline inputs/outputs | `SecurityContextHolder.clearContext()` called by `FilterChainProxy` |
| Pipeline | — | — | `AuthenticationManager`, providers, user store | — |
| Filter dispatch | — | — | — | `SecurityFilterChain`, `SimpleFilterChainProxy`, `VirtualFilterChain` |
| Request matching | — | — | — | `RequestMatcher`, `SimplePathPatternRequestMatcher`, `SimpleAnyRequestMatcher` |
| Servlet bridge | — | — | — | `SimpleDelegatingFilterProxy` |

Feature 4 is the first web-layer feature. It bridges the servlet container (HTTP) to the
security model built in Features 1-3. After this chapter, the framework can intercept
HTTP requests and route them through security filters — but it doesn't yet have any
*specific* security filters (authentication, authorization, CSRF). Those come in Features 5-8.

---

## Connection to Real Framework

| Our Class | Real Framework Class | Key Difference |
|-----------|---------------------|----------------|
| `RequestMatcher` | `o.s.s.web.util.matcher.RequestMatcher` | No `matcher()` method returning `MatchResult` with URI variables |
| `SimpleAnyRequestMatcher` | `o.s.s.web.util.matcher.AnyRequestMatcher` | Identical behavior |
| `SimplePathPatternRequestMatcher` | `o.s.s.web.servlet.util.matcher.PathPatternRequestMatcher` | Simple prefix/exact matching only (no `PathPattern` parser, no HTTP method constraint, no `Builder`, no `basePath`, no URI variable extraction) |
| `SecurityFilterChain` | `o.s.s.web.SecurityFilterChain` | Identical interface |
| `SimpleDefaultSecurityFilterChain` | `o.s.s.web.DefaultSecurityFilterChain` | No `BeanNameAware`/`BeanFactoryAware` (used for diagnostic logging in the real framework) |
| `SimpleFilterChainProxy` | `o.s.s.web.FilterChainProxy` | No `HttpFirewall` (request/response wrapping for XSS protection), no `RequestRejectedHandler`, no `FilterChainDecorator` (observability hooks), no `ThrowableAnalyzer` |
| `SimpleDelegatingFilterProxy` | `o.s.web.filter.DelegatingFilterProxy` | Accepts delegate directly instead of lazy bean lookup from `WebApplicationContext`; lives in our security module instead of Spring Framework's `spring-web` |

### Real framework: the firewall and observability layers

The real `FilterChainProxy` wraps every request/response through an `HttpFirewall`
(default: `StrictHttpFirewall`) before matching or executing filters. The firewall
normalizes the request path, blocks path traversal attacks (`/../`), rejects
non-printable characters, and guards against HTTP response splitting:

```
FilterChainProxy.doFilter()
  ├── firewall.getFirewalledRequest(request)   ← normalize + validate
  ├── firewall.getFirewalledResponse(response) ← response wrapping
  ├── getFilters(firewallRequest)              ← match against chains
  └── filterChainDecorator.decorate(chain, filters)
        ← observability hook (traces, metrics)
```

The `FilterChainDecorator` (since Spring Security 6.0) wraps the `VirtualFilterChain`
to enable observability integration — each filter execution can be traced via Micrometer.

Our simplified version omits both the firewall and the decorator. These are important
in production but orthogonal to understanding the core dispatch mechanism.

---

## Complete Code

### [NEW] `simple-security-web/src/main/java/simple/security/web/util/matcher/RequestMatcher.java`

```java
package simple.security.web.util.matcher;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Strategy interface for matching an {@link HttpServletRequest}.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.util.matcher.RequestMatcher}.
 */
@FunctionalInterface
public interface RequestMatcher {

	/**
	 * Decides whether the rule implemented by the strategy matches the supplied request.
	 * @param request the HTTP request to check
	 * @return {@code true} if the request matches, {@code false} otherwise
	 */
	boolean matches(HttpServletRequest request);

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/util/matcher/SimpleAnyRequestMatcher.java`

```java
package simple.security.web.util.matcher;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Matches any request unconditionally. Used as the catch-all matcher.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.util.matcher.AnyRequestMatcher}.
 */
public final class SimpleAnyRequestMatcher implements RequestMatcher {

	public static final RequestMatcher INSTANCE = new SimpleAnyRequestMatcher();

	private SimpleAnyRequestMatcher() {
	}

	@Override
	public boolean matches(HttpServletRequest request) {
		return true;
	}

	@Override
	public String toString() {
		return "any request";
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/util/matcher/SimplePathPatternRequestMatcher.java`

```java
package simple.security.web.util.matcher;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Matches requests against a URL path pattern using simple ant-style matching.
 *
 * <p>Supported patterns:
 * <ul>
 *   <li>{@code /exact/path} — matches only that exact path</li>
 *   <li>{@code /prefix/**} — matches the prefix and any path below it</li>
 * </ul>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.servlet.util.matcher.PathPatternRequestMatcher}.
 */
public final class SimplePathPatternRequestMatcher implements RequestMatcher {

	private final String pattern;

	public SimplePathPatternRequestMatcher(String pattern) {
		if (pattern == null || pattern.isEmpty()) {
			throw new IllegalArgumentException("Pattern must not be null or empty");
		}
		this.pattern = pattern;
	}

	@Override
	public boolean matches(HttpServletRequest request) {
		String requestPath = getRequestPath(request);
		if (pattern.endsWith("/**")) {
			String prefix = pattern.substring(0, pattern.length() - 3);
			return requestPath.equals(prefix) || requestPath.startsWith(prefix + "/");
		}
		return pattern.equals(requestPath);
	}

	private String getRequestPath(HttpServletRequest request) {
		String uri = request.getRequestURI();
		String contextPath = request.getContextPath();
		if (contextPath != null && !contextPath.isEmpty()) {
			return uri.substring(contextPath.length());
		}
		return uri;
	}

	public String getPattern() {
		return pattern;
	}

	@Override
	public String toString() {
		return pattern;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/SecurityFilterChain.java`

```java
package simple.security.web;

import java.util.List;

import jakarta.servlet.Filter;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Defines a filter chain that can be matched against an {@link HttpServletRequest}.
 *
 * <p>A {@code SecurityFilterChain} pairs a request matcher with an ordered list of
 * security filters. {@link SimpleFilterChainProxy} uses this interface to decide
 * which chain of filters applies to a given request.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.SecurityFilterChain}.
 */
public interface SecurityFilterChain {

	/**
	 * Decides whether this filter chain applies to the given request.
	 * @param request the incoming HTTP request
	 * @return {@code true} if this chain should handle the request
	 */
	boolean matches(HttpServletRequest request);

	/**
	 * Returns the ordered list of security {@link Filter}s in this chain.
	 * @return the filters (never {@code null})
	 */
	List<Filter> getFilters();

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/SimpleDefaultSecurityFilterChain.java`

```java
package simple.security.web;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

import jakarta.servlet.Filter;
import jakarta.servlet.http.HttpServletRequest;

import simple.security.web.util.matcher.RequestMatcher;

/**
 * Default implementation of {@link SecurityFilterChain} that pairs a
 * {@link RequestMatcher} with an ordered list of servlet {@link Filter}s.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.DefaultSecurityFilterChain}.
 */
public final class SimpleDefaultSecurityFilterChain implements SecurityFilterChain {

	private final RequestMatcher requestMatcher;

	private final List<Filter> filters;

	public SimpleDefaultSecurityFilterChain(RequestMatcher requestMatcher, Filter... filters) {
		this(requestMatcher, Arrays.asList(filters));
	}

	public SimpleDefaultSecurityFilterChain(RequestMatcher requestMatcher, List<Filter> filters) {
		this.requestMatcher = requestMatcher;
		this.filters = new ArrayList<>(filters);
	}

	@Override
	public boolean matches(HttpServletRequest request) {
		return this.requestMatcher.matches(request);
	}

	@Override
	public List<Filter> getFilters() {
		return this.filters;
	}

	public RequestMatcher getRequestMatcher() {
		return this.requestMatcher;
	}

	@Override
	public String toString() {
		return "SimpleDefaultSecurityFilterChain [requestMatcher=" + this.requestMatcher
				+ ", filters=" + this.filters + "]";
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/SimpleFilterChainProxy.java`

```java
package simple.security.web;

import java.io.IOException;
import java.util.List;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.context.SecurityContextHolder;

/**
 * Dispatches incoming HTTP requests to the first matching {@link SecurityFilterChain}
 * and executes its security filters in order before continuing to the original
 * servlet filter chain.
 *
 * <p>This is the single entry-point for all Spring Security request processing.
 * A {@link SimpleDelegatingFilterProxy} registered with the servlet container
 * delegates to this filter.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.FilterChainProxy}.
 */
public class SimpleFilterChainProxy implements Filter {

	private static final String FILTER_APPLIED = SimpleFilterChainProxy.class.getName() + ".APPLIED";

	private final List<SecurityFilterChain> filterChains;

	public SimpleFilterChainProxy(SecurityFilterChain chain) {
		this(List.of(chain));
	}

	public SimpleFilterChainProxy(List<SecurityFilterChain> filterChains) {
		this.filterChains = filterChains;
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest httpRequest = (HttpServletRequest) request;
		HttpServletResponse httpResponse = (HttpServletResponse) response;

		// Prevent re-entrant filtering (e.g. forward dispatches)
		if (httpRequest.getAttribute(FILTER_APPLIED) != null) {
			doFilterInternal(httpRequest, httpResponse, chain);
			return;
		}

		try {
			httpRequest.setAttribute(FILTER_APPLIED, Boolean.TRUE);
			doFilterInternal(httpRequest, httpResponse, chain);
		}
		finally {
			SecurityContextHolder.clearContext();
			httpRequest.removeAttribute(FILTER_APPLIED);
		}
	}

	private void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		List<Filter> filters = getFilters(request);
		if (filters == null || filters.isEmpty()) {
			// No security filter chain matched — continue to the original chain
			chain.doFilter(request, response);
			return;
		}
		VirtualFilterChain virtualChain = new VirtualFilterChain(chain, filters);
		virtualChain.doFilter(request, response);
	}

	private List<Filter> getFilters(HttpServletRequest request) {
		for (SecurityFilterChain filterChain : this.filterChains) {
			if (filterChain.matches(request)) {
				return filterChain.getFilters();
			}
		}
		return null;
	}

	/**
	 * Returns an unmodifiable view of the configured {@link SecurityFilterChain}s.
	 */
	public List<SecurityFilterChain> getFilterChains() {
		return List.copyOf(this.filterChains);
	}

	/**
	 * Internal filter chain that iterates through the security filters and then
	 * delegates to the original servlet filter chain.
	 *
	 * <p>Simplified version of {@code FilterChainProxy.VirtualFilterChain}.
	 */
	private static final class VirtualFilterChain implements FilterChain {

		private final FilterChain originalChain;

		private final List<Filter> filters;

		private int currentPosition = 0;

		VirtualFilterChain(FilterChain originalChain, List<Filter> filters) {
			this.originalChain = originalChain;
			this.filters = filters;
		}

		@Override
		public void doFilter(ServletRequest request, ServletResponse response)
				throws IOException, ServletException {
			if (this.currentPosition == this.filters.size()) {
				this.originalChain.doFilter(request, response);
				return;
			}
			Filter nextFilter = this.filters.get(this.currentPosition);
			this.currentPosition++;
			nextFilter.doFilter(request, response, this);
		}

	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/SimpleDelegatingFilterProxy.java`

```java
package simple.security.web;

import java.io.IOException;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;

/**
 * A servlet {@link Filter} that delegates to another {@link Filter}, acting as a bridge
 * between the servlet container and a Spring-managed security filter (typically a
 * {@link SimpleFilterChainProxy}).
 *
 * <p>In a real application, the servlet container knows only about this filter
 * (registered in {@code web.xml} or programmatically). It delegates every request
 * to the target filter bean looked up from the Spring {@code ApplicationContext}.
 *
 * <p>This simplified version accepts the delegate directly rather than performing
 * a bean lookup at runtime.
 *
 * <p>Simplified version of
 * {@code org.springframework.web.filter.DelegatingFilterProxy}.
 */
public class SimpleDelegatingFilterProxy implements Filter {

	private final String targetBeanName;

	private final Filter delegate;

	/**
	 * Creates a proxy that delegates to the given filter with the conventional
	 * bean name {@code "springSecurityFilterChain"}.
	 * @param delegate the target filter to delegate to
	 */
	public SimpleDelegatingFilterProxy(Filter delegate) {
		this("springSecurityFilterChain", delegate);
	}

	/**
	 * Creates a proxy that delegates to the given filter with the specified bean name.
	 * @param targetBeanName the name of the target bean (for diagnostics)
	 * @param delegate the target filter to delegate to
	 */
	public SimpleDelegatingFilterProxy(String targetBeanName, Filter delegate) {
		this.targetBeanName = targetBeanName;
		this.delegate = delegate;
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		this.delegate.doFilter(request, response, chain);
	}

	public String getTargetBeanName() {
		return this.targetBeanName;
	}

	@Override
	public void destroy() {
		this.delegate.destroy();
	}

}
```

### [NEW] `simple-security-web/src/test/java/simple/security/web/SecurityFilterChainApiTest.java`

> See `src/test/java/simple/security/web/SecurityFilterChainApiTest.java` in the repository.
> Full test file with 12 test cases covering: path matching, any-request matching, filter list,
> filter execution order, fall-through to original chain, first-matching-chain selection,
> no-match bypass, SecurityContext cleanup, DelegatingFilterProxy delegation, default bean name,
> and end-to-end pipeline (DelegatingFilterProxy → FilterChainProxy → chain → servlet).

### [NEW] `simple-security-web/src/test/java/simple/security/web/util/matcher/SimplePathPatternRequestMatcherTest.java`

> See `src/test/java/simple/security/web/util/matcher/SimplePathPatternRequestMatcherTest.java` in the repository.
> Full test file with 11 test cases covering: exact matching (equal, different, trailing segment),
> prefix wildcard matching (sub-paths, exact prefix, non-matching, root wildcard),
> context path stripping, null/empty pattern validation, toString, and AnyRequestMatcher.
