# Chapter 10: Session Management

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure session management via the DSL:
SimpleHttpSecurity http = new SimpleHttpSecurity();

// Stateless for REST APIs (no HttpSession created):
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS));

// Session-based with defaults (context persisted in HttpSession):
http.sessionManagement(Customizer.withDefaults());
http.httpBasic(Customizer.withDefaults());

SecurityFilterChain chain = http.build();

// A SimpleSecurityContextHolderFilter is added at position 100 (first filter).
// It intercepts every request and:
//   1. Loads SecurityContext from the SecurityContextRepository (e.g., HttpSession)
//   2. Sets it in SecurityContextHolder (ThreadLocal)
//   3. Continues the filter chain
//   4. In the finally block: clears SecurityContextHolder
//
// Authentication filters (HTTP Basic, form login) explicitly save the context
// to the repository after successful authentication.
```

Questions to guide your thinking:
- Why does `SecurityContextHolderFilter` only *load* the context but not *save* it back? What problems did the old auto-save approach cause?
- What's the difference between `NEVER` and `STATELESS`? Both prevent session creation, so why do we need both?
- Why must authentication filters explicitly call `securityContextRepository.saveContext()` instead of relying on automatic persistence?
- How does the shared object mechanism allow the session management configurer to communicate the repository to authentication configurers?
- Why is the `SecurityContextHolderFilter` positioned first (position 100) in the chain?

---

## The API Contract

Feature 10 adds session management — the ability to control how the `SecurityContext` is persisted between HTTP requests. Without session management, each request starts with an empty `SecurityContext`, and authentication state is lost after the filter chain completes.

### The DSL Entry Point

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity sessionManagement(
        Customizer<SimpleSessionManagementConfigurer> sessionManagementCustomizer) {
    sessionManagementCustomizer.customize(getOrApply(new SimpleSessionManagementConfigurer()));
    return this;
}
```

### What the client configures

| DSL Method | Purpose | Default |
|-----------|---------|---------|
| `sessionCreationPolicy(...)` | Controls session creation behavior | `IF_REQUIRED` |
| `securityContextRepository(...)` | Custom context persistence strategy | `SimpleHttpSessionSecurityContextRepository` |
| `disable()` | Turn off session management entirely | Enabled |

### Session creation policies

| Policy | Creates Session? | Uses Existing Session? | Use Case |
|--------|:---:|:---:|---------|
| `ALWAYS` | Yes, eagerly | Yes | Legacy apps that always need a session |
| `IF_REQUIRED` | Only when needed | Yes | **Default** for web apps with login |
| `NEVER` | No | Yes | Apps where sessions are created by other means |
| `STATELESS` | No | No | REST APIs with token-based auth |

### The behavioral contract

1. **Every request** -> `SecurityContextHolderFilter` loads the context from the repository
2. **Context found** -> set it in `SecurityContextHolder` (ThreadLocal)
3. **No context** -> `SecurityContextHolder` remains empty
4. **After authentication** -> authentication filter explicitly saves to the repository
5. **After request** -> `SecurityContextHolderFilter` clears the ThreadLocal in `finally`

---

## Client Usage & Tests

### Test 1: Session-based context persistence after authentication

```java
@Test
void shouldPersistSecurityContextInSessionAfterAuthentication() throws Exception {
    SimpleFilterChainProxy proxy = buildProxyWithHttpBasic(SessionCreationPolicy.IF_REQUIRED);

    // Authenticate via HTTP Basic
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/resource");
    request.addHeader("Authorization", basicAuth("user", "password"));
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Context should be saved in the session
    MockHttpSession session = (MockHttpSession) request.getSession(false);
    assertThat(session).isNotNull();
    Object savedContext = session.getAttribute(
            SimpleHttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY);
    assertThat(savedContext).isInstanceOf(SecurityContext.class);
    assertThat(((SecurityContext) savedContext).getAuthentication().getName()).isEqualTo("user");
}
```

### Test 2: Context restored from session on subsequent request

```java
@Test
void shouldRestoreSecurityContextFromSessionOnSubsequentRequest() throws Exception {
    SimpleFilterChainProxy proxy = buildProxyWithHttpBasic(SessionCreationPolicy.IF_REQUIRED);

    // First request: authenticate and get session
    MockHttpServletRequest request1 = new MockHttpServletRequest("GET", "/resource");
    request1.addHeader("Authorization", basicAuth("user", "password"));
    MockHttpServletResponse response1 = new MockHttpServletResponse();
    proxy.doFilter(request1, response1, new MockFilterChain());
    MockHttpSession session = (MockHttpSession) request1.getSession(false);

    // Second request: no credentials, but same session
    MockHttpServletRequest request2 = new MockHttpServletRequest("GET", "/resource");
    request2.setSession(session);

    Authentication[] captured = new Authentication[1];
    FilterChain capturingChain = (req, res) -> {
        captured[0] = SecurityContextHolder.getContext().getAuthentication();
    };
    proxy.doFilter(request2, new MockHttpServletResponse(), capturingChain);

    // SecurityContext was loaded from the session
    assertThat(captured[0]).isNotNull();
    assertThat(captured[0].getName()).isEqualTo("user");
}
```

### Test 3: Stateless mode prevents session creation

```java
@Test
void shouldNotCreateSessionWhenStateless() throws Exception {
    SimpleFilterChainProxy proxy = buildProxyWithHttpBasic(SessionCreationPolicy.STATELESS);

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/resource");
    request.addHeader("Authorization", basicAuth("user", "password"));
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // No session should be created
    assertThat(request.getSession(false)).isNull();
}
```

### Test 4: Stateless mode ignores existing session context

```java
@Test
void shouldNotRestoreContextFromSessionWhenStateless() throws Exception {
    // Pre-populate a session with a SecurityContext
    MockHttpSession session = new MockHttpSession();
    SecurityContextImpl preExistingContext = new SecurityContextImpl(
            UsernamePasswordAuthenticationToken.authenticated(
                    "admin", null, List.of(new SimpleGrantedAuthority("ROLE_ADMIN"))));
    session.setAttribute(
            SimpleHttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
            preExistingContext);

    SimpleFilterChainProxy proxy = buildProxyWithHttpBasic(SessionCreationPolicy.STATELESS);

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/resource");
    request.setSession(session);

    Authentication[] captured = new Authentication[1];
    FilterChain capturingChain = (req, res) -> {
        captured[0] = SecurityContextHolder.getContext().getAuthentication();
    };
    proxy.doFilter(request, new MockHttpServletResponse(), capturingChain);

    // STATELESS ignores the session — context is null
    assertThat(captured[0]).isNull();
}
```

### Test 5: NEVER policy reads existing session but doesn't create new ones

```java
@Test
void shouldReadFromExistingSessionWhenNever() throws Exception {
    // Pre-populate session
    MockHttpSession session = new MockHttpSession();
    SecurityContextImpl existingContext = new SecurityContextImpl(
            UsernamePasswordAuthenticationToken.authenticated(
                    "admin", null, List.of(new SimpleGrantedAuthority("ROLE_ADMIN"))));
    session.setAttribute(
            SimpleHttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
            existingContext);

    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.authenticationManager(createAuthManager())
            .sessionManagement(sm -> sm
                    .sessionCreationPolicy(SessionCreationPolicy.NEVER));

    SecurityFilterChain chain = http.build();
    SimpleFilterChainProxy proxy = new SimpleFilterChainProxy(chain);

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/resource");
    request.setSession(session);

    Authentication[] captured = new Authentication[1];
    FilterChain capturingChain = (req, res) -> {
        captured[0] = SecurityContextHolder.getContext().getAuthentication();
    };
    proxy.doFilter(request, new MockHttpServletResponse(), capturingChain);

    // NEVER still reads from an existing session
    assertThat(captured[0]).isNotNull();
    assertThat(captured[0].getName()).isEqualTo("admin");
}
```

### Test 6: Filter chain ordering

```java
@Test
void shouldPlaceSecurityContextHolderFilterBeforeCsrfFilter() throws Exception {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.sessionManagement(Customizer.withDefaults())
            .csrf(Customizer.withDefaults());

    SecurityFilterChain chain = http.build();

    // Position 100 (context) before position 200 (CSRF)
    assertThat(chain.getFilters()).hasSize(2);
    assertThat(chain.getFilters().get(0)).isInstanceOf(SimpleSecurityContextHolderFilter.class);
    assertThat(chain.getFilters().get(1)).isInstanceOf(SimpleCsrfFilter.class);
}
```

---

## Implementing the Call Chain

### Architecture Overview

```
Client DSL                        Configurer Lifecycle                    Request-Time Filter Chain
─────────                         ────────────────────                    ─────────────────────────

http.sessionManagement(           init():                                 Request arrives
  session -> session                resolve repository                         │
    .sessionCreationPolicy(           based on policy                           ▼
      STATELESS))                   register as shared object              ┌─────────────────────────────┐
                                                                          │ SimpleSecurityContextHolder  │
                                  configure():                            │         Filter               │
                                    create filter with                    │                              │
                                    repository                           │ 1. Load context from repo    │
                                    add at position 100                  │ 2. Set in SecurityContextHolder│
                                                                          │ 3. chain.doFilter()          │
http.httpBasic(...)               configure():                            │ 4. finally: clearContext()   │
                                    read repository from                 └──────────┬──────────────────┘
                                    shared objects                                  │
                                    inject into filter                              ▼
                                                                          ┌─────────────────────────────┐
                                                                          │ SimpleBasicAuthFilter        │
                                                                          │                              │
                                                                          │ On success:                  │
                                                                          │   set SecurityContext         │
                                                                          │   repo.saveContext(...)      │
                                                                          └─────────────────────────────┘
```

### Layer 1: The Repository Interface (API surface)

The first thing we need is a strategy interface for persisting the `SecurityContext` between requests. The interface has three methods: load, save, and check.

```java
// SecurityContextRepository.java — the strategy interface
public interface SecurityContextRepository {

    SecurityContext loadContext(HttpServletRequest request);

    void saveContext(SecurityContext context, HttpServletRequest request,
                     HttpServletResponse response);

    boolean containsContext(HttpServletRequest request);
}
```

**Key design decision:** `saveContext` is NOT called automatically by the filter. Authentication mechanisms must call it explicitly. This prevents accidental persistence of empty or partially-modified contexts.

### Layer 2: The Session Repository (persistence implementation)

The default implementation stores the `SecurityContext` in the `HttpSession` under the key `"SPRING_SECURITY_CONTEXT"`:

```java
// SimpleHttpSessionSecurityContextRepository.java
public class SimpleHttpSessionSecurityContextRepository implements SecurityContextRepository {

    public static final String SPRING_SECURITY_CONTEXT_KEY = "SPRING_SECURITY_CONTEXT";

    private boolean allowSessionCreation = true;

    @Override
    public SecurityContext loadContext(HttpServletRequest request) {
        HttpSession session = request.getSession(false);  // don't create session
        if (session == null) {
            return null;
        }
        Object contextObj = session.getAttribute(SPRING_SECURITY_CONTEXT_KEY);
        if (contextObj instanceof SecurityContext securityContext) {
            return securityContext;
        }
        return null;
    }

    @Override
    public void saveContext(SecurityContext context, HttpServletRequest request,
                            HttpServletResponse response) {
        if (context.getAuthentication() == null) {
            // Empty context — remove from session
            HttpSession session = request.getSession(false);
            if (session != null) {
                session.removeAttribute(SPRING_SECURITY_CONTEXT_KEY);
            }
            return;
        }
        // Save to session (creating if allowSessionCreation is true)
        HttpSession session = request.getSession(this.allowSessionCreation);
        if (session != null) {
            session.setAttribute(SPRING_SECURITY_CONTEXT_KEY, context);
        }
    }

    @Override
    public boolean containsContext(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        return session != null
                && session.getAttribute(SPRING_SECURITY_CONTEXT_KEY) != null;
    }
}
```

**Critical detail:** `loadContext` calls `request.getSession(false)` — it never creates a session just to check for a context. This is how `NEVER` policy works: the repository can read from an existing session but won't create one.

### Layer 3: The Filter (request-time loading)

The `SimpleSecurityContextHolderFilter` is the first filter in the chain (position 100). It bridges the repository and the ThreadLocal:

```java
// SimpleSecurityContextHolderFilter.java
public class SimpleSecurityContextHolderFilter implements Filter {

    private static final String FILTER_APPLIED =
            SimpleSecurityContextHolderFilter.class.getName() + ".APPLIED";

    private final SecurityContextRepository securityContextRepository;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                          FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // Prevent double application (e.g., during a forward)
        if (httpRequest.getAttribute(FILTER_APPLIED) != null) {
            chain.doFilter(request, response);
            return;
        }
        httpRequest.setAttribute(FILTER_APPLIED, Boolean.TRUE);

        try {
            SecurityContext context = securityContextRepository.loadContext(httpRequest);
            if (context != null) {
                SecurityContextHolder.setContext(context);
            }
            chain.doFilter(request, response);
        }
        finally {
            SecurityContextHolder.clearContext();
            httpRequest.removeAttribute(FILTER_APPLIED);
        }
    }
}
```

**Why `finally`?** The context MUST be cleared after every request. If it isn't, the thread returns to the pool still holding the previous user's authentication — a security vulnerability in server environments that reuse threads.

### Layer 4: The SessionCreationPolicy (configuration)

A simple enum that controls which repository implementation is used:

```java
// SessionCreationPolicy.java
public enum SessionCreationPolicy {
    ALWAYS,       // Always create a session
    IF_REQUIRED,  // Create only when needed (default)
    NEVER,        // Never create, but use existing
    STATELESS     // Never create or use sessions
}
```

### Layer 5: The Configurer (DSL integration)

The `SimpleSessionManagementConfigurer` ties everything together during the build lifecycle:

```java
// SimpleSessionManagementConfigurer.java
public class SimpleSessionManagementConfigurer
        extends SimpleAbstractHttpConfigurer<SimpleSessionManagementConfigurer> {

    private SessionCreationPolicy sessionCreationPolicy = SessionCreationPolicy.IF_REQUIRED;
    private SecurityContextRepository securityContextRepository;

    @Override
    public void init(SimpleHttpSecurity builder) throws Exception {
        // Register the repository as a shared object so authentication
        // configurers can access it during their configure() phase
        SecurityContextRepository repository = resolveSecurityContextRepository();
        builder.setSharedObject(SecurityContextRepository.class, repository);
    }

    @Override
    public void configure(SimpleHttpSecurity builder) throws Exception {
        SecurityContextRepository repository =
                builder.getSharedObject(SecurityContextRepository.class);
        SimpleSecurityContextHolderFilter filter =
                new SimpleSecurityContextHolderFilter(repository);
        builder.addFilter(filter);
    }

    private SecurityContextRepository resolveSecurityContextRepository() {
        if (this.securityContextRepository != null) {
            return this.securityContextRepository;
        }
        if (this.sessionCreationPolicy == SessionCreationPolicy.STATELESS) {
            return new NullSecurityContextRepository();  // never reads/writes
        }
        SimpleHttpSessionSecurityContextRepository repository =
                new SimpleHttpSessionSecurityContextRepository();
        if (this.sessionCreationPolicy == SessionCreationPolicy.NEVER) {
            repository.setAllowSessionCreation(false);
        }
        return repository;
    }
}
```

### Layer 6: Wiring to Authentication Filters (modification)

Authentication filters were modified to explicitly save the context to the repository after successful authentication. This is the "explicit save" pattern:

```java
// SimpleAbstractAuthenticationProcessingFilter.java — the change
protected void successfulAuthentication(HttpServletRequest request,
        HttpServletResponse response, Authentication authResult)
        throws IOException, ServletException {
    SecurityContext context = SecurityContextHolder.createEmptyContext();
    context.setAuthentication(authResult);
    SecurityContextHolder.setContext(context);

    // NEW: Explicitly save to the repository (e.g., HttpSession)
    if (this.securityContextRepository != null) {
        this.securityContextRepository.saveContext(context, request, response);
    }

    this.successHandler.onAuthenticationSuccess(request, response, authResult);
}
```

The form login and HTTP basic configurers wire the repository from shared objects:

```java
// SimpleFormLoginConfigurer.configure() — the change
SecurityContextRepository contextRepository =
        builder.getSharedObject(SecurityContextRepository.class);
if (contextRepository != null) {
    this.authFilter.setSecurityContextRepository(contextRepository);
}
```

---

## Try It Yourself

1. **Redis repository**: Implement a `SecurityContextRepository` that stores contexts in Redis instead of the HttpSession. What serialization strategy would you use?

2. **Session fixation protection**: When a user logs in, the session ID should change to prevent session fixation attacks. Where in the call chain would you add `request.changeSessionId()`?

3. **Concurrent session control**: How would you limit a user to at most N active sessions? What data structure would you need to track active sessions?

---

## Why This Works

### The Strategy Pattern
`SecurityContextRepository` is a classic Strategy — it decouples *what* gets stored (SecurityContext) from *where* it's stored (session, cookie, header, database). The `STATELESS` policy simply uses a null-object strategy (`NullSecurityContextRepository`) that does nothing.

### The Shared Object Communication Pattern
The configurer lifecycle enables cross-configurer communication without coupling:
1. `SessionManagementConfigurer.init()` registers the repository as a shared object
2. `FormLoginConfigurer.configure()` reads the repository from shared objects
3. Neither configurer references the other

This is the same pattern the real framework uses — shared objects are Spring Security's internal service locator.

### The "Load Only, Don't Save" Design
The old `SecurityContextPersistenceFilter` (pre-Spring Security 5.7) auto-saved the context after every request. This caused subtle bugs:
- An anonymous request could accidentally persist an empty context
- A partially-modified context from a failed authentication could be saved
- It was impossible to control *when* persistence happened

The modern approach is explicit: `SecurityContextHolderFilter` only loads and clears. Authentication mechanisms call `saveContext()` when they know the context is correct. This makes the data flow deterministic and auditable.

---

## What We Enhanced

| Component | Previous State | Enhancement in Feature 10 |
|-----------|---------------|--------------------------|
| `SimpleFilterChainProxy` | Clears `SecurityContextHolder` in finally | Filter still clears as a safety net; `SecurityContextHolderFilter` now owns the load/clear lifecycle |
| `SimpleAbstractAuthenticationProcessingFilter` | Sets context in `SecurityContextHolder` only | Now also saves to `SecurityContextRepository` if configured |
| `SimpleBasicAuthenticationFilter` | Sets context in `SecurityContextHolder` only | Now also saves to `SecurityContextRepository` if configured |
| `SimpleFormLoginConfigurer` | Wires AuthenticationManager and handlers | Now also wires `SecurityContextRepository` from shared objects |
| `SimpleHttpBasicConfigurer` | Wires AuthenticationManager and entry point | Now also wires `SecurityContextRepository` from shared objects |
| `SimpleHttpSecurity` | Filter order map had `"SecurityContextHolderFilter"` (placeholder) | Renamed to `"SimpleSecurityContextHolderFilter"`; added `sessionManagement()` DSL method |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | File |
|------------------|----------------------|------|
| `SecurityContextRepository` | `SecurityContextRepository` | `web/src/main/java/.../web/context/SecurityContextRepository.java` |
| `SimpleHttpSessionSecurityContextRepository` | `HttpSessionSecurityContextRepository` | `web/src/main/java/.../web/context/HttpSessionSecurityContextRepository.java` |
| `SimpleSecurityContextHolderFilter` | `SecurityContextHolderFilter` | `web/src/main/java/.../web/context/SecurityContextHolderFilter.java` |
| `SessionCreationPolicy` | `SessionCreationPolicy` | `config/src/main/java/.../config/http/SessionCreationPolicy.java` |
| `SimpleSessionManagementConfigurer` | `SessionManagementConfigurer` | `config/src/main/java/.../configurers/SessionManagementConfigurer.java` |
| `NullSecurityContextRepository` (inner class) | `NullSecurityContextRepository` | `web/src/main/java/.../web/context/NullSecurityContextRepository.java` |

### What the real framework adds

- **Deferred context loading**: `SecurityContextHolderFilter` uses `Supplier<SecurityContext>` for lazy loading — the context is only read from the session when `SecurityContextHolder.getContext()` is first called
- **`DelegatingSecurityContextRepository`**: Wraps multiple repositories (session + request attribute) so the context is available even in stateless mode within a single request
- **`RequestAttributeSecurityContextRepository`**: Stores context as a request attribute (not session) for per-request availability
- **`SessionManagementFilter`**: A separate filter for session fixation protection and concurrent session control (our simplified version omits this)
- **`ForceEagerSessionCreationFilter`**: For `ALWAYS` policy, ensures a session exists before any other processing
- **`@Transient` annotation support**: Prevents specific `Authentication` implementations from being persisted
- **`AuthenticationTrustResolver`**: Prevents anonymous authentications from being saved to the session

---

## Complete Code

### New Files

#### `[NEW] simple-security-web/src/main/java/simple/security/web/context/SecurityContextRepository.java`

```java
package simple.security.web.context;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.context.SecurityContext;

public interface SecurityContextRepository {

	SecurityContext loadContext(HttpServletRequest request);

	void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response);

	boolean containsContext(HttpServletRequest request);

}
```

#### `[NEW] simple-security-web/src/main/java/simple/security/web/context/SimpleHttpSessionSecurityContextRepository.java`

```java
package simple.security.web.context;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.context.SecurityContextImpl;

public class SimpleHttpSessionSecurityContextRepository implements SecurityContextRepository {

	public static final String SPRING_SECURITY_CONTEXT_KEY = "SPRING_SECURITY_CONTEXT";

	private boolean allowSessionCreation = true;

	@Override
	public SecurityContext loadContext(HttpServletRequest request) {
		HttpSession session = request.getSession(false);
		if (session == null) {
			return null;
		}
		Object contextObj = session.getAttribute(SPRING_SECURITY_CONTEXT_KEY);
		if (contextObj instanceof SecurityContext securityContext) {
			return securityContext;
		}
		return null;
	}

	@Override
	public void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response) {
		if (context == null) {
			throw new IllegalArgumentException("SecurityContext cannot be null");
		}
		if (context.getAuthentication() == null) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				session.removeAttribute(SPRING_SECURITY_CONTEXT_KEY);
			}
			return;
		}
		HttpSession session = request.getSession(this.allowSessionCreation);
		if (session != null) {
			session.setAttribute(SPRING_SECURITY_CONTEXT_KEY, context);
		}
	}

	@Override
	public boolean containsContext(HttpServletRequest request) {
		HttpSession session = request.getSession(false);
		if (session == null) {
			return false;
		}
		return session.getAttribute(SPRING_SECURITY_CONTEXT_KEY) != null;
	}

	public void setAllowSessionCreation(boolean allowSessionCreation) {
		this.allowSessionCreation = allowSessionCreation;
	}

	public boolean isAllowSessionCreation() {
		return this.allowSessionCreation;
	}

}
```

#### `[NEW] simple-security-web/src/main/java/simple/security/web/context/SimpleSecurityContextHolderFilter.java`

```java
package simple.security.web.context;

import java.io.IOException;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;

public class SimpleSecurityContextHolderFilter implements Filter {

	private static final String FILTER_APPLIED =
			SimpleSecurityContextHolderFilter.class.getName() + ".APPLIED";

	private final SecurityContextRepository securityContextRepository;

	public SimpleSecurityContextHolderFilter(SecurityContextRepository securityContextRepository) {
		if (securityContextRepository == null) {
			throw new IllegalArgumentException("securityContextRepository cannot be null");
		}
		this.securityContextRepository = securityContextRepository;
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);
	}

	private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		if (request.getAttribute(FILTER_APPLIED) != null) {
			chain.doFilter(request, response);
			return;
		}
		request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
		try {
			SecurityContext context = this.securityContextRepository.loadContext(request);
			if (context != null) {
				SecurityContextHolder.setContext(context);
			}
			chain.doFilter(request, response);
		}
		finally {
			SecurityContextHolder.clearContext();
			request.removeAttribute(FILTER_APPLIED);
		}
	}

	public SecurityContextRepository getSecurityContextRepository() {
		return this.securityContextRepository;
	}

}
```

#### `[NEW] simple-security-config/src/main/java/simple/security/config/http/SessionCreationPolicy.java`

```java
package simple.security.config.http;

public enum SessionCreationPolicy {

	ALWAYS,
	IF_REQUIRED,
	NEVER,
	STATELESS

}
```

#### `[NEW] simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleSessionManagementConfigurer.java`

```java
package simple.security.config.annotation.web.configurers;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.config.http.SessionCreationPolicy;
import simple.security.web.context.SecurityContextRepository;
import simple.security.web.context.SimpleHttpSessionSecurityContextRepository;
import simple.security.web.context.SimpleSecurityContextHolderFilter;

public class SimpleSessionManagementConfigurer
		extends SimpleAbstractHttpConfigurer<SimpleSessionManagementConfigurer> {

	private SessionCreationPolicy sessionCreationPolicy = SessionCreationPolicy.IF_REQUIRED;

	private SecurityContextRepository securityContextRepository;

	@Override
	public void init(SimpleHttpSecurity builder) throws Exception {
		SecurityContextRepository repository = resolveSecurityContextRepository();
		builder.setSharedObject(SecurityContextRepository.class, repository);
	}

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		SecurityContextRepository repository = builder.getSharedObject(SecurityContextRepository.class);
		SimpleSecurityContextHolderFilter filter = new SimpleSecurityContextHolderFilter(repository);
		builder.addFilter(filter);
	}

	public SimpleSessionManagementConfigurer sessionCreationPolicy(SessionCreationPolicy policy) {
		if (policy == null) {
			throw new IllegalArgumentException("sessionCreationPolicy cannot be null");
		}
		this.sessionCreationPolicy = policy;
		return this;
	}

	public SimpleSessionManagementConfigurer securityContextRepository(SecurityContextRepository repository) {
		if (repository == null) {
			throw new IllegalArgumentException("securityContextRepository cannot be null");
		}
		this.securityContextRepository = repository;
		return this;
	}

	private SecurityContextRepository resolveSecurityContextRepository() {
		if (this.securityContextRepository != null) {
			return this.securityContextRepository;
		}
		if (this.sessionCreationPolicy == SessionCreationPolicy.STATELESS) {
			return new NullSecurityContextRepository();
		}
		SimpleHttpSessionSecurityContextRepository repository =
				new SimpleHttpSessionSecurityContextRepository();
		if (this.sessionCreationPolicy == SessionCreationPolicy.NEVER) {
			repository.setAllowSessionCreation(false);
		}
		return repository;
	}

	public SessionCreationPolicy getSessionCreationPolicy() {
		return this.sessionCreationPolicy;
	}

	static class NullSecurityContextRepository implements SecurityContextRepository {

		@Override
		public simple.security.core.context.SecurityContext loadContext(
				jakarta.servlet.http.HttpServletRequest request) {
			return null;
		}

		@Override
		public void saveContext(simple.security.core.context.SecurityContext context,
				jakarta.servlet.http.HttpServletRequest request,
				jakarta.servlet.http.HttpServletResponse response) {
		}

		@Override
		public boolean containsContext(jakarta.servlet.http.HttpServletRequest request) {
			return false;
		}

	}

}
```

### Modified Files

#### `[MODIFIED] simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

Changes:
1. Renamed `"SecurityContextHolderFilter"` to `"SimpleSecurityContextHolderFilter"` in `FILTER_ORDER_MAP`
2. Added import for `SimpleSessionManagementConfigurer`
3. Added `sessionManagement(Customizer)` DSL method

#### `[MODIFIED] simple-security-web/src/main/java/simple/security/web/authentication/SimpleAbstractAuthenticationProcessingFilter.java`

Changes:
1. Added `SecurityContextRepository` field
2. `successfulAuthentication()` now saves context to repository if configured
3. Added `setSecurityContextRepository()` setter

#### `[MODIFIED] simple-security-web/src/main/java/simple/security/web/authentication/SimpleBasicAuthenticationFilter.java`

Changes:
1. Added `SecurityContextRepository` field
2. `doFilter()` success path now saves context to repository if configured
3. Added `setSecurityContextRepository()` setter

#### `[MODIFIED] simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleFormLoginConfigurer.java`

Changes:
1. `configure()` now reads `SecurityContextRepository` from shared objects and wires it into the auth filter

#### `[MODIFIED] simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleHttpBasicConfigurer.java`

Changes:
1. `configure()` now reads `SecurityContextRepository` from shared objects and wires it into the auth filter

### Test Files

#### `[NEW] simple-security-config/src/test/java/simple/security/config/SessionManagementApiTest.java`
10 client-perspective tests covering: session persistence, stateless mode, NEVER policy, filter ordering, context cleanup, custom repository.

#### `[NEW] simple-security-web/src/test/java/simple/security/web/context/SimpleHttpSessionSecurityContextRepositoryTest.java`
12 internal tests covering: load/save/contains operations, session creation control, null handling.

#### `[NEW] simple-security-web/src/test/java/simple/security/web/context/SimpleSecurityContextHolderFilterTest.java`
6 internal tests covering: context population, cleanup, exception safety, double-application guard.
