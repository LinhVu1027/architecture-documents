# Chapter 6: Form Login & HTTP Basic Authentication

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure form login with defaults:
SimpleHttpSecurity http = new SimpleHttpSecurity();
http.authenticationManager(myAuthManager);
http.formLogin(Customizer.withDefaults());
SecurityFilterChain chain = http.build();

// POST /login with username=user&password=password
// → authenticates via AuthenticationManager
// → on success: sets SecurityContext, redirects to "/"
// → on failure: clears SecurityContext, redirects to "/login?error"

// Configure HTTP Basic with defaults:
SimpleHttpSecurity http2 = new SimpleHttpSecurity();
http2.authenticationManager(myAuthManager);
http2.httpBasic(Customizer.withDefaults());
SecurityFilterChain chain2 = http2.build();

// GET /api/data with Authorization: Basic dXNlcjpwYXNzd29yZA==
// → decodes Base64, authenticates via AuthenticationManager
// → on success: sets SecurityContext, CONTINUES the filter chain
// → on failure: clears SecurityContext, sends 401 + WWW-Authenticate

// Both together — form login filter at position 300, Basic at 400:
SimpleHttpSecurity http3 = new SimpleHttpSecurity();
http3.authenticationManager(myAuthManager);
http3.formLogin(Customizer.withDefaults());
http3.httpBasic(Customizer.withDefaults());
SecurityFilterChain chain3 = http3.build();
assertThat(chain3.getFilters()).hasSize(2);
assertThat(chain3.getFilters().get(0)).isInstanceOf(SimpleUsernamePasswordAuthenticationFilter.class);
assertThat(chain3.getFilters().get(1)).isInstanceOf(SimpleBasicAuthenticationFilter.class);
```

Questions to guide your thinking:
- Why does `SimpleBasicAuthenticationFilter` implement `Filter` directly instead of extending `SimpleAbstractAuthenticationProcessingFilter`?
- Why does the form login filter **not** continue the filter chain after successful authentication, while the Basic filter always does?
- How does the two-phase configurer lifecycle (init then configure) enable the form login configurer to register an `AuthenticationEntryPoint` that the HTTP Basic configurer can see?
- Why does `SimpleBasicAuthenticationFilter` have two constructors — one with an entry point and one without? What does the `ignoreFailure` flag control?

---

## The API Contract

Feature 6 introduces two concrete authentication mechanisms that plug into the DSL
from Feature 5. Each mechanism is exposed as a convenience method on `SimpleHttpSecurity`
and backed by a configurer that produces a filter.

### Form Login DSL

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity formLogin(Customizer<SimpleFormLoginConfigurer> formLoginCustomizer) {
    formLoginCustomizer.customize(getOrApply(new SimpleFormLoginConfigurer()));
    return this;
}
```

The configurer exposes these DSL methods:

```java
// simple/security/config/annotation/web/configurers/SimpleFormLoginConfigurer.java
public class SimpleFormLoginConfigurer extends SimpleAbstractHttpConfigurer<SimpleFormLoginConfigurer> {
    public SimpleFormLoginConfigurer loginPage(String loginPage) { ... }
    public SimpleFormLoginConfigurer loginProcessingUrl(String loginProcessingUrl) { ... }
    public SimpleFormLoginConfigurer usernameParameter(String usernameParameter) { ... }
    public SimpleFormLoginConfigurer passwordParameter(String passwordParameter) { ... }
    public SimpleFormLoginConfigurer defaultSuccessUrl(String defaultSuccessUrl) { ... }
    public SimpleFormLoginConfigurer failureUrl(String failureUrl) { ... }
    public SimpleFormLoginConfigurer successHandler(AuthenticationSuccessHandler handler) { ... }
    public SimpleFormLoginConfigurer failureHandler(AuthenticationFailureHandler handler) { ... }
    public SimpleFormLoginConfigurer authenticationEntryPoint(AuthenticationEntryPoint entryPoint) { ... }
}
```

### HTTP Basic DSL

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity httpBasic(Customizer<SimpleHttpBasicConfigurer> httpBasicCustomizer) {
    httpBasicCustomizer.customize(getOrApply(new SimpleHttpBasicConfigurer()));
    return this;
}
```

The configurer exposes:

```java
// simple/security/config/annotation/web/configurers/SimpleHttpBasicConfigurer.java
public class SimpleHttpBasicConfigurer extends SimpleAbstractHttpConfigurer<SimpleHttpBasicConfigurer> {
    public SimpleHttpBasicConfigurer realmName(String realmName) { ... }
    public SimpleHttpBasicConfigurer authenticationEntryPoint(AuthenticationEntryPoint entryPoint) { ... }
}
```

### Strategy interfaces

Both mechanisms rely on three new strategy interfaces:

```java
// How to start authentication (redirect to login page? send 401?)
@FunctionalInterface
public interface AuthenticationEntryPoint {
    void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException, ServletException;
}

// What to do after successful authentication (redirect? write response?)
@FunctionalInterface
public interface AuthenticationSuccessHandler {
    void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
            Authentication authentication) throws IOException, ServletException;
}

// What to do after failed authentication (redirect to error? send 401?)
@FunctionalInterface
public interface AuthenticationFailureHandler {
    void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException exception) throws IOException, ServletException;
}
```

These are `@FunctionalInterface` — clients can use lambdas:

```java
http.formLogin(form -> form
    .successHandler((req, res, auth) -> {
        res.setStatus(200);
        res.getWriter().write("Welcome " + auth.getName());
    })
    .failureHandler((req, res, ex) -> {
        res.setStatus(403);
        res.getWriter().write("Login failed: " + ex.getMessage());
    }));
```

---

## Client Usage & Tests

The tests in `FormLoginHttpBasicApiTest` define the behavioral contract from a client's
perspective. Each test builds a complete filter chain and sends mock HTTP requests through it.

### Form Login with defaults

```java
@Test
void shouldAuthenticateWithFormLoginDefaults() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(Customizer.withDefaults());
    });

    // POST /login with correct credentials
    MockHttpServletRequest request = postLogin("user", "password");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Default success: redirect to "/"
    assertThat(response.getRedirectedUrl()).isEqualTo("/");
}

@Test
void shouldRejectInvalidCredentialsWithFormLogin() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(Customizer.withDefaults());
    });

    MockHttpServletRequest request = postLogin("user", "wrong");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    // Default failure: redirect to /login?error
    assertThat(response.getRedirectedUrl()).isEqualTo("/login?error");
    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
}

@Test
void shouldPassThroughNonLoginRequests() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(Customizer.withDefaults());
    });

    // GET /other — should pass through the form login filter
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/other");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();
    proxy.doFilter(request, response, chain);

    assertThat(chain.getRequest()).isNotNull();
}
```

### Form Login with custom configuration

```java
@Test
void shouldCustomizeFormLoginParameters() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(form -> form
                .usernameParameter("email")
                .passwordParameter("pass"));
    });

    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
    request.setParameter("email", "user");
    request.setParameter("pass", "password");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getRedirectedUrl()).isEqualTo("/");
}

@Test
void shouldRedirectToCustomSuccessUrl() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(form -> form.defaultSuccessUrl("/dashboard"));
    });

    MockHttpServletRequest request = postLogin("user", "password");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getRedirectedUrl()).isEqualTo("/dashboard");
}

@Test
void shouldUseCustomSuccessHandler() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(form -> form.successHandler(
                (request, response, auth) -> {
                    response.setStatus(200);
                    response.getWriter().write("Welcome " + auth.getName());
                }));
    });

    MockHttpServletRequest request = postLogin("user", "password");
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(response.getContentAsString()).isEqualTo("Welcome user");
}
```

### HTTP Basic with defaults

```java
@Test
void shouldAuthenticateWithHttpBasicHeader() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.httpBasic(Customizer.withDefaults());
    });

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", basicAuth("user", "password"));
    MockHttpServletResponse response = new MockHttpServletResponse();
    AtomicReference<Authentication> captured = new AtomicReference<>();
    FilterChain capturingChain = (req, res) ->
            captured.set(SecurityContextHolder.getContext().getAuthentication());
    proxy.doFilter(request, response, capturingChain);

    // Chain continues (Basic auth always continues)
    assertThat(captured.get()).isNotNull();
    assertThat(captured.get().getName()).isEqualTo("user");
    assertThat(captured.get().isAuthenticated()).isTrue();
}

@Test
void shouldRejectInvalidBasicCredentials() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.httpBasic(Customizer.withDefaults());
    });

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", basicAuth("user", "wrong"));
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(401);
    assertThat(response.getHeader("WWW-Authenticate")).contains("Basic realm=\"Realm\"");
}

@Test
void shouldUseCustomRealmName() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.httpBasic(basic -> basic.realmName("My API"));
    });

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
    request.addHeader("Authorization", basicAuth("user", "wrong"));
    MockHttpServletResponse response = new MockHttpServletResponse();
    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getHeader("WWW-Authenticate")).isEqualTo("Basic realm=\"My API\"");
}
```

### Both form login and HTTP Basic together

```java
@Test
void shouldConfigureBothFormLoginAndHttpBasic() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy(http -> {
        http.authenticationManager(testAuthenticationManager());
        http.formLogin(Customizer.withDefaults());
        http.httpBasic(Customizer.withDefaults());
    });

    // Form login works
    MockHttpServletRequest formRequest = postLogin("user", "password");
    MockHttpServletResponse formResponse = new MockHttpServletResponse();
    proxy.doFilter(formRequest, formResponse, new MockFilterChain());
    assertThat(formResponse.getRedirectedUrl()).isEqualTo("/");

    SecurityContextHolder.clearContext();

    // HTTP Basic also works
    MockHttpServletRequest basicRequest = new MockHttpServletRequest("GET", "/api");
    basicRequest.addHeader("Authorization", basicAuth("user", "password"));
    MockHttpServletResponse basicResponse = new MockHttpServletResponse();
    AtomicReference<Authentication> captured = new AtomicReference<>();
    FilterChain capturingChain = (req, res) ->
            captured.set(SecurityContextHolder.getContext().getAuthentication());
    proxy.doFilter(basicRequest, basicResponse, capturingChain);
    assertThat(captured.get()).isNotNull();
    assertThat(captured.get().getName()).isEqualTo("user");
}
```

### Filter chain ordering

```java
@Test
void shouldOrderFormLoginBeforeBasic() throws Exception {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.authenticationManager(testAuthenticationManager());
    http.httpBasic(Customizer.withDefaults());    // registered second
    http.formLogin(Customizer.withDefaults());     // registered first? No — order doesn't matter

    SecurityFilterChain chain = http.build();
    List<Filter> filters = chain.getFilters();

    assertThat(filters).hasSize(2);
    // Form login (300) comes before Basic (400) regardless of registration order
    assertThat(filters.get(0)).isInstanceOf(SimpleUsernamePasswordAuthenticationFilter.class);
    assertThat(filters.get(1)).isInstanceOf(SimpleBasicAuthenticationFilter.class);
}
```

---

## Implementing the Call Chain

### Call chain recap

```
Client calls http.formLogin(Customizer.withDefaults())
  │
  ├─► SimpleHttpSecurity.formLogin()
  │     getOrApply(new SimpleFormLoginConfigurer()) — register configurer idempotently
  │     customizer.customize(configurer) — apply user's lambda (or no-op for defaults)
  │
  ├─► Client calls http.httpBasic(Customizer.withDefaults())
  │     getOrApply(new SimpleHttpBasicConfigurer()) — same pattern
  │
  ├─► Client calls http.build()
  │     │
  │     ├─► Phase 1: INITIALIZING
  │     │     SimpleFormLoginConfigurer.init(http)
  │     │       Resolves default URLs (loginProcessingUrl, failureUrl)
  │     │       Creates LoginUrlAuthenticationEntryPoint
  │     │       Registers entry point as shared object
  │     │     SimpleHttpBasicConfigurer.init(http)
  │     │       Checks: entry point already registered by form login → skip
  │     │
  │     ├─► Phase 2: beforeConfigure()
  │     │     Resolves AuthenticationManager → setSharedObject
  │     │
  │     ├─► Phase 3: CONFIGURING
  │     │     SimpleFormLoginConfigurer.configure(http)
  │     │       Gets AuthenticationManager from shared objects
  │     │       Wires filter: request matcher, auth manager, success/failure handlers
  │     │       Calls http.addFilter(usernamePasswordFilter) → position 300
  │     │     SimpleHttpBasicConfigurer.configure(http)
  │     │       Gets AuthenticationManager from shared objects
  │     │       Creates BasicAuthenticationFilter with entry point
  │     │       Calls http.addFilter(basicFilter) → position 400
  │     │
  │     └─► Phase 4: performBuild()
  │           Sort filters: [300: UsernamePasswordFilter, 400: BasicFilter]
  │           Wrap in SimpleDefaultSecurityFilterChain
  │
  └─► At request time:
        POST /login → UsernamePasswordFilter intercepts → authenticate → redirect (no chain continuation)
        GET /api + Authorization: Basic → BasicFilter decodes → authenticate → continue chain
        GET /other (no creds) → both filters pass through → chain continues unauthenticated
```

### 6a. Strategy interfaces: entry points and handlers

The three strategy interfaces decouple the *decision* about what to do from the *mechanism*
that does it. Each authentication filter delegates to these strategies rather than
hardcoding behavior.

**`AuthenticationEntryPoint`** — how to start authentication:

```java
@FunctionalInterface
public interface AuthenticationEntryPoint {
    void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException, ServletException;
}
```

Two implementations:

- `SimpleLoginUrlAuthenticationEntryPoint` — sends a 302 redirect to the login page
- `SimpleBasicAuthenticationEntryPoint` — sends 401 with `WWW-Authenticate: Basic realm="..."`

**Why is this an interface and not hardcoded?** The same filter infrastructure can work with
completely different authentication schemes. Form login redirects to a page. HTTP Basic
sends a challenge header. A custom API might return JSON. The entry point abstraction makes
the filter framework-agnostic about *how* authentication starts.

**`AuthenticationSuccessHandler`** and **`AuthenticationFailureHandler`** follow the same
pattern — they decouple the response behavior from the authentication filter. Default
implementations redirect; custom implementations can do anything (write JSON, set cookies,
log audit events).

### 6b. The Template Method base: SimpleAbstractAuthenticationProcessingFilter

This is the heart of form-based authentication. It uses the **Template Method** pattern:
the `doFilter` method defines the algorithm skeleton, and the `attemptAuthentication`
method is the extension point that subclasses override.

```java
public abstract class SimpleAbstractAuthenticationProcessingFilter implements Filter {

    private RequestMatcher requiresAuthenticationRequestMatcher;
    private AuthenticationManager authenticationManager;
    private AuthenticationSuccessHandler successHandler;
    private AuthenticationFailureHandler failureHandler;
```

The `doFilter` flow:

```java
@Override
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        throws IOException, ServletException {
    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) res;

    if (!requiresAuthentication(request)) {
        chain.doFilter(request, response);  // Not our URL — pass through
        return;
    }

    try {
        Authentication authResult = attemptAuthentication(request, response);  // TEMPLATE METHOD
        if (authResult == null) {
            return;  // Incomplete (e.g., multi-step)
        }
        successfulAuthentication(request, response, authResult);
    }
    catch (AuthenticationException ex) {
        unsuccessfulAuthentication(request, response, ex);
    }
}
```

**Critical design decision: no `chain.doFilter()` after success.** The success handler
takes over the response (typically redirecting). This is fundamentally different from
HTTP Basic, which always continues the chain. Form login is a *one-time login ceremony* —
the POST /login request's entire purpose is to authenticate. There is no "rest of the
request" to process.

The success and failure handlers:

```java
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
        Authentication authResult) throws IOException, ServletException {
    SecurityContext context = SecurityContextHolder.createEmptyContext();
    context.setAuthentication(authResult);
    SecurityContextHolder.setContext(context);
    this.successHandler.onAuthenticationSuccess(request, response, authResult);
}

protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
        AuthenticationException failed) throws IOException, ServletException {
    SecurityContextHolder.clearContext();
    this.failureHandler.onAuthenticationFailure(request, response, failed);
}
```

### 6c. The form login filter: SimpleUsernamePasswordAuthenticationFilter

The concrete subclass provides the `attemptAuthentication` template method:

```java
public class SimpleUsernamePasswordAuthenticationFilter
        extends SimpleAbstractAuthenticationProcessingFilter {

    public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
    public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

    private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
    private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
    private boolean postOnly = true;
```

The default request matcher only activates on `POST /login`:

```java
private static final RequestMatcher DEFAULT_REQUEST_MATCHER = request ->
        "POST".equalsIgnoreCase(request.getMethod())
                && "/login".equals(getRequestPath(request));
```

The template method implementation:

```java
@Override
public Authentication attemptAuthentication(HttpServletRequest request,
        HttpServletResponse response) throws AuthenticationException {
    if (this.postOnly && !"POST".equalsIgnoreCase(request.getMethod())) {
        throw new AuthenticationServiceException(
                "Authentication method not supported: " + request.getMethod());
    }

    String username = obtainUsername(request);
    String password = obtainPassword(request);

    UsernamePasswordAuthenticationToken authRequest =
            UsernamePasswordAuthenticationToken.unauthenticated(username, password);

    return getAuthenticationManager().authenticate(authRequest);
}
```

**Why `AuthenticationServiceException` instead of `BadCredentialsException`?** Receiving
a GET on a POST-only endpoint is a *system* error (the client used the wrong HTTP method),
not a *credential* error. The distinction matters for error reporting.

### 6d. The HTTP Basic filter: SimpleBasicAuthenticationFilter

This filter has a fundamentally different design from the form login filter. It implements
`Filter` directly — it does NOT extend `SimpleAbstractAuthenticationProcessingFilter`.

**Why?** Three reasons:

1. **Every request vs. one URL**: Basic auth inspects *every* request for an `Authorization`
   header. Form login only activates on `POST /login`. The request-matching model is
   completely different.

2. **Continue chain vs. redirect**: On success, Basic auth *always* continues the filter
   chain — the request proceeds to the actual resource. Form login *never* continues —
   the success handler redirects. They have opposite post-authentication behavior.

3. **Stateless vs. stateful**: Basic auth is per-request (credentials on every request).
   Form login is a one-time ceremony (authenticate once, then use sessions). The lifecycle
   models are incompatible.

```java
public class SimpleBasicAuthenticationFilter implements Filter {

    private final AuthenticationManager authenticationManager;
    private final AuthenticationEntryPoint authenticationEntryPoint;
    private final boolean ignoreFailure;
```

**The two-constructor pattern and `ignoreFailure`**:

```java
// Constructor 1: no entry point, ignoreFailure = true
// Used when Basic auth is optional — failure just means unauthenticated
public SimpleBasicAuthenticationFilter(AuthenticationManager authenticationManager) {
    this.authenticationManager = authenticationManager;
    this.authenticationEntryPoint = null;
    this.ignoreFailure = true;
}

// Constructor 2: with entry point, ignoreFailure = false
// Used when Basic auth is required — failure triggers a 401 challenge
public SimpleBasicAuthenticationFilter(AuthenticationManager authenticationManager,
        AuthenticationEntryPoint authenticationEntryPoint) {
    this.authenticationManager = authenticationManager;
    this.authenticationEntryPoint = authenticationEntryPoint;
    this.ignoreFailure = false;
}
```

The `doFilter` flow:

```java
@Override
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        throws IOException, ServletException {
    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) res;

    try {
        UsernamePasswordAuthenticationToken authRequest = convertRequest(request);
        if (authRequest == null) {
            chain.doFilter(request, response);  // No Basic header → pass through
            return;
        }

        String username = authRequest.getName();
        if (!authenticationIsRequired(username)) {
            chain.doFilter(request, response);  // Same user already authenticated → skip
            return;
        }

        Authentication authResult = this.authenticationManager.authenticate(authRequest);

        // Success: set SecurityContext and CONTINUE the chain
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authResult);
        SecurityContextHolder.setContext(context);

        chain.doFilter(request, response);  // ALWAYS continues
    }
    catch (AuthenticationException ex) {
        SecurityContextHolder.clearContext();
        if (this.ignoreFailure) {
            chain.doFilter(request, response);  // Continue unauthenticated
        }
        else {
            this.authenticationEntryPoint.commence(request, response, ex);  // 401 challenge
        }
    }
}
```

**The `authenticationIsRequired` optimization**: If the `SecurityContext` already contains
an authenticated user with the same username, the filter skips re-authentication. This
avoids hitting the `AuthenticationManager` on every request when the same credentials
are sent repeatedly.

```java
private boolean authenticationIsRequired(String username) {
    Authentication existingAuth = SecurityContextHolder.getContext().getAuthentication();
    if (existingAuth == null || !existingAuth.isAuthenticated()) {
        return true;
    }
    if (!existingAuth.getName().equals(username)) {
        return true;  // Different user — must re-authenticate
    }
    return false;
}
```

The Base64 credential extraction:

```java
private UsernamePasswordAuthenticationToken convertRequest(HttpServletRequest request) {
    String header = request.getHeader("Authorization");
    if (header == null || !header.regionMatches(true, 0, "Basic ", 0, 6)) {
        return null;
    }

    byte[] decoded = Base64.getDecoder().decode(header.substring(6).trim());
    String credentials = new String(decoded, StandardCharsets.UTF_8);
    int colonIndex = credentials.indexOf(':');
    if (colonIndex == -1) {
        throw new BadCredentialsException("Invalid Basic authentication token");
    }

    String username = credentials.substring(0, colonIndex);
    String password = credentials.substring(colonIndex + 1);
    return UsernamePasswordAuthenticationToken.unauthenticated(username, password);
}
```

**Why `indexOf(':')` and not `split(":")`?** Passwords can contain colons. `split(":", 2)`
would also work, but `indexOf` plus `substring` makes the intent explicit: split on the
*first* colon only. The test `shouldHandlePasswordWithColon` verifies this with the
password `"pass:word:123"`.

### 6e. Entry point implementations

**`SimpleLoginUrlAuthenticationEntryPoint`** — for form login:

```java
public class SimpleLoginUrlAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private final String loginFormUrl;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException, ServletException {
        response.sendRedirect(request.getContextPath() + this.loginFormUrl);
    }
}
```

**`SimpleBasicAuthenticationEntryPoint`** — for HTTP Basic:

```java
public class SimpleBasicAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private final String realmName;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException, ServletException {
        response.setHeader("WWW-Authenticate", "Basic realm=\"" + this.realmName + "\"");
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, authException.getMessage());
    }
}
```

### 6f. The configurers: SimpleFormLoginConfigurer and SimpleHttpBasicConfigurer

**`SimpleFormLoginConfigurer`** uses both lifecycle phases:

**`init()`** — resolves defaults and registers the entry point:

```java
@Override
public void init(SimpleHttpSecurity builder) throws Exception {
    if (this.loginProcessingUrl == null) {
        this.loginProcessingUrl = this.loginPage;
    }
    if (this.failureUrl == null) {
        this.failureUrl = this.loginPage + "?error";
    }
    if (this.authenticationEntryPoint == null) {
        this.authenticationEntryPoint = new SimpleLoginUrlAuthenticationEntryPoint(this.loginPage);
    }

    // Register entry point so ExceptionTranslationFilter (future feature) can use it
    builder.setSharedObject(AuthenticationEntryPoint.class, this.authenticationEntryPoint);
}
```

**`configure()`** — creates and wires the filter:

```java
@Override
public void configure(SimpleHttpSecurity builder) throws Exception {
    AuthenticationManager authManager = builder.getSharedObject(AuthenticationManager.class);

    String processingUrl = this.loginProcessingUrl;
    this.authFilter.setRequiresAuthenticationRequestMatcher(
            request -> "POST".equalsIgnoreCase(request.getMethod())
                    && processingUrl.equals(getRequestPath(request)));

    this.authFilter.setAuthenticationManager(authManager);

    if (this.successHandler != null) {
        this.authFilter.setAuthenticationSuccessHandler(this.successHandler);
    } else {
        this.authFilter.setAuthenticationSuccessHandler(
                new SimpleRedirectAuthenticationSuccessHandler(this.defaultSuccessUrl));
    }

    if (this.failureHandler != null) {
        this.authFilter.setAuthenticationFailureHandler(this.failureHandler);
    } else {
        this.authFilter.setAuthenticationFailureHandler(
                new SimpleRedirectAuthenticationFailureHandler(this.failureUrl));
    }

    builder.addFilter(this.authFilter);
}
```

**`SimpleHttpBasicConfigurer`** demonstrates cross-configurer coordination:

```java
@Override
public void init(SimpleHttpSecurity builder) throws Exception {
    // If no form login entry point is registered, register the Basic one
    // (form login takes priority because it provides a user-friendly login page)
    if (builder.getSharedObject(AuthenticationEntryPoint.class) == null) {
        AuthenticationEntryPoint entryPoint = resolveEntryPoint();
        builder.setSharedObject(AuthenticationEntryPoint.class, entryPoint);
    }
}

@Override
public void configure(SimpleHttpSecurity builder) throws Exception {
    AuthenticationManager authManager = builder.getSharedObject(AuthenticationManager.class);
    AuthenticationEntryPoint entryPoint = resolveEntryPoint();
    SimpleBasicAuthenticationFilter filter =
            new SimpleBasicAuthenticationFilter(authManager, entryPoint);
    builder.addFilter(filter);
}
```

**Why does `init()` check before setting the entry point?** When both form login and HTTP
Basic are configured, form login's `init()` runs first (registration order) and sets the
entry point to a login page redirect. HTTP Basic's `init()` sees the entry point is
already set and skips — so the login page redirect wins. This is the two-phase lifecycle
enabling **cross-configurer coordination**: form login publishes during init, HTTP Basic
reads during init, and both create their filters during configure.

### 6g. DSL methods on SimpleHttpSecurity

The convenience methods follow the same pattern:

```java
public SimpleHttpSecurity formLogin(Customizer<SimpleFormLoginConfigurer> formLoginCustomizer) {
    formLoginCustomizer.customize(getOrApply(new SimpleFormLoginConfigurer()));
    return this;
}

public SimpleHttpSecurity httpBasic(Customizer<SimpleHttpBasicConfigurer> httpBasicCustomizer) {
    httpBasicCustomizer.customize(getOrApply(new SimpleHttpBasicConfigurer()));
    return this;
}
```

`getOrApply()` ensures idempotence — calling `http.formLogin(c1).formLogin(c2)` reuses
the same configurer instance. The first call creates and registers it; the second call
retrieves the existing one. Both customizers run against the same configurer, so settings
accumulate rather than conflict.

---

## Try It Yourself

**Challenge 1**: Implement a `SimpleDigestAuthenticationFilter` that reads a custom
`X-Digest-Auth` header, validates a SHA-256 hash of the password, and authenticates the
user. Should it extend `SimpleAbstractAuthenticationProcessingFilter` or implement
`Filter` directly? Think about whether it should continue the chain on success.

**Challenge 2**: Write a configurer `SimpleRememberMeConfigurer` that adds a filter
after `SimpleUsernamePasswordAuthenticationFilter` (position 301) which checks for a
`remember-me` cookie and auto-authenticates the user. Use `http.addFilterAfter()` in the
configurer's `configure()` method.

**Challenge 3**: Modify `SimpleBasicAuthenticationFilter` to support a configurable
`credentialsCharset` (the real framework uses `StandardCharsets.UTF_8` by default but
allows customization). Add a test that authenticates with a username containing non-ASCII
characters.

**Challenge 4**: What happens if a client configures `http.httpBasic(Customizer.withDefaults())`
without calling `http.authenticationManager(...)`? Trace the code path through the build
lifecycle and identify where the `NullPointerException` would occur. How would you add a
validation check to produce a clear error message?

---

## Why This Works

**The Template Method pattern in AbstractAuthenticationProcessingFilter**: The `doFilter`
method defines the invariant algorithm (match → attempt → success/failure), while
`attemptAuthentication` is the variable step that subclasses provide. This means the
security context management, success handling, and failure handling are written once in the
base class. A new authentication scheme (SAML, OAuth, custom header) only needs to provide
credential extraction logic.

**Two fundamentally different filter designs**: Form login and HTTP Basic look similar on
the surface (both authenticate users), but they have incompatible lifecycles. Form login
is a one-time ceremony on a specific URL that redirects after completion. HTTP Basic is a
per-request mechanism on every URL that continues the chain. The real framework recognized
this and gave them completely different base classes. Trying to force both into the same
abstraction would require so many conditionals and flags that the abstraction would provide
no value.

**Strategy pattern for response handling**: By extracting success, failure, and entry point
behaviors into interfaces, the filters become reusable across wildly different deployment
scenarios. A browser-based app redirects. A REST API returns JSON. A mobile backend returns
custom status codes. The filter framework handles all of these without modification.

**Cross-configurer coordination via shared objects**: The two-phase lifecycle (init then
configure) is not just for ordering — it enables configurers to *communicate*. Form login
registers an entry point during init. HTTP Basic checks for it during init. The
ExceptionTranslationFilter (future feature) will read it during configure. Without two
phases, configurers would have to be aware of each other's execution order, creating
tight coupling.

**The ignoreFailure toggle**: The one-arg constructor (no entry point, `ignoreFailure=true`)
vs. two-arg constructor (with entry point, `ignoreFailure=false`) encodes a real design
decision: is HTTP Basic the *primary* authentication mechanism (send a challenge on failure)
or a *supplementary* one (silently pass through if credentials are wrong, let a later filter
handle it)? The configurer always uses the two-arg constructor because it creates an entry
point, but the one-arg constructor exists for programmatic use.

---

## What We Enhanced

| What Changed | Before (Feature 5) | After (Feature 6) | Why |
|---|---|---|---|
| `SimpleHttpSecurity` | Generic `with()` only | Added `formLogin()` and `httpBasic()` DSL methods | Convenience methods that use `getOrApply()` for idempotent registration |
| Filter chain population | Only manual `addFilter`/`addFilterAt` | Configurers automatically add filters at well-known positions | The configurer lifecycle (init -> configure) now produces real security filters |
| Shared objects | Only `AuthenticationManager` | + `AuthenticationEntryPoint` | Entry point registered by form login configurer for future `ExceptionTranslationFilter` |
| Exception hierarchy | `AuthenticationException`, `BadCredentialsException` | + `AuthenticationServiceException` | Distinguishes system errors (wrong HTTP method) from credential errors |
| Web module | `SecurityFilterChain`, `FilterChainProxy`, request matchers | + Entry points, success/failure handlers, two authentication filters | The web module now contains real authentication processing filters |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|---|---|---|
| `AuthenticationEntryPoint` | `AuthenticationEntryPoint` | `web/src/main/java/org/springframework/security/web/AuthenticationEntryPoint.java` |
| `AuthenticationSuccessHandler` | `AuthenticationSuccessHandler` | `web/src/main/java/org/springframework/security/web/authentication/AuthenticationSuccessHandler.java` |
| `AuthenticationFailureHandler` | `AuthenticationFailureHandler` | `web/src/main/java/org/springframework/security/web/authentication/AuthenticationFailureHandler.java` |
| `SimpleAbstractAuthenticationProcessingFilter` | `AbstractAuthenticationProcessingFilter` | `web/src/main/java/org/springframework/security/web/authentication/AbstractAuthenticationProcessingFilter.java` |
| `SimpleUsernamePasswordAuthenticationFilter` | `UsernamePasswordAuthenticationFilter` | `web/src/main/java/org/springframework/security/web/authentication/UsernamePasswordAuthenticationFilter.java` |
| `SimpleBasicAuthenticationFilter` | `BasicAuthenticationFilter` | `web/src/main/java/org/springframework/security/web/authentication/www/BasicAuthenticationFilter.java` |
| `SimpleLoginUrlAuthenticationEntryPoint` | `LoginUrlAuthenticationEntryPoint` | `web/src/main/java/org/springframework/security/web/authentication/LoginUrlAuthenticationEntryPoint.java` |
| `SimpleBasicAuthenticationEntryPoint` | `BasicAuthenticationEntryPoint` | `web/src/main/java/org/springframework/security/web/authentication/www/BasicAuthenticationEntryPoint.java` |
| `SimpleRedirectAuthenticationSuccessHandler` | `SimpleUrlAuthenticationSuccessHandler` | `web/src/main/java/org/springframework/security/web/authentication/SimpleUrlAuthenticationSuccessHandler.java` |
| `SimpleRedirectAuthenticationFailureHandler` | `SimpleUrlAuthenticationFailureHandler` | `web/src/main/java/org/springframework/security/web/authentication/SimpleUrlAuthenticationFailureHandler.java` |
| `SimpleFormLoginConfigurer` | `FormLoginConfigurer` | `config/src/main/java/org/springframework/security/config/annotation/web/configurers/FormLoginConfigurer.java` |
| `SimpleHttpBasicConfigurer` | `HttpBasicConfigurer` | `config/src/main/java/org/springframework/security/config/annotation/web/configurers/HttpBasicConfigurer.java` |

### Key differences from the real framework

**`AbstractAuthenticationProcessingFilter`**: The real class has session management
(`SessionAuthenticationStrategy`), `RequestCache` for saved request replay,
`SecurityContextRepository` for persisting the context, event publishing, and
`SecurityContextHolderStrategy` support. Our simplified version focuses on the core
template method pattern.

**`BasicAuthenticationFilter`**: The real class extends `OncePerRequestFilter` (ensuring
it runs at most once per request in forward/include scenarios), uses
`SecurityContextHolderStrategy`, supports a configurable `AuthenticationConverter` for
credential extraction, and integrates with `SecurityContextRepository`. Our version
captures the essential design — implement `Filter` directly, always continue chain on
success, and the `ignoreFailure` toggle.

**`FormLoginConfigurer`**: The real class extends `AbstractAuthenticationFilterConfigurer`
(a shared base for form login and OpenID Connect) which extends
`AbstractHttpConfigurer`. It configures `DefaultLoginPageGeneratingFilter`,
`LogoutFilter` integration, `RequestCache`, `SessionAuthenticationStrategy`, and more.
Our version demonstrates the configurer lifecycle pattern with init/configure phases.

---

## Complete Code

### [NEW] `simple-security-web/src/main/java/simple/security/web/AuthenticationEntryPoint.java`

```java
package simple.security.web;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;

/**
 * Strategy interface used to commence an authentication scheme.
 *
 * <p>Called by security infrastructure when an unauthenticated request tries to access
 * a protected resource. Implementations decide HOW to start authentication — for example,
 * redirecting to a login page (form login) or sending a {@code WWW-Authenticate} challenge
 * (HTTP Basic).
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.AuthenticationEntryPoint}.
 *
 * Mirrors: org.springframework.security.web.AuthenticationEntryPoint
 * Source:  web/src/main/java/org/springframework/security/web/AuthenticationEntryPoint.java
 */
@FunctionalInterface
public interface AuthenticationEntryPoint {

	/**
	 * Commences an authentication scheme.
	 *
	 * @param request       the request that resulted in an AuthenticationException
	 * @param response      the response to modify (redirect, set status, add headers)
	 * @param authException the exception that triggered the entry point
	 */
	void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException;

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/AuthenticationSuccessHandler.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;

/**
 * Strategy interface for handling a successful authentication.
 *
 * <p>Implementations typically redirect to a target URL (e.g., the page the user
 * originally requested) or return a success response for API clients.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.AuthenticationSuccessHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.AuthenticationSuccessHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/AuthenticationSuccessHandler.java
 */
@FunctionalInterface
public interface AuthenticationSuccessHandler {

	/**
	 * Called when a user has been successfully authenticated.
	 *
	 * @param request        the request which caused the successful authentication
	 * @param response       the response
	 * @param authentication the Authentication object created during authentication
	 */
	void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) throws IOException, ServletException;

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/AuthenticationFailureHandler.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;

/**
 * Strategy interface for handling a failed authentication attempt.
 *
 * <p>Implementations typically redirect to an error page or return an error response
 * for API clients.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.AuthenticationFailureHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.AuthenticationFailureHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/AuthenticationFailureHandler.java
 */
@FunctionalInterface
public interface AuthenticationFailureHandler {

	/**
	 * Called when an authentication attempt fails.
	 *
	 * @param request   the request during which the authentication attempt occurred
	 * @param response  the response
	 * @param exception the exception which was thrown to reject the authentication request
	 */
	void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException exception) throws IOException, ServletException;

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleAbstractAuthenticationProcessingFilter.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.authentication.AuthenticationManager;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.util.matcher.RequestMatcher;

/**
 * Abstract base for authentication filters that intercept a specific URL.
 *
 * <p>This implements the <b>Template Method</b> pattern: the {@link #doFilter} method
 * handles the overall flow (match → attempt → success/failure), while subclasses
 * provide the {@link #attemptAuthentication} step that knows how to extract credentials
 * from the specific authentication scheme (form POST, SAML assertion, etc.).
 *
 * <h2>doFilter flow</h2>
 * <pre>
 * 1. Does this request match? (via RequestMatcher)
 *    └── NO  → chain.doFilter() (pass through)
 *    └── YES → attemptAuthentication()
 *        ├── SUCCESS → set SecurityContext, call successHandler
 *        └── FAILURE → clear SecurityContext, call failureHandler
 * </pre>
 *
 * <p>Key design difference from {@link SimpleBasicAuthenticationFilter}: this filter
 * does NOT continue the filter chain after successful authentication. The success
 * handler takes over the response (typically by redirecting).
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter}.
 *
 * Mirrors: org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter
 * Source:  web/src/main/java/org/springframework/security/web/authentication/AbstractAuthenticationProcessingFilter.java
 */
public abstract class SimpleAbstractAuthenticationProcessingFilter implements Filter {

	private RequestMatcher requiresAuthenticationRequestMatcher;

	private AuthenticationManager authenticationManager;

	private AuthenticationSuccessHandler successHandler =
			new SimpleRedirectAuthenticationSuccessHandler("/");

	private AuthenticationFailureHandler failureHandler =
			new SimpleRedirectAuthenticationFailureHandler(null);

	// --- Constructors ---

	protected SimpleAbstractAuthenticationProcessingFilter(RequestMatcher requiresAuthenticationRequestMatcher) {
		this.requiresAuthenticationRequestMatcher = requiresAuthenticationRequestMatcher;
	}

	protected SimpleAbstractAuthenticationProcessingFilter(
			RequestMatcher requiresAuthenticationRequestMatcher, AuthenticationManager authenticationManager) {
		this.requiresAuthenticationRequestMatcher = requiresAuthenticationRequestMatcher;
		this.authenticationManager = authenticationManager;
	}

	// --- Filter implementation ---

	@Override
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		if (!requiresAuthentication(request)) {
			chain.doFilter(request, response);
			return;
		}

		try {
			Authentication authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// Incomplete authentication (e.g., multi-step). Return without action.
				return;
			}
			successfulAuthentication(request, response, authResult);
		}
		catch (AuthenticationException ex) {
			unsuccessfulAuthentication(request, response, ex);
		}
	}

	// --- Template methods ---

	/**
	 * Performs the actual authentication. Subclasses extract credentials from the
	 * request and delegate to the {@link AuthenticationManager}.
	 *
	 * @return the authenticated user token, or {@code null} for incomplete authentication
	 * @throws AuthenticationException if authentication fails
	 */
	public abstract Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException, IOException, ServletException;

	// --- Success / failure handling ---

	/**
	 * Sets the {@link SecurityContext} with the authenticated user and delegates
	 * to the success handler.
	 */
	protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
			Authentication authResult) throws IOException, ServletException {
		SecurityContext context = SecurityContextHolder.createEmptyContext();
		context.setAuthentication(authResult);
		SecurityContextHolder.setContext(context);
		this.successHandler.onAuthenticationSuccess(request, response, authResult);
	}

	/**
	 * Clears the {@link SecurityContext} and delegates to the failure handler.
	 */
	protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException failed) throws IOException, ServletException {
		SecurityContextHolder.clearContext();
		this.failureHandler.onAuthenticationFailure(request, response, failed);
	}

	// --- Request matching ---

	protected boolean requiresAuthentication(HttpServletRequest request) {
		return this.requiresAuthenticationRequestMatcher.matches(request);
	}

	// --- Accessors ---

	public AuthenticationManager getAuthenticationManager() {
		return this.authenticationManager;
	}

	public void setAuthenticationManager(AuthenticationManager authenticationManager) {
		this.authenticationManager = authenticationManager;
	}

	public void setAuthenticationSuccessHandler(AuthenticationSuccessHandler successHandler) {
		this.successHandler = successHandler;
	}

	public void setAuthenticationFailureHandler(AuthenticationFailureHandler failureHandler) {
		this.failureHandler = failureHandler;
	}

	public void setRequiresAuthenticationRequestMatcher(RequestMatcher requestMatcher) {
		this.requiresAuthenticationRequestMatcher = requestMatcher;
	}

	public AuthenticationSuccessHandler getSuccessHandler() {
		return this.successHandler;
	}

	public AuthenticationFailureHandler getFailureHandler() {
		return this.failureHandler;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleUsernamePasswordAuthenticationFilter.java`

```java
package simple.security.web.authentication;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.authentication.AuthenticationManager;
import simple.security.authentication.AuthenticationServiceException;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.web.util.matcher.RequestMatcher;

/**
 * Processes a form login submission (typically {@code POST /login}) by extracting
 * username and password from request parameters and delegating to the
 * {@link AuthenticationManager}.
 *
 * <h2>How it works</h2>
 * <ol>
 *   <li>The filter only activates on requests matching the configured URL (default: {@code POST /login})</li>
 *   <li>Extracts {@code username} and {@code password} from request parameters</li>
 *   <li>Creates an unauthenticated {@link UsernamePasswordAuthenticationToken}</li>
 *   <li>Delegates to {@link AuthenticationManager#authenticate}</li>
 *   <li>Success → set SecurityContext + redirect (via success handler)</li>
 *   <li>Failure → clear SecurityContext + redirect to error page (via failure handler)</li>
 * </ol>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter}.
 *
 * Mirrors: org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
 * Source:  web/src/main/java/org/springframework/security/web/authentication/UsernamePasswordAuthenticationFilter.java
 */
public class SimpleUsernamePasswordAuthenticationFilter extends SimpleAbstractAuthenticationProcessingFilter {

	public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";

	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

	private static final RequestMatcher DEFAULT_REQUEST_MATCHER = request ->
			"POST".equalsIgnoreCase(request.getMethod())
					&& "/login".equals(getRequestPath(request));

	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;

	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;

	private boolean postOnly = true;

	// --- Constructors ---

	public SimpleUsernamePasswordAuthenticationFilter() {
		super(DEFAULT_REQUEST_MATCHER);
	}

	public SimpleUsernamePasswordAuthenticationFilter(AuthenticationManager authenticationManager) {
		super(DEFAULT_REQUEST_MATCHER, authenticationManager);
	}

	// --- Core authentication logic ---

	@Override
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException {
		if (this.postOnly && !"POST".equalsIgnoreCase(request.getMethod())) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}

		String username = obtainUsername(request);
		String password = obtainPassword(request);

		UsernamePasswordAuthenticationToken authRequest =
				UsernamePasswordAuthenticationToken.unauthenticated(username, password);

		return getAuthenticationManager().authenticate(authRequest);
	}

	// --- Parameter extraction ---

	private String obtainUsername(HttpServletRequest request) {
		String username = request.getParameter(this.usernameParameter);
		return (username != null) ? username.trim() : "";
	}

	private String obtainPassword(HttpServletRequest request) {
		String password = request.getParameter(this.passwordParameter);
		return (password != null) ? password : "";
	}

	// --- Accessors ---

	public void setUsernameParameter(String usernameParameter) {
		this.usernameParameter = usernameParameter;
	}

	public void setPasswordParameter(String passwordParameter) {
		this.passwordParameter = passwordParameter;
	}

	public String getUsernameParameter() {
		return this.usernameParameter;
	}

	public String getPasswordParameter() {
		return this.passwordParameter;
	}

	// --- Utility ---

	static String getRequestPath(HttpServletRequest request) {
		String uri = request.getRequestURI();
		String contextPath = request.getContextPath();
		if (contextPath != null && !contextPath.isEmpty()) {
			uri = uri.substring(contextPath.length());
		}
		return uri;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleBasicAuthenticationFilter.java`

```java
package simple.security.web.authentication;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.authentication.AuthenticationManager;
import simple.security.authentication.BadCredentialsException;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.AuthenticationEntryPoint;

/**
 * Processes an HTTP Basic authentication header ({@code Authorization: Basic <base64>})
 * on every request, authenticating the user if credentials are present.
 *
 * <h2>How it works (fundamentally different from form login)</h2>
 * <ol>
 *   <li>Inspects EVERY request for an {@code Authorization: Basic} header</li>
 *   <li>No header → pass through silently (no-op)</li>
 *   <li>Header present → Base64-decode, split on {@code :}, create token</li>
 *   <li>Skip if same user is already authenticated</li>
 *   <li>Authenticate via {@link AuthenticationManager}</li>
 *   <li>Success → set SecurityContext, ALWAYS continue filter chain</li>
 *   <li>Failure → clear SecurityContext:
 *       <ul>
 *         <li>With entry point: send 401 challenge (stop chain)</li>
 *         <li>Without entry point (ignoreFailure): continue chain unauthenticated</li>
 *       </ul>
 *   </li>
 * </ol>
 *
 * <p><b>Key design difference from form login:</b> This filter does NOT extend
 * {@link SimpleAbstractAuthenticationProcessingFilter}. It implements {@link Filter}
 * directly because it always continues the chain on success (Basic auth is per-request,
 * not a one-time login ceremony).
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.www.BasicAuthenticationFilter}.
 *
 * Mirrors: org.springframework.security.web.authentication.www.BasicAuthenticationFilter
 * Source:  web/src/main/java/org/springframework/security/web/authentication/www/BasicAuthenticationFilter.java
 */
public class SimpleBasicAuthenticationFilter implements Filter {

	private final AuthenticationManager authenticationManager;

	private final AuthenticationEntryPoint authenticationEntryPoint;

	private final boolean ignoreFailure;

	// --- Constructors ---

	/**
	 * Creates a filter that silently passes through on authentication failure
	 * (no entry point, no challenge). Useful when Basic auth is optional.
	 */
	public SimpleBasicAuthenticationFilter(AuthenticationManager authenticationManager) {
		this.authenticationManager = authenticationManager;
		this.authenticationEntryPoint = null;
		this.ignoreFailure = true;
	}

	/**
	 * Creates a filter that sends a {@code WWW-Authenticate} challenge on
	 * authentication failure via the provided entry point.
	 */
	public SimpleBasicAuthenticationFilter(AuthenticationManager authenticationManager,
			AuthenticationEntryPoint authenticationEntryPoint) {
		this.authenticationManager = authenticationManager;
		this.authenticationEntryPoint = authenticationEntryPoint;
		this.ignoreFailure = false;
	}

	// --- Filter implementation ---

	@Override
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		try {
			UsernamePasswordAuthenticationToken authRequest = convertRequest(request);
			if (authRequest == null) {
				// No Basic header present — pass through silently
				chain.doFilter(request, response);
				return;
			}

			String username = authRequest.getName();
			if (!authenticationIsRequired(username)) {
				// Same user already authenticated — skip re-authentication
				chain.doFilter(request, response);
				return;
			}

			Authentication authResult = this.authenticationManager.authenticate(authRequest);

			// Success: set SecurityContext and CONTINUE the chain
			SecurityContext context = SecurityContextHolder.createEmptyContext();
			context.setAuthentication(authResult);
			SecurityContextHolder.setContext(context);

			chain.doFilter(request, response);
		}
		catch (AuthenticationException ex) {
			SecurityContextHolder.clearContext();
			if (this.ignoreFailure) {
				chain.doFilter(request, response);
			}
			else {
				this.authenticationEntryPoint.commence(request, response, ex);
			}
		}
	}

	// --- Credential extraction ---

	/**
	 * Extracts Basic credentials from the {@code Authorization} header.
	 *
	 * @return an unauthenticated token, or {@code null} if no Basic header is present
	 */
	private UsernamePasswordAuthenticationToken convertRequest(HttpServletRequest request) {
		String header = request.getHeader("Authorization");
		if (header == null || !header.regionMatches(true, 0, "Basic ", 0, 6)) {
			return null;
		}

		byte[] decoded;
		try {
			decoded = Base64.getDecoder().decode(header.substring(6).trim());
		}
		catch (IllegalArgumentException ex) {
			throw new BadCredentialsException("Failed to decode Basic authentication token");
		}

		String credentials = new String(decoded, StandardCharsets.UTF_8);
		int colonIndex = credentials.indexOf(':');
		if (colonIndex == -1) {
			throw new BadCredentialsException("Invalid Basic authentication token");
		}

		String username = credentials.substring(0, colonIndex);
		String password = credentials.substring(colonIndex + 1);

		return UsernamePasswordAuthenticationToken.unauthenticated(username, password);
	}

	/**
	 * Checks whether the given username needs to be (re)authenticated.
	 * Returns {@code false} if the same user is already fully authenticated.
	 */
	private boolean authenticationIsRequired(String username) {
		Authentication existingAuth = SecurityContextHolder.getContext().getAuthentication();
		if (existingAuth == null || !existingAuth.isAuthenticated()) {
			return true;
		}
		if (!existingAuth.getName().equals(username)) {
			return true;
		}
		return false;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleLoginUrlAuthenticationEntryPoint.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;
import simple.security.web.AuthenticationEntryPoint;

/**
 * Entry point that redirects the user to a login page URL when authentication is required.
 *
 * <p>This is the default entry point for form-based login. When an unauthenticated
 * user tries to access a protected resource, this entry point sends a 302 redirect
 * to the configured login page.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint}.
 *
 * Mirrors: org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint
 * Source:  web/src/main/java/org/springframework/security/web/authentication/LoginUrlAuthenticationEntryPoint.java
 */
public class SimpleLoginUrlAuthenticationEntryPoint implements AuthenticationEntryPoint {

	private final String loginFormUrl;

	/**
	 * @param loginFormUrl the URL to redirect to for login (e.g., "/login")
	 */
	public SimpleLoginUrlAuthenticationEntryPoint(String loginFormUrl) {
		this.loginFormUrl = loginFormUrl;
	}

	@Override
	public void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException {
		response.sendRedirect(request.getContextPath() + this.loginFormUrl);
	}

	public String getLoginFormUrl() {
		return this.loginFormUrl;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleBasicAuthenticationEntryPoint.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;
import simple.security.web.AuthenticationEntryPoint;

/**
 * Entry point for HTTP Basic authentication that sends a {@code 401 Unauthorized}
 * response with a {@code WWW-Authenticate: Basic realm="..."} header.
 *
 * <p>This triggers the browser's built-in Basic authentication dialog. For API
 * clients, it signals that Basic credentials are expected.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.www.BasicAuthenticationEntryPoint}.
 *
 * Mirrors: org.springframework.security.web.authentication.www.BasicAuthenticationEntryPoint
 * Source:  web/src/main/java/org/springframework/security/web/authentication/www/BasicAuthenticationEntryPoint.java
 */
public class SimpleBasicAuthenticationEntryPoint implements AuthenticationEntryPoint {

	private final String realmName;

	public SimpleBasicAuthenticationEntryPoint(String realmName) {
		this.realmName = realmName;
	}

	@Override
	public void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException {
		response.setHeader("WWW-Authenticate", "Basic realm=\"" + this.realmName + "\"");
		response.sendError(HttpServletResponse.SC_UNAUTHORIZED, authException.getMessage());
	}

	public String getRealmName() {
		return this.realmName;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleRedirectAuthenticationSuccessHandler.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;

/**
 * Default success handler that redirects to a configured target URL after
 * successful authentication.
 *
 * <p>In the real framework, {@code SavedRequestAwareAuthenticationSuccessHandler}
 * would redirect back to the originally requested page. This simplified version
 * always redirects to the configured default URL.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/SimpleUrlAuthenticationSuccessHandler.java
 */
public class SimpleRedirectAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

	private final String defaultTargetUrl;

	/**
	 * @param defaultTargetUrl the URL to redirect to after successful authentication
	 */
	public SimpleRedirectAuthenticationSuccessHandler(String defaultTargetUrl) {
		this.defaultTargetUrl = defaultTargetUrl;
	}

	@Override
	public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) throws IOException, ServletException {
		response.sendRedirect(request.getContextPath() + this.defaultTargetUrl);
	}

	public String getDefaultTargetUrl() {
		return this.defaultTargetUrl;
	}

}
```

### [NEW] `simple-security-web/src/main/java/simple/security/web/authentication/SimpleRedirectAuthenticationFailureHandler.java`

```java
package simple.security.web.authentication;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;

/**
 * Default failure handler that redirects to a configured failure URL after
 * a failed authentication attempt.
 *
 * <p>If no failure URL is configured, sends a {@code 401 Unauthorized} response
 * instead of redirecting.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler}.
 *
 * Mirrors: org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler
 * Source:  web/src/main/java/org/springframework/security/web/authentication/SimpleUrlAuthenticationFailureHandler.java
 */
public class SimpleRedirectAuthenticationFailureHandler implements AuthenticationFailureHandler {

	private final String defaultFailureUrl;

	/**
	 * @param defaultFailureUrl the URL to redirect to on authentication failure
	 *                          (e.g., "/login?error"); may be {@code null} for 401 response
	 */
	public SimpleRedirectAuthenticationFailureHandler(String defaultFailureUrl) {
		this.defaultFailureUrl = defaultFailureUrl;
	}

	@Override
	public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException exception) throws IOException, ServletException {
		if (this.defaultFailureUrl == null) {
			response.sendError(HttpServletResponse.SC_UNAUTHORIZED, exception.getMessage());
		}
		else {
			response.sendRedirect(request.getContextPath() + this.defaultFailureUrl);
		}
	}

	public String getDefaultFailureUrl() {
		return this.defaultFailureUrl;
	}

}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/AuthenticationServiceException.java`

```java
package simple.security.authentication;

import simple.security.core.AuthenticationException;

/**
 * Thrown when an internal system error occurs during authentication — as opposed to
 * a user-supplied credential being wrong. For example, receiving a GET request on
 * a login endpoint that only accepts POST.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.authentication.AuthenticationServiceException}.
 */
public class AuthenticationServiceException extends AuthenticationException {

	public AuthenticationServiceException(String message) {
		super(message);
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleFormLoginConfigurer.java`

```java
package simple.security.config.annotation.web.configurers;

import simple.security.authentication.AuthenticationManager;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.AuthenticationEntryPoint;
import simple.security.web.authentication.AuthenticationFailureHandler;
import simple.security.web.authentication.AuthenticationSuccessHandler;
import simple.security.web.authentication.SimpleLoginUrlAuthenticationEntryPoint;
import simple.security.web.authentication.SimpleRedirectAuthenticationFailureHandler;
import simple.security.web.authentication.SimpleRedirectAuthenticationSuccessHandler;
import simple.security.web.authentication.SimpleUsernamePasswordAuthenticationFilter;

/**
 * Configures form-based login via the {@link SimpleHttpSecurity} DSL.
 *
 * <h2>Usage</h2>
 * <pre>
 * // Defaults (POST /login, username/password params, redirect to /)
 * http.formLogin(Customizer.withDefaults());
 *
 * // Custom configuration
 * http.formLogin(form -> form
 *     .loginPage("/custom-login")
 *     .usernameParameter("email")
 *     .defaultSuccessUrl("/dashboard")
 *     .failureUrl("/custom-login?error"));
 * </pre>
 *
 * <h2>What it does</h2>
 * <ol>
 *   <li>{@code init()} — resolves default URLs and registers the entry point</li>
 *   <li>{@code configure()} — creates and wires a {@link SimpleUsernamePasswordAuthenticationFilter},
 *       then adds it to the filter chain at position 300</li>
 * </ol>
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.web.configurers.FormLoginConfigurer}.
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.FormLoginConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/FormLoginConfigurer.java
 */
public class SimpleFormLoginConfigurer extends SimpleAbstractHttpConfigurer<SimpleFormLoginConfigurer> {

	private final SimpleUsernamePasswordAuthenticationFilter authFilter =
			new SimpleUsernamePasswordAuthenticationFilter();

	private String loginPage = "/login";

	private String loginProcessingUrl;

	private String defaultSuccessUrl = "/";

	private String failureUrl;

	private AuthenticationSuccessHandler successHandler;

	private AuthenticationFailureHandler failureHandler;

	private AuthenticationEntryPoint authenticationEntryPoint;

	// === Lifecycle ===

	/**
	 * Resolves default URLs and registers the authentication entry point as a shared
	 * object. Called during the INITIALIZING phase before any filter is created.
	 */
	@Override
	public void init(SimpleHttpSecurity builder) throws Exception {
		// Fill in defaults for anything not explicitly set
		if (this.loginProcessingUrl == null) {
			this.loginProcessingUrl = this.loginPage;
		}
		if (this.failureUrl == null) {
			this.failureUrl = this.loginPage + "?error";
		}
		if (this.authenticationEntryPoint == null) {
			this.authenticationEntryPoint = new SimpleLoginUrlAuthenticationEntryPoint(this.loginPage);
		}

		// Register entry point so ExceptionTranslationFilter (future feature) can use it
		builder.setSharedObject(AuthenticationEntryPoint.class, this.authenticationEntryPoint);
	}

	/**
	 * Creates and wires the {@link SimpleUsernamePasswordAuthenticationFilter},
	 * then adds it to the filter chain.
	 */
	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		AuthenticationManager authManager = builder.getSharedObject(AuthenticationManager.class);

		// Configure the request matcher for the login processing URL
		String processingUrl = this.loginProcessingUrl;
		this.authFilter.setRequiresAuthenticationRequestMatcher(
				request -> "POST".equalsIgnoreCase(request.getMethod())
						&& processingUrl.equals(getRequestPath(request)));

		// Wire AuthenticationManager
		this.authFilter.setAuthenticationManager(authManager);

		// Wire success handler
		if (this.successHandler != null) {
			this.authFilter.setAuthenticationSuccessHandler(this.successHandler);
		}
		else {
			this.authFilter.setAuthenticationSuccessHandler(
					new SimpleRedirectAuthenticationSuccessHandler(this.defaultSuccessUrl));
		}

		// Wire failure handler
		if (this.failureHandler != null) {
			this.authFilter.setAuthenticationFailureHandler(this.failureHandler);
		}
		else {
			this.authFilter.setAuthenticationFailureHandler(
					new SimpleRedirectAuthenticationFailureHandler(this.failureUrl));
		}

		builder.addFilter(this.authFilter);
	}

	// === DSL methods ===

	/**
	 * Specifies the URL to send users to for login. Also sets the entry point to
	 * redirect here when authentication is required.
	 *
	 * @param loginPage the login page URL (default: "/login")
	 */
	public SimpleFormLoginConfigurer loginPage(String loginPage) {
		this.loginPage = loginPage;
		return this;
	}

	/**
	 * Specifies the URL that processes the login form POST. Defaults to the login page URL.
	 *
	 * @param loginProcessingUrl the URL the filter watches (default: same as loginPage)
	 */
	public SimpleFormLoginConfigurer loginProcessingUrl(String loginProcessingUrl) {
		this.loginProcessingUrl = loginProcessingUrl;
		return this;
	}

	/**
	 * Specifies the request parameter name for the username.
	 *
	 * @param usernameParameter the parameter name (default: "username")
	 */
	public SimpleFormLoginConfigurer usernameParameter(String usernameParameter) {
		this.authFilter.setUsernameParameter(usernameParameter);
		return this;
	}

	/**
	 * Specifies the request parameter name for the password.
	 *
	 * @param passwordParameter the parameter name (default: "password")
	 */
	public SimpleFormLoginConfigurer passwordParameter(String passwordParameter) {
		this.authFilter.setPasswordParameter(passwordParameter);
		return this;
	}

	/**
	 * Specifies the URL to redirect to after successful authentication.
	 *
	 * @param defaultSuccessUrl the URL to redirect to on success (default: "/")
	 */
	public SimpleFormLoginConfigurer defaultSuccessUrl(String defaultSuccessUrl) {
		this.defaultSuccessUrl = defaultSuccessUrl;
		return this;
	}

	/**
	 * Specifies the URL to redirect to after authentication failure.
	 *
	 * @param failureUrl the URL to redirect to on failure (default: loginPage + "?error")
	 */
	public SimpleFormLoginConfigurer failureUrl(String failureUrl) {
		this.failureUrl = failureUrl;
		return this;
	}

	/**
	 * Specifies a custom success handler.
	 */
	public SimpleFormLoginConfigurer successHandler(AuthenticationSuccessHandler successHandler) {
		this.successHandler = successHandler;
		return this;
	}

	/**
	 * Specifies a custom failure handler.
	 */
	public SimpleFormLoginConfigurer failureHandler(AuthenticationFailureHandler failureHandler) {
		this.failureHandler = failureHandler;
		return this;
	}

	/**
	 * Specifies a custom authentication entry point.
	 */
	public SimpleFormLoginConfigurer authenticationEntryPoint(AuthenticationEntryPoint entryPoint) {
		this.authenticationEntryPoint = entryPoint;
		return this;
	}

	// === Accessors (for testing) ===

	public SimpleUsernamePasswordAuthenticationFilter getAuthFilter() {
		return this.authFilter;
	}

	// === Utility ===

	private static String getRequestPath(jakarta.servlet.http.HttpServletRequest request) {
		String uri = request.getRequestURI();
		String contextPath = request.getContextPath();
		if (contextPath != null && !contextPath.isEmpty()) {
			uri = uri.substring(contextPath.length());
		}
		return uri;
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleHttpBasicConfigurer.java`

```java
package simple.security.config.annotation.web.configurers;

import simple.security.authentication.AuthenticationManager;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.AuthenticationEntryPoint;
import simple.security.web.authentication.SimpleBasicAuthenticationEntryPoint;
import simple.security.web.authentication.SimpleBasicAuthenticationFilter;

/**
 * Configures HTTP Basic authentication via the {@link SimpleHttpSecurity} DSL.
 *
 * <h2>Usage</h2>
 * <pre>
 * // Defaults (realm "Realm", 401 + WWW-Authenticate on failure)
 * http.httpBasic(Customizer.withDefaults());
 *
 * // Custom realm
 * http.httpBasic(basic -> basic.realmName("My Application"));
 *
 * // Custom entry point
 * http.httpBasic(basic -> basic.authenticationEntryPoint(customEntryPoint));
 * </pre>
 *
 * <h2>What it does</h2>
 * <p>{@code configure()} creates a {@link SimpleBasicAuthenticationFilter} wired with
 * the {@link AuthenticationManager} and an {@link AuthenticationEntryPoint}, then adds
 * it to the filter chain at position 400.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.web.configurers.HttpBasicConfigurer}.
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.HttpBasicConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/HttpBasicConfigurer.java
 */
public class SimpleHttpBasicConfigurer extends SimpleAbstractHttpConfigurer<SimpleHttpBasicConfigurer> {

	private static final String DEFAULT_REALM = "Realm";

	private String realmName = DEFAULT_REALM;

	private AuthenticationEntryPoint authenticationEntryPoint;

	// === Lifecycle ===

	@Override
	public void init(SimpleHttpSecurity builder) throws Exception {
		// If no form login entry point is registered, register the Basic one
		// (form login takes priority because it provides a user-friendly login page)
		if (builder.getSharedObject(AuthenticationEntryPoint.class) == null) {
			AuthenticationEntryPoint entryPoint = resolveEntryPoint();
			builder.setSharedObject(AuthenticationEntryPoint.class, entryPoint);
		}
	}

	/**
	 * Creates the {@link SimpleBasicAuthenticationFilter} and adds it to the filter chain.
	 */
	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		AuthenticationManager authManager = builder.getSharedObject(AuthenticationManager.class);

		AuthenticationEntryPoint entryPoint = resolveEntryPoint();

		SimpleBasicAuthenticationFilter filter =
				new SimpleBasicAuthenticationFilter(authManager, entryPoint);

		builder.addFilter(filter);
	}

	// === DSL methods ===

	/**
	 * Sets the realm name for the {@code WWW-Authenticate} header.
	 *
	 * @param realmName the realm name (default: "Realm")
	 */
	public SimpleHttpBasicConfigurer realmName(String realmName) {
		this.realmName = realmName;
		return this;
	}

	/**
	 * Sets a custom entry point. When set, this replaces the default
	 * {@link SimpleBasicAuthenticationEntryPoint}.
	 */
	public SimpleHttpBasicConfigurer authenticationEntryPoint(AuthenticationEntryPoint entryPoint) {
		this.authenticationEntryPoint = entryPoint;
		return this;
	}

	// === Internal ===

	private AuthenticationEntryPoint resolveEntryPoint() {
		if (this.authenticationEntryPoint != null) {
			return this.authenticationEntryPoint;
		}
		return new SimpleBasicAuthenticationEntryPoint(this.realmName);
	}

}
```

### [MODIFIED] `simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

```java
package simple.security.config.annotation.web.builders;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import jakarta.servlet.Filter;

import simple.security.authentication.AuthenticationManager;
import simple.security.authentication.AuthenticationProvider;
import simple.security.authentication.SimpleProviderManager;
import simple.security.config.Customizer;
import simple.security.config.annotation.SecurityBuilder;
import simple.security.config.annotation.SecurityConfigurer;
import simple.security.config.annotation.web.configurers.SimpleAbstractHttpConfigurer;
import simple.security.config.annotation.web.configurers.SimpleFormLoginConfigurer;
import simple.security.config.annotation.web.configurers.SimpleHttpBasicConfigurer;
import simple.security.core.userdetails.UserDetailsService;
import simple.security.web.SecurityFilterChain;
import simple.security.web.SimpleDefaultSecurityFilterChain;
import simple.security.web.util.matcher.RequestMatcher;
import simple.security.web.util.matcher.SimpleAnyRequestMatcher;
import simple.security.web.util.matcher.SimplePathPatternRequestMatcher;

/**
 * The fluent DSL builder that produces a {@link SecurityFilterChain}.
 *
 * <p>Clients interact with this class by calling DSL methods (added by later features)
 * and calling {@link #build()} to produce the final filter chain. Each DSL method
 * registers a {@link SimpleAbstractHttpConfigurer} that adds its filters during the
 * build lifecycle.
 *
 * <h2>Build lifecycle</h2>
 * <pre>
 * 1. INITIALIZING  — configurer.init(this)     for ALL configurers
 * 2. beforeConfigure — resolve AuthenticationManager as a shared object
 * 3. CONFIGURING   — configurer.configure(this) for ALL configurers (they call addFilter)
 * 4. BUILDING      — sort filters by order, wrap in DefaultSecurityFilterChain
 * </pre>
 *
 * <h2>Filter ordering</h2>
 * Well-known security filter positions are pre-registered. Configurers add filters
 * via {@link #addFilter(Filter)} (for registered types) or
 * {@link #addFilterBefore}/{@link #addFilterAfter} (for custom positioning).
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.web.builders.HttpSecurity}.
 *
 * Mirrors: org.springframework.security.config.annotation.web.builders.HttpSecurity
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/builders/HttpSecurity.java
 */
public final class SimpleHttpSecurity implements SecurityBuilder<SecurityFilterChain> {

	// --- Well-known filter ordering (mirrors FilterOrderRegistration) ---

	private static final Map<String, Integer> FILTER_ORDER_MAP = new LinkedHashMap<>();

	static {
		int order = 100;
		FILTER_ORDER_MAP.put("SecurityContextHolderFilter", order);
		order += 100;
		FILTER_ORDER_MAP.put("CsrfFilter", order);
		order += 100;
		FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", order);
		order += 100;
		FILTER_ORDER_MAP.put("SimpleBasicAuthenticationFilter", order);
		order += 100;
		FILTER_ORDER_MAP.put("ExceptionTranslationFilter", order);
		order += 100;
		FILTER_ORDER_MAP.put("AuthorizationFilter", order);
	}

	// --- State ---

	private final LinkedHashMap<Class<? extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>>,
			SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>> configurers = new LinkedHashMap<>();

	private final List<OrderedFilter> filters = new ArrayList<>();

	private RequestMatcher requestMatcher = SimpleAnyRequestMatcher.INSTANCE;

	private final Map<Class<?>, Object> sharedObjects = new HashMap<>();

	private AuthenticationManager authenticationManager;

	private final List<AuthenticationProvider> authenticationProviders = new ArrayList<>();

	private boolean built = false;

	// --- Constructors ---

	public SimpleHttpSecurity() {
	}

	public SimpleHttpSecurity(Map<Class<?>, Object> sharedObjects) {
		if (sharedObjects != null) {
			this.sharedObjects.putAll(sharedObjects);
		}
	}

	// === Build lifecycle ===

	/**
	 * Builds the {@link SecurityFilterChain}. This method can only be called once.
	 *
	 * <p>Executes the full configurer lifecycle: init → beforeConfigure → configure → performBuild.
	 */
	@Override
	public SecurityFilterChain build() throws Exception {
		if (this.built) {
			throw new IllegalStateException("This SimpleHttpSecurity has already been built");
		}
		this.built = true;

		// Phase 1: INITIALIZING — init all configurers (they set up shared state)
		for (SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity> configurer : getConfigurers()) {
			configurer.init(this);
		}

		// Between init and configure: resolve AuthenticationManager
		beforeConfigure();

		// Phase 2: CONFIGURING — configure all configurers (they create and add filters)
		for (SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity> configurer : getConfigurers()) {
			configurer.configure(this);
		}

		// Phase 3: BUILDING — sort filters and build chain
		return performBuild();
	}

	/**
	 * Called between init and configure phases. Resolves the {@link AuthenticationManager}
	 * and makes it available as a shared object for configurers that need it.
	 */
	private void beforeConfigure() {
		if (this.authenticationManager != null) {
			setSharedObject(AuthenticationManager.class, this.authenticationManager);
		}
		else if (getSharedObject(AuthenticationManager.class) == null
				&& !this.authenticationProviders.isEmpty()) {
			setSharedObject(AuthenticationManager.class,
					new SimpleProviderManager(this.authenticationProviders));
		}
	}

	/**
	 * Sorts accumulated filters by their order and wraps them in a
	 * {@link SimpleDefaultSecurityFilterChain}.
	 */
	private SecurityFilterChain performBuild() {
		this.filters.sort(Comparator.comparingInt(f -> f.order));
		List<Filter> sortedFilters = new ArrayList<>(this.filters.size());
		for (OrderedFilter of : this.filters) {
			sortedFilters.add(of.filter);
		}
		return new SimpleDefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
	}

	private Collection<SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>> getConfigurers() {
		return new ArrayList<>(this.configurers.values());
	}

	// === Configurer management ===

	/**
	 * Registers a custom configurer and applies the given customizer to it.
	 * This is the general-purpose entry point for adding security features.
	 *
	 * @param configurer the configurer to register
	 * @param customizer the customizer to apply
	 * @return this builder for chaining
	 */
	@SuppressWarnings("unchecked")
	public <C extends SimpleAbstractHttpConfigurer<C>> SimpleHttpSecurity with(
			C configurer, Customizer<C> customizer) {
		Class<? extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>> key =
				(Class<? extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>>) configurer.getClass();
		configurer.setBuilder(this);
		this.configurers.put(key, configurer);
		customizer.customize(configurer);
		return this;
	}

	/**
	 * Gets an existing configurer of the given type, or applies (registers) the
	 * new one if none exists. This is the idempotent entry point used by DSL methods.
	 *
	 * <p>Calling the same DSL method twice (e.g., {@code http.formLogin(c1).formLogin(c2)})
	 * does NOT create two configurers — the second call reuses the existing one.
	 */
	@SuppressWarnings("unchecked")
	public <C extends SimpleAbstractHttpConfigurer<C>> C getOrApply(C configurer) {
		Class<? extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>> key =
				(Class<? extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>>) configurer.getClass();
		SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity> existing = this.configurers.get(key);
		if (existing != null) {
			return (C) existing;
		}
		configurer.setBuilder(this);
		this.configurers.put(key, configurer);
		return configurer;
	}

	/**
	 * Returns the configurer of the given type, or {@code null} if not registered.
	 */
	@SuppressWarnings("unchecked")
	public <C extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>> C getConfigurer(Class<C> type) {
		return (C) this.configurers.get(type);
	}

	/**
	 * Removes and returns the configurer of the given type.
	 * Used by {@link SimpleAbstractHttpConfigurer#disable()}.
	 */
	@SuppressWarnings("unchecked")
	public <C extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>> C removeConfigurer(Class<C> type) {
		return (C) this.configurers.remove(type);
	}

	// === Filter management ===

	/**
	 * Adds a filter at its well-known position in the filter chain.
	 * The filter's class must be registered in the filter order map.
	 *
	 * @throws IllegalArgumentException if the filter type has no registered order
	 */
	public SimpleHttpSecurity addFilter(Filter filter) {
		Integer order = FILTER_ORDER_MAP.get(filter.getClass().getSimpleName());
		if (order == null) {
			throw new IllegalArgumentException(
					"Filter [" + filter.getClass().getSimpleName()
							+ "] does not have a registered order. "
							+ "Use addFilterBefore or addFilterAfter instead.");
		}
		this.filters.add(new OrderedFilter(filter, order));
		return this;
	}

	/**
	 * Adds a filter immediately before the specified well-known filter type.
	 *
	 * @throws IllegalArgumentException if the reference filter has no registered order
	 */
	public SimpleHttpSecurity addFilterBefore(Filter filter, Class<? extends Filter> beforeFilter) {
		Integer order = FILTER_ORDER_MAP.get(beforeFilter.getSimpleName());
		if (order == null) {
			throw new IllegalArgumentException(
					"Cannot insert filter before [" + beforeFilter.getSimpleName()
							+ "] — it has no registered order.");
		}
		this.filters.add(new OrderedFilter(filter, order - 1));
		return this;
	}

	/**
	 * Adds a filter immediately after the specified well-known filter type.
	 *
	 * @throws IllegalArgumentException if the reference filter has no registered order
	 */
	public SimpleHttpSecurity addFilterAfter(Filter filter, Class<? extends Filter> afterFilter) {
		Integer order = FILTER_ORDER_MAP.get(afterFilter.getSimpleName());
		if (order == null) {
			throw new IllegalArgumentException(
					"Cannot insert filter after [" + afterFilter.getSimpleName()
							+ "] — it has no registered order.");
		}
		this.filters.add(new OrderedFilter(filter, order + 1));
		return this;
	}

	/**
	 * Adds a filter at the specified explicit order position.
	 * Use when the filter does not have a well-known position.
	 */
	public SimpleHttpSecurity addFilterAt(Filter filter, int order) {
		this.filters.add(new OrderedFilter(filter, order));
		return this;
	}

	// === DSL convenience methods ===

	/**
	 * Configures form-based login. The {@link SimpleFormLoginConfigurer} adds a
	 * {@code SimpleUsernamePasswordAuthenticationFilter} that processes {@code POST /login}.
	 *
	 * <pre>
	 * // Defaults
	 * http.formLogin(Customizer.withDefaults());
	 *
	 * // Custom
	 * http.formLogin(form -> form
	 *     .loginPage("/custom-login")
	 *     .defaultSuccessUrl("/dashboard"));
	 * </pre>
	 */
	public SimpleHttpSecurity formLogin(Customizer<SimpleFormLoginConfigurer> formLoginCustomizer) {
		formLoginCustomizer.customize(getOrApply(new SimpleFormLoginConfigurer()));
		return this;
	}

	/**
	 * Configures HTTP Basic authentication. The {@link SimpleHttpBasicConfigurer} adds
	 * a {@code SimpleBasicAuthenticationFilter} that processes {@code Authorization: Basic} headers.
	 *
	 * <pre>
	 * // Defaults
	 * http.httpBasic(Customizer.withDefaults());
	 *
	 * // Custom realm
	 * http.httpBasic(basic -> basic.realmName("My API"));
	 * </pre>
	 */
	public SimpleHttpSecurity httpBasic(Customizer<SimpleHttpBasicConfigurer> httpBasicCustomizer) {
		httpBasicCustomizer.customize(getOrApply(new SimpleHttpBasicConfigurer()));
		return this;
	}

	// === Authentication ===

	/**
	 * Sets the {@link AuthenticationManager} to be used. If set, this takes
	 * precedence over any registered providers.
	 */
	public SimpleHttpSecurity authenticationManager(AuthenticationManager authenticationManager) {
		this.authenticationManager = authenticationManager;
		return this;
	}

	/**
	 * Adds an {@link AuthenticationProvider} to be used when building the
	 * {@link AuthenticationManager}. Only used if no explicit AuthenticationManager
	 * is set via {@link #authenticationManager}.
	 */
	public SimpleHttpSecurity authenticationProvider(AuthenticationProvider provider) {
		this.authenticationProviders.add(provider);
		return this;
	}

	/**
	 * Sets the {@link UserDetailsService} to be used. Stored as a shared object
	 * so configurers (like form login, HTTP basic) can access it.
	 */
	public SimpleHttpSecurity userDetailsService(UserDetailsService userDetailsService) {
		setSharedObject(UserDetailsService.class, userDetailsService);
		return this;
	}

	// === Security matcher ===

	/**
	 * Sets the {@link RequestMatcher} that determines which requests this
	 * {@link SecurityFilterChain} applies to.
	 */
	public SimpleHttpSecurity securityMatcher(RequestMatcher requestMatcher) {
		this.requestMatcher = requestMatcher;
		return this;
	}

	/**
	 * Sets path patterns that determine which requests this filter chain applies to.
	 * Multiple patterns are OR'd together.
	 */
	public SimpleHttpSecurity securityMatcher(String... patterns) {
		if (patterns.length == 1) {
			this.requestMatcher = new SimplePathPatternRequestMatcher(patterns[0]);
		}
		else {
			List<RequestMatcher> matchers = new ArrayList<>();
			for (String pattern : patterns) {
				matchers.add(new SimplePathPatternRequestMatcher(pattern));
			}
			this.requestMatcher = request -> matchers.stream().anyMatch(m -> m.matches(request));
		}
		return this;
	}

	// === Shared objects ===

	/**
	 * Gets a shared object by type. Shared objects are used to pass state between
	 * configurers (e.g., the {@link AuthenticationManager}).
	 */
	@SuppressWarnings("unchecked")
	public <C> C getSharedObject(Class<C> type) {
		return (C) this.sharedObjects.get(type);
	}

	/**
	 * Sets a shared object. Configurers use this during {@code init()} to make
	 * state available to other configurers during {@code configure()}.
	 */
	public <C> void setSharedObject(Class<C> type, C object) {
		this.sharedObjects.put(type, object);
	}

	/**
	 * Returns an unmodifiable view of all shared objects.
	 */
	public Map<Class<?>, Object> getSharedObjects() {
		return Collections.unmodifiableMap(this.sharedObjects);
	}

	// === Package-private accessors for testing ===

	/**
	 * Returns the registered order for a well-known filter class name, or {@code null}.
	 */
	static Integer getFilterOrder(String filterClassName) {
		return FILTER_ORDER_MAP.get(filterClassName);
	}

	// === Inner classes ===

	/**
	 * Wraps a {@link Filter} with an integer order value for sorting.
	 * Mirrors {@code HttpSecurity.OrderedFilter} in the real framework.
	 */
	private static final class OrderedFilter {

		final Filter filter;

		final int order;

		OrderedFilter(Filter filter, int order) {
			this.filter = filter;
			this.order = order;
		}

	}

}
```

### [NEW] `simple-security-config/src/test/java/simple/security/config/FormLoginHttpBasicApiTest.java`

```java
package simple.security.config;

import java.io.IOException;
import java.util.Base64;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockFilterChain;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import simple.security.authentication.AuthenticationManager;
import simple.security.authentication.BadCredentialsException;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.authentication.dao.SimpleDaoAuthenticationProvider;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.userdetails.User;
import simple.security.crypto.password.NoOpPasswordEncoder;
import simple.security.provisioning.SimpleInMemoryUserDetailsManager;
import simple.security.web.AuthenticationEntryPoint;
import simple.security.web.SecurityFilterChain;
import simple.security.web.SimpleFilterChainProxy;
import simple.security.web.authentication.AuthenticationFailureHandler;
import simple.security.web.authentication.AuthenticationSuccessHandler;
import simple.security.web.authentication.SimpleBasicAuthenticationFilter;
import simple.security.web.authentication.SimpleUsernamePasswordAuthenticationFilter;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for Feature 6: Form Login & HTTP Basic Authentication.
 *
 * <p>These tests show how a real client would configure and use form login and
 * HTTP Basic authentication via the {@link SimpleHttpSecurity} DSL. Each test
 * builds a complete filter chain and sends mock HTTP requests through it.
 */
class FormLoginHttpBasicApiTest {

	@AfterEach
	void clearSecurityContext() {
		SecurityContextHolder.clearContext();
	}

	// ========== Form Login: Defaults ==========

	@Test
	void shouldAuthenticateWithFormLoginDefaults() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(Customizer.withDefaults());
		});

		// POST /login with correct credentials
		MockHttpServletRequest request = postLogin("user", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Default success: redirect to "/"
		// (The redirect confirms authentication succeeded — the success handler
		// only fires after the SecurityContext is populated)
		assertThat(response.getRedirectedUrl()).isEqualTo("/");
	}

	@Test
	void shouldRejectInvalidCredentialsWithFormLogin() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(Customizer.withDefaults());
		});

		// POST /login with wrong password
		MockHttpServletRequest request = postLogin("user", "wrong");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Default failure: redirect to /login?error
		assertThat(response.getRedirectedUrl()).isEqualTo("/login?error");
		// SecurityContext should be clear
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldPassThroughNonLoginRequests() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(Customizer.withDefaults());
		});

		// GET /other — should pass through the form login filter
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/other");
		MockHttpServletResponse response = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();
		proxy.doFilter(request, response, chain);

		// The downstream filter chain should have been invoked
		assertThat(chain.getRequest()).isNotNull();
	}

	// ========== Form Login: Custom Configuration ==========

	@Test
	void shouldCustomizeFormLoginParameters() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(form -> form
					.usernameParameter("email")
					.passwordParameter("pass"));
		});

		// POST /login with custom parameter names
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("email", "user");
		request.setParameter("pass", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Redirect to "/" confirms authentication succeeded with custom params
		assertThat(response.getRedirectedUrl()).isEqualTo("/");
	}

	@Test
	void shouldRedirectToCustomSuccessUrl() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(form -> form.defaultSuccessUrl("/dashboard"));
		});

		MockHttpServletRequest request = postLogin("user", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getRedirectedUrl()).isEqualTo("/dashboard");
	}

	@Test
	void shouldRedirectToCustomFailureUrl() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(form -> form.failureUrl("/auth/failed"));
		});

		MockHttpServletRequest request = postLogin("user", "wrong");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getRedirectedUrl()).isEqualTo("/auth/failed");
	}

	@Test
	void shouldUseCustomLoginProcessingUrl() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(form -> form.loginProcessingUrl("/authenticate"));
		});

		// POST to custom URL
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/authenticate");
		request.setParameter("username", "user");
		request.setParameter("password", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		// Redirect to "/" confirms the custom processing URL was matched
		assertThat(response.getRedirectedUrl()).isEqualTo("/");
	}

	@Test
	void shouldUseCustomSuccessHandler() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(form -> form.successHandler(
					(request, response, auth) -> {
						response.setStatus(200);
						response.getWriter().write("Welcome " + auth.getName());
					}));
		});

		MockHttpServletRequest request = postLogin("user", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getStatus()).isEqualTo(200);
		assertThat(response.getContentAsString()).isEqualTo("Welcome user");
	}

	@Test
	void shouldUseCustomFailureHandler() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(form -> form.failureHandler(
					(request, response, exception) -> {
						response.setStatus(403);
						response.getWriter().write("Login failed: " + exception.getMessage());
					}));
		});

		MockHttpServletRequest request = postLogin("user", "wrong");
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getStatus()).isEqualTo(403);
		assertThat(response.getContentAsString()).startsWith("Login failed:");
	}

	// ========== HTTP Basic: Defaults ==========

	@Test
	void shouldAuthenticateWithHttpBasicHeader() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.httpBasic(Customizer.withDefaults());
		});

		// GET /api/data with Basic auth header
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		// Capture the authentication inside the chain (before proxy clears context)
		AtomicReference<Authentication> captured = new AtomicReference<>();
		FilterChain capturingChain = (req, res) ->
				captured.set(SecurityContextHolder.getContext().getAuthentication());
		proxy.doFilter(request, response, capturingChain);

		// Chain continues (Basic auth always continues)
		assertThat(captured.get()).isNotNull();
		assertThat(captured.get().getName()).isEqualTo("user");
		assertThat(captured.get().isAuthenticated()).isTrue();
	}

	@Test
	void shouldChallengeWith401WhenNoBasicHeader() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.httpBasic(Customizer.withDefaults());
		});

		// GET /api/data with no auth header — filter passes through,
		// no authentication happens. Chain continues unauthenticated.
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		MockHttpServletResponse response = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();
		proxy.doFilter(request, response, chain);

		// The Basic filter is a no-op when there's no header — chain continues
		assertThat(chain.getRequest()).isNotNull();
		// No authentication set
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldRejectInvalidBasicCredentials() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.httpBasic(Customizer.withDefaults());
		});

		// GET /api/data with wrong password
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", basicAuth("user", "wrong"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();
		proxy.doFilter(request, response, chain);

		// Entry point sends 401 with WWW-Authenticate
		assertThat(response.getStatus()).isEqualTo(401);
		assertThat(response.getHeader("WWW-Authenticate")).contains("Basic realm=\"Realm\"");
	}

	@Test
	void shouldUseCustomRealmName() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.httpBasic(basic -> basic.realmName("My API"));
		});

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "wrong"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getHeader("WWW-Authenticate")).isEqualTo("Basic realm=\"My API\"");
	}

	@Test
	void shouldSkipReauthenticationForSameUser() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.httpBasic(Customizer.withDefaults());
		});

		// Pre-authenticate the same user in the SecurityContext
		Authentication existing = UsernamePasswordAuthenticationToken.authenticated(
				"user", "password", List.of(new SimpleGrantedAuthority("ROLE_USER")));
		SecurityContextHolder.getContext().setAuthentication(existing);

		// Send another request with the same user's Basic creds
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		// Capture authentication inside the chain (before proxy clears context)
		AtomicReference<Authentication> captured = new AtomicReference<>();
		FilterChain capturingChain = (req, res) ->
				captured.set(SecurityContextHolder.getContext().getAuthentication());
		proxy.doFilter(request, response, capturingChain);

		// Chain continues, and the existing authentication is preserved (not re-authenticated)
		assertThat(captured.get()).isSameAs(existing);
	}

	// ========== Both Form Login and HTTP Basic ==========

	@Test
	void shouldConfigureBothFormLoginAndHttpBasic() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy(http -> {
			http.authenticationManager(testAuthenticationManager());
			http.formLogin(Customizer.withDefaults());
			http.httpBasic(Customizer.withDefaults());
		});

		// Form login works
		MockHttpServletRequest formRequest = postLogin("user", "password");
		MockHttpServletResponse formResponse = new MockHttpServletResponse();
		proxy.doFilter(formRequest, formResponse, new MockFilterChain());
		assertThat(formResponse.getRedirectedUrl()).isEqualTo("/");

		SecurityContextHolder.clearContext();

		// HTTP Basic also works
		MockHttpServletRequest basicRequest = new MockHttpServletRequest("GET", "/api");
		basicRequest.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse basicResponse = new MockHttpServletResponse();
		AtomicReference<Authentication> captured = new AtomicReference<>();
		FilterChain capturingChain = (req, res) ->
				captured.set(SecurityContextHolder.getContext().getAuthentication());
		proxy.doFilter(basicRequest, basicResponse, capturingChain);
		assertThat(captured.get()).isNotNull();
		assertThat(captured.get().getName()).isEqualTo("user");
	}

	// ========== Filter chain structure ==========

	@Test
	void shouldAddFormLoginFilterAtCorrectPosition() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.authenticationManager(testAuthenticationManager());
		http.formLogin(Customizer.withDefaults());

		SecurityFilterChain chain = http.build();
		List<Filter> filters = chain.getFilters();

		assertThat(filters).hasSize(1);
		assertThat(filters.get(0)).isInstanceOf(SimpleUsernamePasswordAuthenticationFilter.class);
	}

	@Test
	void shouldAddBasicFilterAtCorrectPosition() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.authenticationManager(testAuthenticationManager());
		http.httpBasic(Customizer.withDefaults());

		SecurityFilterChain chain = http.build();
		List<Filter> filters = chain.getFilters();

		assertThat(filters).hasSize(1);
		assertThat(filters.get(0)).isInstanceOf(SimpleBasicAuthenticationFilter.class);
	}

	@Test
	void shouldOrderFormLoginBeforeBasic() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.authenticationManager(testAuthenticationManager());
		http.httpBasic(Customizer.withDefaults());
		http.formLogin(Customizer.withDefaults());

		SecurityFilterChain chain = http.build();
		List<Filter> filters = chain.getFilters();

		assertThat(filters).hasSize(2);
		// Form login (300) should come before Basic (400) regardless of registration order
		assertThat(filters.get(0)).isInstanceOf(SimpleUsernamePasswordAuthenticationFilter.class);
		assertThat(filters.get(1)).isInstanceOf(SimpleBasicAuthenticationFilter.class);
	}

	// ========== Entry point registration ==========

	@Test
	void shouldRegisterFormLoginEntryPointAsSharedObject() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.authenticationManager(testAuthenticationManager());
		http.formLogin(Customizer.withDefaults());
		http.build();

		AuthenticationEntryPoint entryPoint = http.getSharedObject(AuthenticationEntryPoint.class);
		assertThat(entryPoint).isNotNull();
	}

	@Test
	void shouldRegisterCustomEntryPoint() throws Exception {
		AuthenticationEntryPoint custom = (req, res, ex) -> res.sendError(418);

		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.authenticationManager(testAuthenticationManager());
		http.formLogin(form -> form.authenticationEntryPoint(custom));
		http.build();

		assertThat(http.getSharedObject(AuthenticationEntryPoint.class)).isSameAs(custom);
	}

	// ========== Test helpers ==========

	/**
	 * Builds a {@link SimpleFilterChainProxy} from a DSL configuration lambda.
	 */
	private SimpleFilterChainProxy buildProxy(HttpSecurityConfigurer configurer) throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		configurer.configure(http);
		SecurityFilterChain chain = http.build();
		return new SimpleFilterChainProxy(List.of(chain));
	}

	/**
	 * Creates a mock POST /login request with form parameters.
	 */
	private MockHttpServletRequest postLogin(String username, String password) {
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("username", username);
		request.setParameter("password", password);
		return request;
	}

	/**
	 * Encodes username:password as a Basic auth header value.
	 */
	private String basicAuth(String username, String password) {
		return "Basic " + Base64.getEncoder().encodeToString(
				(username + ":" + password).getBytes());
	}

	/**
	 * Creates an AuthenticationManager backed by an in-memory user store
	 * with a single user "user"/"password" having ROLE_USER.
	 */
	private AuthenticationManager testAuthenticationManager() {
		var userDetailsService = new SimpleInMemoryUserDetailsManager(
				User.withUsername("user")
						.password("password")
						.authorities("ROLE_USER")
						.build()
		);

		var provider = new SimpleDaoAuthenticationProvider(userDetailsService, NoOpPasswordEncoder.INSTANCE);

		return provider::authenticate;
	}

	@FunctionalInterface
	private interface HttpSecurityConfigurer {
		void configure(SimpleHttpSecurity http) throws Exception;
	}

}
```

### [NEW] `simple-security-web/src/test/java/simple/security/web/authentication/SimpleUsernamePasswordAuthenticationFilterTest.java`

```java
package simple.security.web.authentication;

import java.io.IOException;
import java.util.List;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import simple.security.authentication.AuthenticationServiceException;
import simple.security.authentication.BadCredentialsException;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContextHolder;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;

/**
 * Internal unit tests for {@link SimpleUsernamePasswordAuthenticationFilter}.
 *
 * <p>Tests the filter in isolation, focusing on credential extraction, request
 * matching, and success/failure flows.
 */
class SimpleUsernamePasswordAuthenticationFilterTest {

	private SimpleUsernamePasswordAuthenticationFilter filter;

	@BeforeEach
	void setUp() {
		filter = new SimpleUsernamePasswordAuthenticationFilter();
		// AuthenticationManager that accepts "user"/"password"
		filter.setAuthenticationManager(auth -> {
			UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) auth;
			if ("user".equals(token.getName()) && "password".equals(token.getCredentials())) {
				return UsernamePasswordAuthenticationToken.authenticated(
						"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
			}
			throw new BadCredentialsException("Bad credentials");
		});
	}

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	// --- Request matching ---

	@Test
	void shouldOnlyMatchPostLogin() throws Exception {
		FilterChain chain = mock(FilterChain.class);

		// GET /login should pass through
		MockHttpServletRequest getRequest = new MockHttpServletRequest("GET", "/login");
		MockHttpServletResponse getResponse = new MockHttpServletResponse();
		filter.doFilter(getRequest, getResponse, chain);
		verify(chain).doFilter(getRequest, getResponse);
	}

	@Test
	void shouldNotMatchDifferentUrl() throws Exception {
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/other");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);
		verify(chain).doFilter(request, response);
	}

	// --- Credential extraction ---

	@Test
	void shouldExtractUsernameAndPassword() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("username", "user");
		request.setParameter("password", "password");

		Authentication result = filter.attemptAuthentication(request, new MockHttpServletResponse());

		assertThat(result).isNotNull();
		assertThat(result.getName()).isEqualTo("user");
		assertThat(result.isAuthenticated()).isTrue();
	}

	@Test
	void shouldTrimUsername() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("username", "  user  ");
		request.setParameter("password", "password");

		Authentication result = filter.attemptAuthentication(request, new MockHttpServletResponse());
		assertThat(result.getName()).isEqualTo("user");
	}

	@Test
	void shouldUseEmptyStringForMissingParameters() throws Exception {
		// Missing both parameters — should use empty strings
		filter.setAuthenticationManager(auth -> {
			UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) auth;
			assertThat(token.getName()).isEmpty();
			assertThat(token.getCredentials()).isEqualTo("");
			throw new BadCredentialsException("Expected");
		});

		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.setAuthenticationFailureHandler((req, res, ex) -> res.setStatus(401));
		filter.doFilter(request, response, mock(FilterChain.class));

		assertThat(response.getStatus()).isEqualTo(401);
	}

	@Test
	void shouldUseCustomParameterNames() throws Exception {
		filter.setUsernameParameter("email");
		filter.setPasswordParameter("secret");

		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("email", "user");
		request.setParameter("secret", "password");

		Authentication result = filter.attemptAuthentication(request, new MockHttpServletResponse());
		assertThat(result.getName()).isEqualTo("user");
	}

	// --- Success / failure handling ---

	@Test
	void shouldSetSecurityContextOnSuccess() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("username", "user");
		request.setParameter("password", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, mock(FilterChain.class));

		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNotNull();
		assertThat(SecurityContextHolder.getContext().getAuthentication().getName()).isEqualTo("user");
	}

	@Test
	void shouldClearSecurityContextOnFailure() throws Exception {
		// Pre-populate context
		SecurityContextHolder.getContext().setAuthentication(
				UsernamePasswordAuthenticationToken.authenticated("old", null, List.of()));

		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("username", "user");
		request.setParameter("password", "wrong");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.setAuthenticationFailureHandler((req, res, ex) -> res.setStatus(401));
		filter.doFilter(request, response, mock(FilterChain.class));

		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldNotContinueChainOnSuccess() throws Exception {
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("POST", "/login");
		request.setParameter("username", "user");
		request.setParameter("password", "password");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// The chain should NOT be called — success handler takes over
		verifyNoInteractions(chain);
	}

}
```

### [NEW] `simple-security-web/src/test/java/simple/security/web/authentication/SimpleBasicAuthenticationFilterTest.java`

```java
package simple.security.web.authentication;

import java.util.Base64;
import java.util.List;

import jakarta.servlet.FilterChain;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import simple.security.authentication.BadCredentialsException;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContextHolder;
import simple.security.web.AuthenticationEntryPoint;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

/**
 * Internal unit tests for {@link SimpleBasicAuthenticationFilter}.
 *
 * <p>Tests the filter in isolation, focusing on Base64 decoding, header parsing,
 * success/failure flows, and the ignoreFailure toggle.
 */
class SimpleBasicAuthenticationFilterTest {

	private AuthenticationEntryPoint entryPoint;

	@BeforeEach
	void setUp() {
		entryPoint = mock(AuthenticationEntryPoint.class);
	}

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	// --- No header: pass through ---

	@Test
	void shouldPassThroughWhenNoAuthorizationHeader() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		verify(chain).doFilter(request, response);
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldPassThroughForNonBasicAuthHeader() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", "Bearer some-jwt-token");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		verify(chain).doFilter(request, response);
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	// --- Success: authenticate and continue chain ---

	@Test
	void shouldAuthenticateAndContinueChain() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Chain always continues on success
		verify(chain).doFilter(request, response);
		// SecurityContext is populated
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth).isNotNull();
		assertThat(auth.getName()).isEqualTo("user");
		assertThat(auth.isAuthenticated()).isTrue();
	}

	@Test
	void shouldWorkWithAnyHttpMethod() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		// PUT request with Basic auth should also work
		MockHttpServletRequest request = new MockHttpServletRequest("PUT", "/api/resource");
		request.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		verify(chain).doFilter(request, response);
		assertThat(SecurityContextHolder.getContext().getAuthentication().getName()).isEqualTo("user");
	}

	// --- Failure: with entry point ---

	@Test
	void shouldInvokeEntryPointOnFailure() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "wrong"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Entry point should be called
		verify(entryPoint).commence(eq(request), eq(response), any());
		// Chain should NOT continue
		verify(chain, never()).doFilter(any(), any());
		// SecurityContext should be cleared
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	// --- Failure: ignoreFailure (no entry point) ---

	@Test
	void shouldContinueChainOnFailureWhenIgnoringFailure() throws Exception {
		// One-arg constructor sets ignoreFailure = true
		SimpleBasicAuthenticationFilter filter = filterIgnoringFailure();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "wrong"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Chain continues even after failure
		verify(chain).doFilter(request, response);
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	// --- Skip re-authentication ---

	@Test
	void shouldSkipReauthenticationForSameUser() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();

		// Pre-authenticate
		Authentication existing = UsernamePasswordAuthenticationToken.authenticated(
				"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
		SecurityContextHolder.getContext().setAuthentication(existing);

		FilterChain chain = mock(FilterChain.class);
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Chain continues with the existing authentication preserved
		verify(chain).doFilter(request, response);
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isSameAs(existing);
	}

	@Test
	void shouldReauthenticateForDifferentUser() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();

		// Pre-authenticate as "admin"
		Authentication existing = UsernamePasswordAuthenticationToken.authenticated(
				"admin", null, List.of(new SimpleGrantedAuthority("ROLE_ADMIN")));
		SecurityContextHolder.getContext().setAuthentication(existing);

		FilterChain chain = mock(FilterChain.class);
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "password"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Should re-authenticate as "user"
		assertThat(SecurityContextHolder.getContext().getAuthentication().getName()).isEqualTo("user");
	}

	// --- Edge cases ---

	@Test
	void shouldHandleMalformedBase64() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", "Basic not-valid-base64!!!");
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Should invoke entry point with BadCredentialsException
		verify(entryPoint).commence(eq(request), eq(response), any());
	}

	@Test
	void shouldHandleMissingColonInCredentials() throws Exception {
		SimpleBasicAuthenticationFilter filter = filterWithEntryPoint();
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		// Base64 of "nocolon" (no colon separator)
		request.addHeader("Authorization", "Basic " +
				Base64.getEncoder().encodeToString("nocolon".getBytes()));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		verify(entryPoint).commence(eq(request), eq(response), any());
	}

	@Test
	void shouldHandlePasswordWithColon() throws Exception {
		// Password containing colons: "pass:word:123"
		SimpleBasicAuthenticationFilter filter = new SimpleBasicAuthenticationFilter(
				auth -> {
					UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) auth;
					if ("user".equals(token.getName()) && "pass:word:123".equals(token.getCredentials())) {
						return UsernamePasswordAuthenticationToken.authenticated(
								"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
					}
					throw new BadCredentialsException("Bad credentials");
				},
				entryPoint);
		FilterChain chain = mock(FilterChain.class);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api");
		request.addHeader("Authorization", basicAuth("user", "pass:word:123"));
		MockHttpServletResponse response = new MockHttpServletResponse();
		filter.doFilter(request, response, chain);

		// Should parse correctly: split on first colon only
		verify(chain).doFilter(request, response);
		assertThat(SecurityContextHolder.getContext().getAuthentication().getName()).isEqualTo("user");
	}

	// ========== Helpers ==========

	private SimpleBasicAuthenticationFilter filterWithEntryPoint() {
		return new SimpleBasicAuthenticationFilter(testAuthManager(), entryPoint);
	}

	private SimpleBasicAuthenticationFilter filterIgnoringFailure() {
		return new SimpleBasicAuthenticationFilter(testAuthManager());
	}

	private simple.security.authentication.AuthenticationManager testAuthManager() {
		return auth -> {
			UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) auth;
			if ("user".equals(token.getName()) && "password".equals(token.getCredentials())) {
				return UsernamePasswordAuthenticationToken.authenticated(
						"user", null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
			}
			throw new BadCredentialsException("Bad credentials");
		};
	}

	private String basicAuth(String username, String password) {
		return "Basic " + Base64.getEncoder().encodeToString(
				(username + ":" + password).getBytes());
	}

}
```
