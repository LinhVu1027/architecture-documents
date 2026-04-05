# Chapter 15: Testing Support

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Pattern 1: @WithMockUser on a test method
@Test
@WithMockUser(username = "admin", roles = "ADMIN")
void adminCanAccessDashboard() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getName()).isEqualTo("admin");
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactly("ROLE_ADMIN");
}

// Pattern 2: MockMvc with user() post-processor
mockMvc.perform(get("/api/data")
        .with(user("john").roles("USER")))
        .andExpect(status().isOk());

// Pattern 3: MockMvc with csrf() and jwt()
mockMvc.perform(post("/api/orders")
        .with(csrf())
        .with(user("john").roles("USER")))
        .andExpect(status().isCreated());

mockMvc.perform(get("/api/data")
        .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_read"))))
        .andExpect(status().isOk());
```

Questions to guide your thinking:
- Why does `@WithMockUser` need a JUnit 5 extension? An annotation alone cannot modify runtime state -- something must detect the annotation, build a `SecurityContext`, set it on `SecurityContextHolder` before the test, and clear it after. What lifecycle hook provides the right timing?
- Why are MockMvc post-processors separate from annotation-based testing? Annotations work at the JUnit level (before/after the test method). Post-processors work at the MockMvc level (before the request enters the filter chain). These are two different integration points for two different test styles.
- How does `csrf()` know what token value to put on the request? It does not need to know an existing token -- it generates a fresh UUID and places it as both a request attribute (so the CSRF filter loads it as the "expected" token) and a request parameter/header (so the CSRF filter sees it as the "actual" token). The two values match, so validation passes.
- Why does the `@WithSecurityContext` meta-annotation exist rather than having `@WithMockUser` directly implement the factory logic? Extensibility -- any developer can create their own custom annotation (e.g., `@WithMockAdmin`) by meta-annotating it with `@WithSecurityContext` and pointing to a custom factory. The extension only needs to understand `@WithSecurityContext`, not every possible test annotation.

---

## The API Contract

Feature 15 introduces a dedicated `simple-security-test` module that provides three distinct
API surfaces for testing secured code. Each surface targets a different testing style.

### Surface 1: @WithMockUser annotation

The annotation-based API lets you declare a mock authenticated user directly on test methods
or classes. A JUnit 5 extension detects the annotation and populates `SecurityContextHolder`
before each test.

```java
// simple/security/test/context/support/WithMockUser.java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@WithSecurityContext(factory = SimpleWithMockUserSecurityContextFactory.class)
@ExtendWith(WithSecurityContextExtension.class)
public @interface WithMockUser {
    String value() default "user";
    String username() default "";
    String[] roles() default {"USER"};
    String[] authorities() default {};
    String password() default "password";
}
```

The annotation carries two meta-annotations that drive the machinery:
- `@WithSecurityContext(factory = ...)` -- tells the extension which factory creates the `SecurityContext`
- `@ExtendWith(WithSecurityContextExtension.class)` -- registers the JUnit 5 extension that processes it

### Surface 2: MockMvc request post-processors

The `SimpleSecurityMockMvcRequestPostProcessors` class provides static factory methods
that create `RequestPostProcessor` instances for use with MockMvc's `.with(...)` method.

```java
// simple/security/test/web/servlet/request/SimpleSecurityMockMvcRequestPostProcessors.java
public final class SimpleSecurityMockMvcRequestPostProcessors {
    public static CsrfRequestPostProcessor csrf() { ... }
    public static UserRequestPostProcessor user(String username) { ... }
    public static RequestPostProcessor user(UserDetails userDetails) { ... }
    public static RequestPostProcessor authentication(Authentication authentication) { ... }
    public static JwtRequestPostProcessor jwt() { ... }
}
```

Each post-processor modifies the `MockHttpServletRequest` before it enters the security
filter chain:
- `csrf()` -- adds a valid CSRF token as both a request attribute and a parameter/header
- `user(String)` -- creates a `UsernamePasswordAuthenticationToken` and sets it on `SecurityContextHolder`
- `jwt()` -- creates a `JwtAuthenticationToken` with mock JWT claims and sets it on `SecurityContextHolder`
- `authentication(Authentication)` -- sets an arbitrary `Authentication` on `SecurityContextHolder`

### Surface 3: MockMvc configurer

The `SimpleSecurityMockMvcConfigurer` integrates the security filter chain with MockMvc
so every test request passes through the security filters:

```java
// simple/security/test/web/servlet/setup/SimpleSecurityMockMvcConfigurer.java
public final class SimpleSecurityMockMvcConfigurer extends MockMvcConfigurerAdapter {
    public static SimpleSecurityMockMvcConfigurer springSecurity(Filter securityFilter) { ... }
}
```

Usage:

```java
MockMvc mockMvc = MockMvcBuilders
        .standaloneSetup(controller)
        .apply(springSecurity(filterChainProxy))
        .build();
```

### Key types

```java
// Annotation: declares a mock user on a test
@WithMockUser(username = "admin", roles = "ADMIN")

// Meta-annotation: connects any test annotation to a SecurityContext factory
@WithSecurityContext(factory = SimpleWithMockUserSecurityContextFactory.class)

// Factory interface: creates SecurityContext from annotation attributes
WithSecurityContextFactory<WithMockUser> factory = new SimpleWithMockUserSecurityContextFactory();
SecurityContext context = factory.createSecurityContext(annotation);

// Post-processors: modify MockHttpServletRequest before filter chain
csrf().postProcessRequest(request);
user("john").roles("ADMIN").postProcessRequest(request);
jwt().jwt(builder -> builder.subject("admin")).postProcessRequest(request);
```

---

## Client Usage & Tests

The tests define the behavioral contract from a client's perspective. There are two test
classes -- one for annotation-based testing and one for MockMvc post-processors.

### @WithMockUser defaults

```java
@Test
@WithMockUser
void shouldSetUpDefaultUser() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth).isNotNull();
    assertThat(auth.isAuthenticated()).isTrue();
    assertThat(auth.getName()).isEqualTo("user");
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactly("ROLE_USER");
}
```

With no arguments, `@WithMockUser` creates a user named `"user"` with password `"password"`
and a single `ROLE_USER` authority. The authentication is a fully populated
`UsernamePasswordAuthenticationToken` with a `User` principal -- the same types your
production code would see.

### Custom username and roles

```java
@Test
@WithMockUser("admin")
void shouldUseValueAsUsername() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getName()).isEqualTo("admin");
}

@Test
@WithMockUser(roles = {"ADMIN", "MANAGER"})
void shouldSupportMultipleRoles() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactlyInAnyOrder("ROLE_ADMIN", "ROLE_MANAGER");
}
```

The `value()` attribute is an alias for `username()`. If `username()` is explicitly set
to a non-empty string, it takes precedence. The `roles()` attribute auto-prefixes each
value with `"ROLE_"` -- this mirrors how `User.withUsername(...).roles(...)` works in
production code.

### Authorities vs. roles

```java
@Test
@WithMockUser(authorities = {"ORDER_READ", "ORDER_WRITE"})
void shouldSupportMultipleAuthorities() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactlyInAnyOrder("ORDER_READ", "ORDER_WRITE");
}
```

The `authorities()` attribute uses strings verbatim -- no `ROLE_` prefix. This is for
fine-grained authorities like `"ORDER_READ"`. You cannot set both `roles()` and
`authorities()` -- the factory throws `IllegalStateException` if you try.

### Context isolation

```java
@Test
void shouldNotHaveAuthenticationWithoutAnnotation() {
    // No @WithMockUser -- SecurityContext should be empty
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth).isNull();
}
```

The extension clears `SecurityContextHolder` after each test, so tests without
`@WithMockUser` see an empty context. This proves the cleanup works -- no leakage
between tests.

### MockMvc: csrf()

```java
@Test
void csrfShouldAddTokenAsRequestAttribute() {
    MockHttpServletRequest request = csrf()
            .postProcessRequest(new MockHttpServletRequest());

    CsrfToken token = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
    assertThat(token).isNotNull();
    assertThat(token.getToken()).isNotEmpty();
    assertThat(token.getHeaderName()).isEqualTo("X-CSRF-TOKEN");
    assertThat(token.getParameterName()).isEqualTo("_csrf");
}

@Test
void csrfAsHeaderShouldAddTokenAsHeader() {
    MockHttpServletRequest request = csrf().asHeader()
            .postProcessRequest(new MockHttpServletRequest());

    CsrfToken token = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
    String headerValue = request.getHeader("X-CSRF-TOKEN");
    assertThat(headerValue).isEqualTo(token.getToken());
    // Should NOT be set as parameter when using asHeader()
    assertThat(request.getParameter("_csrf")).isNull();
}
```

The `csrf()` post-processor generates a random UUID token, wraps it in a `DefaultCsrfToken`,
and sets it as a request attribute. By default, the token value is also set as a request
parameter (`_csrf`). Calling `.asHeader()` sends it as the `X-CSRF-TOKEN` header instead.
The `useInvalidToken()` method substitutes `"invalid-token"` for the UUID -- useful for
testing that CSRF rejection actually works.

### MockMvc: user()

```java
@Test
void userWithCustomRoles() {
    user("admin").roles("ADMIN", "MANAGER")
            .postProcessRequest(new MockHttpServletRequest());

    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getName()).isEqualTo("admin");
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactlyInAnyOrder("ROLE_ADMIN", "ROLE_MANAGER");
}

@Test
void userShouldHaveUserDetailsPrincipal() {
    user("john").roles("USER")
            .postProcessRequest(new MockHttpServletRequest());

    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getPrincipal()).isInstanceOf(UserDetails.class);
    UserDetails principal = (UserDetails) auth.getPrincipal();
    assertThat(principal.getUsername()).isEqualTo("john");
}
```

The `user()` post-processor creates a `User` principal wrapped in an authenticated
`UsernamePasswordAuthenticationToken`, then sets it on `SecurityContextHolder`. The
builder methods (`.roles(...)`, `.authorities(...)`, `.password(...)`) follow the same
conventions as `@WithMockUser` -- roles are auto-prefixed, authorities are verbatim,
and you cannot pass `"ROLE_ADMIN"` to `.roles()` (it throws `IllegalArgumentException`).

### MockMvc: jwt()

```java
@Test
void jwtWithCustomClaims() {
    jwt().jwt(builder -> builder
                    .subject("admin")
                    .claim("scope", "read write"))
            .postProcessRequest(new MockHttpServletRequest());

    JwtAuthenticationToken auth = (JwtAuthenticationToken) SecurityContextHolder
            .getContext().getAuthentication();
    assertThat(auth.getName()).isEqualTo("admin");
    Jwt token = auth.getToken();
    assertThat(token.getSubject()).isEqualTo("admin");
    assertThat(token.<String>getClaim("scope")).isEqualTo("read write");
}

@Test
void jwtWithPreBuiltJwt() {
    Jwt prebuilt = Jwt.withTokenValue("my-custom-token")
            .header("alg", "RS256")
            .subject("service-account")
            .claim("aud", "my-api")
            .build();

    jwt().jwt(prebuilt)
            .authorities(new SimpleGrantedAuthority("ROLE_SERVICE"))
            .postProcessRequest(new MockHttpServletRequest());

    JwtAuthenticationToken auth = (JwtAuthenticationToken) SecurityContextHolder
            .getContext().getAuthentication();
    assertThat(auth.getToken().getTokenValue()).isEqualTo("my-custom-token");
    assertThat(auth.getName()).isEqualTo("service-account");
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactly("ROLE_SERVICE");
}
```

The `jwt()` post-processor defaults to a mock JWT with subject `"user"`, scope `"read"`,
and algorithm `"none"`. The builder lambda lets you customize claims and headers. You can
also pass a pre-built `Jwt` object directly. The `.authorities(...)` method overrides the
default `SCOPE_read` authority -- this is useful when your test needs specific authority
values that do not correspond to JWT scopes.

### MockMvc: user(UserDetails) and authentication()

```java
@Test
void userDetailsShouldSetSecurityContext() {
    UserDetails userDetails = User.withUsername("admin")
            .password("secret")
            .roles("ADMIN")
            .build();

    user(userDetails).postProcessRequest(new MockHttpServletRequest());

    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth.getName()).isEqualTo("admin");
    assertThat(auth.getAuthorities()).extracting("authority")
            .containsExactly("ROLE_ADMIN");
}

@Test
void authenticationShouldSetArbitraryAuth() {
    var token = UsernamePasswordAuthenticationToken
            .authenticated("custom-principal", "creds",
                    List.of(new SimpleGrantedAuthority("CUSTOM")));

    authentication(token).postProcessRequest(new MockHttpServletRequest());

    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    assertThat(auth).isSameAs(token);
    assertThat(auth.getPrincipal()).isEqualTo("custom-principal");
}
```

The `user(UserDetails)` overload accepts a pre-built `UserDetails` object, which is
useful when your test needs a user built with the `User.withUsername(...)` builder from
Feature 2. The `authentication()` method is the most general -- it sets any `Authentication`
implementation directly. This is the escape hatch for custom authentication types that
do not fit the user/JWT patterns.

---

## Implementing the Call Chain

Feature 15 has three independent call chains -- one for each API surface. Let us walk
through each one.

### Path 1: Annotation-based (@WithMockUser)

```
Developer adds @WithMockUser(username = "admin", roles = "ADMIN") to a test method
  |
  +---> @WithMockUser is meta-annotated with:
  |       @ExtendWith(WithSecurityContextExtension.class)  -- registers JUnit 5 extension
  |       @WithSecurityContext(factory = SimpleWithMockUserSecurityContextFactory.class)
  |
  +---> JUnit 5 calls WithSecurityContextExtension.beforeEach(context)
  |       |
  |       +---> Scans test METHOD for annotations carrying @WithSecurityContext
  |       |       Found @WithMockUser -> its @WithSecurityContext points to factory
  |       |
  |       +---> Instantiates SimpleWithMockUserSecurityContextFactory via reflection
  |       |       factoryClass.getDeclaredConstructor().newInstance()
  |       |
  |       +---> Calls factory.createSecurityContext(annotation)
  |       |       |
  |       |       +---> Resolves username: annotation.username() if non-empty, else annotation.value()
  |       |       +---> Resolves authorities: roles (prefixed with ROLE_) or raw authorities
  |       |       +---> Creates User principal with (username, password, authorities)
  |       |       +---> Creates UsernamePasswordAuthenticationToken.authenticated(principal, password, authorities)
  |       |       +---> Creates empty SecurityContext, sets authentication
  |       |       +---> Returns populated SecurityContext
  |       |
  |       +---> SecurityContextHolder.setContext(securityContext)
  |
  +---> Test method runs -- SecurityContextHolder.getContext().getAuthentication() is populated
  |
  +---> JUnit 5 calls WithSecurityContextExtension.afterEach(context)
          SecurityContextHolder.clearContext()  -- prevents leakage between tests
```

**Method vs. class annotations**: The extension checks the test method first, then falls
back to the test class. This means you can put `@WithMockUser` on the class to set a
default user, then override it per-method:

```java
@WithMockUser  // default "user" for all tests
class MySecurityTests {

    @Test
    void regularUser() {
        // runs as "user" with ROLE_USER
    }

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void adminUser() {
        // runs as "admin" with ROLE_ADMIN (method overrides class)
    }
}
```

### 15a. The extension: WithSecurityContextExtension

The extension implements `BeforeEachCallback` and `AfterEachCallback` -- the two lifecycle
hooks that bracket every test method:

```java
public class WithSecurityContextExtension implements BeforeEachCallback, AfterEachCallback {

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        // Method-level annotation takes precedence over class-level
        SecurityContext securityContext = findAndCreateSecurityContext(
                context.getRequiredTestMethod());
        if (securityContext == null) {
            securityContext = findAndCreateSecurityContext(
                    context.getRequiredTestClass());
        }
        if (securityContext != null) {
            SecurityContextHolder.setContext(securityContext);
        }
    }

    @Override
    public void afterEach(ExtensionContext context) {
        SecurityContextHolder.clearContext();
    }
```

The `findAndCreateSecurityContext` method is the annotation scanning logic:

```java
    @SuppressWarnings({"unchecked", "rawtypes"})
    private SecurityContext findAndCreateSecurityContext(AnnotatedElement element) throws Exception {
        for (Annotation annotation : element.getAnnotations()) {
            WithSecurityContext withSecurityContext = annotation.annotationType()
                    .getAnnotation(WithSecurityContext.class);
            if (withSecurityContext != null) {
                Class<? extends WithSecurityContextFactory> factoryClass =
                        withSecurityContext.factory();
                WithSecurityContextFactory factory = factoryClass.getDeclaredConstructor()
                        .newInstance();
                return factory.createSecurityContext(annotation);
            }
        }
        return null;
    }
```

**Why iterate through all annotations on the element?** The extension does not look for
`@WithMockUser` specifically. It looks for *any* annotation that carries `@WithSecurityContext`.
This is what makes the system extensible -- if you create `@WithMockAdmin` and meta-annotate
it with `@WithSecurityContext(factory = YourFactory.class)`, this exact same extension
will process it automatically.

**Why reflective instantiation?** The real Spring Security test module injects factories
through the Spring application context, which enables `@Autowired` dependencies. Our
simplified version uses `getDeclaredConstructor().newInstance()` because we do not require
a Spring context for testing. This is a deliberate simplification -- it keeps the test
module independent of any application context setup.

### 15b. The factory: SimpleWithMockUserSecurityContextFactory

The factory reads the `@WithMockUser` annotation and builds a `SecurityContext`:

```java
public final class SimpleWithMockUserSecurityContextFactory
        implements WithSecurityContextFactory<WithMockUser> {

    @Override
    public SecurityContext createSecurityContext(WithMockUser annotation) {
        String username = annotation.username().isEmpty()
                ? annotation.value()
                : annotation.username();

        List<GrantedAuthority> authorities = resolveAuthorities(annotation);

        User principal = new User(username, annotation.password(), authorities);

        var authentication = UsernamePasswordAuthenticationToken.authenticated(
                principal, annotation.password(), authorities);

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        return context;
    }
```

The authority resolution logic enforces the mutual exclusivity of `roles()` and `authorities()`:

```java
    private List<GrantedAuthority> resolveAuthorities(WithMockUser annotation) {
        String[] authorities = annotation.authorities();
        String[] roles = annotation.roles();

        if (authorities.length > 0 && !(roles.length == 1 && "USER".equals(roles[0]))) {
            throw new IllegalStateException(
                    "@WithMockUser cannot have both 'roles' and 'authorities' set. "
                    + "Use roles() for ROLE_-prefixed authorities, or authorities() for raw strings.");
        }

        List<GrantedAuthority> result = new ArrayList<>();
        if (authorities.length > 0) {
            for (String authority : authorities) {
                result.add(new SimpleGrantedAuthority(authority));
            }
        }
        else {
            for (String role : roles) {
                if (role.startsWith("ROLE_")) {
                    throw new IllegalArgumentException(
                            "roles() should not start with ROLE_ -- "
                            + "use authorities() instead. Got: " + role);
                }
                result.add(new SimpleGrantedAuthority("ROLE_" + role));
            }
        }
        return result;
    }
```

**Why check `roles.length == 1 && "USER".equals(roles[0])`?** The `roles()` attribute
defaults to `{"USER"}`. If the developer sets `authorities()` but does not explicitly
change `roles()`, the roles array will still contain the default `"USER"`. The factory
detects this case and does not throw an error -- only when the developer explicitly sets
both does it reject the configuration.

**Why reject `roles()` values that start with `"ROLE_"`?** This is a safety check. Since
`roles()` automatically adds the `"ROLE_"` prefix, passing `"ROLE_ADMIN"` would result
in `"ROLE_ROLE_ADMIN"` -- almost certainly a mistake. The error message tells the developer
to use `authorities()` instead for pre-prefixed strings.

### Path 2: MockMvc post-processors

```
Developer calls .with(user("admin").roles("ADMIN"))
  |
  +---> SimpleSecurityMockMvcRequestPostProcessors.user("admin")
  |       Creates UserRequestPostProcessor with username = "admin"
  |       Default: password = "password", authorities = [ROLE_USER]
  |
  +---> .roles("ADMIN")
  |       Replaces authorities with [ROLE_ADMIN]
  |
  +---> MockMvc calls postProcessRequest(request)
          |
          +---> Creates User principal: new User("admin", "password", [ROLE_ADMIN])
          +---> Creates authenticated token: UsernamePasswordAuthenticationToken.authenticated(...)
          +---> Creates empty SecurityContext
          +---> Sets authentication on context
          +---> SecurityContextHolder.setContext(context)
          +---> Returns the (possibly modified) request
```

For `jwt()`, the flow is similar but creates a `JwtAuthenticationToken` instead:

```
Developer calls .with(jwt().jwt(b -> b.subject("admin")).authorities(...))
  |
  +---> SimpleSecurityMockMvcRequestPostProcessors.jwt()
  |       Creates JwtRequestPostProcessor
  |       Defaults: tokenValue = "mock-jwt-token", alg = "none", sub = "user", scope = "read"
  |
  +---> .jwt(b -> b.subject("admin"))
  |       Stores the customizer lambda
  |
  +---> .authorities(...)
  |       Stores the authority list
  |
  +---> MockMvc calls postProcessRequest(request)
          |
          +---> Builds Jwt: Jwt.withTokenValue("mock-jwt-token").header("alg", "none")
          |       .subject("user").claim("scope", "read") + customizer applied
          +---> Uses provided authorities (or defaults to [SCOPE_read])
          +---> Creates JwtAuthenticationToken(jwt, authorities)
          +---> SecurityContextHolder.setContext(context with authentication)
          +---> Returns the request
```

For `csrf()`, the flow modifies the request rather than the security context:

```
Developer calls .with(csrf())
  |
  +---> SimpleSecurityMockMvcRequestPostProcessors.csrf()
  |       Creates CsrfRequestPostProcessor
  |       Defaults: useHeader = false, useInvalidToken = false
  |
  +---> MockMvc calls postProcessRequest(request)
          |
          +---> Generates tokenValue: UUID.randomUUID().toString()
          +---> Creates DefaultCsrfToken("X-CSRF-TOKEN", "_csrf", tokenValue)
          +---> Sets as request attribute: CsrfToken.class.getName() -> csrfToken
          +---> Sets as request attribute: "_csrf" -> csrfToken
          +---> Sets token value as request parameter: _csrf = tokenValue
          |     (or as header X-CSRF-TOKEN = tokenValue if asHeader() was called)
          +---> Returns the modified request
```

**Why set the token as a request attribute?** The `SimpleCsrfFilter` (from Feature 9)
loads the "expected" CSRF token from the request's `CsrfToken.class.getName()` attribute
(or from a repository). By setting this attribute, the post-processor tells the CSRF filter
what the expected token is. The filter then compares this expected token against the actual
token from the parameter or header -- and they match because the post-processor set both.

**Why does `user()` set `SecurityContextHolder` directly instead of modifying the request?**
The security filters check `SecurityContextHolder` to determine if the request is already
authenticated. By setting the context before the request enters the filter chain, the
authentication filters see a pre-authenticated request and skip their normal processing.
This is the same mechanism that `@WithMockUser` uses -- both set the `SecurityContextHolder`,
just at different points in the lifecycle.

### 15c. The post-processor builders

Each post-processor uses the builder pattern to allow fluent configuration:

**UserRequestPostProcessor** -- builds a username/password authentication:

```java
public static final class UserRequestPostProcessor implements RequestPostProcessor {
    private final String username;
    private String password = "password";
    private List<GrantedAuthority> authorities = new ArrayList<>(
            List.of(new SimpleGrantedAuthority("ROLE_USER")));

    public UserRequestPostProcessor roles(String... roles) { ... }
    public UserRequestPostProcessor authorities(GrantedAuthority... authorities) { ... }
    public UserRequestPostProcessor password(String password) { ... }

    @Override
    public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
        User principal = new User(this.username, this.password, this.authorities);
        Authentication authentication = UsernamePasswordAuthenticationToken.authenticated(
                principal, this.password, this.authorities);

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);

        return request;
    }
}
```

**JwtRequestPostProcessor** -- builds a JWT authentication with configurable claims:

```java
public static final class JwtRequestPostProcessor implements RequestPostProcessor {
    private Jwt jwt;
    private Consumer<Jwt.Builder> jwtCustomizer = builder -> {};
    private List<GrantedAuthority> authorities;

    @Override
    public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
        Jwt jwtToken = this.jwt;
        if (jwtToken == null) {
            Jwt.Builder builder = Jwt.withTokenValue("mock-jwt-token")
                    .header("alg", "none")
                    .subject("user")
                    .claim("scope", "read");
            this.jwtCustomizer.accept(builder);
            jwtToken = builder.build();
        }

        Collection<? extends GrantedAuthority> grantedAuthorities = this.authorities;
        if (grantedAuthorities == null) {
            grantedAuthorities = List.of(new SimpleGrantedAuthority("SCOPE_read"));
        }

        JwtAuthenticationToken authentication = new JwtAuthenticationToken(
                jwtToken, grantedAuthorities);

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);

        return request;
    }
}
```

**Why does `JwtRequestPostProcessor` use `Consumer<Jwt.Builder>` instead of requiring a
pre-built Jwt?** The consumer pattern lets you customize only the claims you care about
while keeping sensible defaults. If you only need to change the subject, you write
`.jwt(b -> b.subject("admin"))` instead of building an entire `Jwt` from scratch. The
defaults (`alg: "none"`, `sub: "user"`, `scope: "read"`) cover the common case where
you just need a valid-looking JWT with some authorities.

### Path 3: MockMvc configurer

```
Developer calls MockMvcBuilders.standaloneSetup(controller)
        .apply(springSecurity(filterChainProxy))
        .build()
  |
  +---> SimpleSecurityMockMvcConfigurer.springSecurity(filterChainProxy)
  |       Creates a SimpleSecurityMockMvcConfigurer wrapping the security filter
  |
  +---> MockMvcBuilders calls afterConfigurerAdded(builder)
  |       builder.addFilters(this.securityFilter)
  |       -> the security filter chain is now applied to every test request
  |
  +---> MockMvcBuilders calls beforeMockMvcCreated(builder, context)
          Returns a default RequestPostProcessor: request -> request (identity/no-op)
```

The configurer does two things:
1. **Adds the security filter** to MockMvc's filter chain via `afterConfigurerAdded()`
2. **Returns a default post-processor** via `beforeMockMvcCreated()` that is a no-op

**Why is the default post-processor a no-op?** In the real framework, this default
post-processor propagates any `SecurityContext` that was set up by `@WithMockUser` into
the MockMvc request pipeline. In our simplified version, `SecurityContextHolder` is a
static thread-local, so the extension's `beforeEach()` already populated it before
MockMvc executes. The no-op post-processor is a placeholder that maintains the same
API contract as the real framework.

### Depth layers table

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Annotation | `@WithMockUser` | `@WithMockUser` | Creates mock Authentication |
| Annotation | `@WithSecurityContext` | `@WithSecurityContext` | Meta-annotation for custom contexts |
| Factory | `SimpleWithMockUserSecurityContextFactory` | `WithMockUserSecurityContextFactory` | Builds SecurityContext from annotation |
| Extension | `WithSecurityContextExtension` | `WithSecurityContextTestExecutionListener` | JUnit lifecycle hook |
| MockMvc | `SimpleSecurityMockMvcRequestPostProcessors` | `SecurityMockMvcRequestPostProcessors` | csrf(), user(), jwt() etc. |
| Setup | `SimpleSecurityMockMvcConfigurer` | `SecurityMockMvcConfigurers` | Applies security to MockMvc |

---

## Try It Yourself

**Challenge 1**: Create a custom `@WithMockAdmin` annotation. Define the annotation,
meta-annotate it with `@WithSecurityContext(factory = WithMockAdminFactory.class)` and
`@ExtendWith(WithSecurityContextExtension.class)`, then implement the factory to always
create a user named `"admin"` with `ROLE_ADMIN`. Write a test that uses `@WithMockAdmin`
and verifies the authentication. How does the extension find your factory without being
modified?

**Challenge 2**: Add a `httpBasic()` post-processor to
`SimpleSecurityMockMvcRequestPostProcessors`. It should accept a username and password,
Base64-encode them, and set the `Authorization: Basic ...` header on the request. Unlike
`user()`, it should *not* set `SecurityContextHolder` directly -- instead, the request
should flow through the Basic authentication filter. Write a test that verifies the header
is set correctly.

**Challenge 3**: Extend the `CsrfRequestPostProcessor` to support a `useTokenValue(String)`
method that lets the developer specify an exact token value. This is useful for testing
CSRF token replay attacks. Write a test where you use one token value on the request
attribute and a different one on the parameter, and verify that the CSRF filter rejects it.

---

## Why This Works

**Annotation-based testing as a composable system**: The `@WithMockUser` annotation does
not contain any SecurityContext-building logic. It declares *what* the test wants (a user
named "admin" with ROLE_ADMIN). The `@WithSecurityContext` meta-annotation declares *how*
to satisfy that request (use this factory class). The `WithSecurityContextExtension`
provides the *when* (before each test method). This separation means you can create
entirely new test annotations without modifying the extension infrastructure.

**Post-processors as request-level hooks**: MockMvc's `RequestPostProcessor` interface
is the right extension point for per-request security setup. Unlike the annotation-based
approach (which applies to the entire test method), post-processors apply to individual
requests. This means a single test method can send multiple requests with different
security contexts:

```java
@Test
void differentUsersInSameTest() {
    mockMvc.perform(get("/api/data").with(user("alice").roles("USER")))
           .andExpect(status().isOk());
    mockMvc.perform(get("/api/admin").with(user("bob").roles("ADMIN")))
           .andExpect(status().isOk());
}
```

**Builder pattern for readable test setup**: The `user("admin").roles("ADMIN").password("secret")`
and `jwt().jwt(b -> b.subject("admin")).authorities(...)` fluent APIs make test setup
read like a specification. Each builder method returns `this`, so they chain naturally.
The defaults are sensible (password is `"password"`, role is `ROLE_USER`, JWT subject is
`"user"`), so most tests only need to customize what is relevant to the test case.

**CSRF post-processor works by controlling both sides**: The `csrf()` post-processor
generates a token and places it in two locations: as a request attribute (the "expected"
token) and as a parameter/header (the "actual" token). The CSRF filter compares these
two values. Since both come from the same UUID, they always match. This is conceptually
the same as putting the same key in both the lock and the keyhole -- the test controls
both sides of the validation.

**The configurer bridges annotations and MockMvc**: Without the configurer, `@WithMockUser`
and MockMvc would be disconnected. The configurer adds the security filter chain to MockMvc
and provides a default post-processor that ensures the `SecurityContext` (populated by the
annotation extension) is available when the request enters the filter chain. This bridging
is what makes the following pattern work:

```java
@Test
@WithMockUser(username = "admin", roles = "ADMIN")
void withMockUserAndMockMvc() {
    mockMvc.perform(get("/api/admin"))
           .andExpect(status().isOk());
}
```

---

## What We Enhanced

| What Changed | Before (Feature 14) | After (Feature 15) | Why |
|---|---|---|---|
| New module | None | `simple-security-test` with `build.gradle` | Dedicated test support module mirroring `spring-security-test` |
| Annotation-based testing | Not available | `@WithMockUser`, `@WithSecurityContext`, `WithSecurityContextFactory` | Declarative security context setup for unit tests |
| JUnit 5 integration | Not available | `WithSecurityContextExtension` (BeforeEach/AfterEach) | Lifecycle hooks to populate and clear SecurityContextHolder |
| MockMvc user simulation | Not available | `user()`, `authentication()` post-processors | Imperative security context setup for MockMvc requests |
| MockMvc CSRF support | Not available | `csrf()` post-processor with `asHeader()` and `useInvalidToken()` | Adds valid/invalid CSRF tokens to test requests |
| MockMvc JWT support | Not available | `jwt()` post-processor with builder pattern | Creates JwtAuthenticationToken for resource server testing |
| MockMvc filter integration | Not available | `SimpleSecurityMockMvcConfigurer.springSecurity(filter)` | Applies security filters to MockMvc test pipeline |

This feature builds on types from four earlier features:

- **Feature 2** (SecurityContext): `SecurityContext`, `SecurityContextHolder`, `Authentication`, `UsernamePasswordAuthenticationToken`, `UserDetails`, `User`, `GrantedAuthority`, `SimpleGrantedAuthority` -- the core types that every test annotation and post-processor creates and populates
- **Feature 4** (SecurityFilterChain): `SecurityFilterChain`, `FilterChainProxy` -- the filter chain that the MockMvc configurer applies to test requests
- **Feature 9** (CSRF Protection): `CsrfToken`, `DefaultCsrfToken` -- the token types that the `csrf()` post-processor creates and places on the request
- **Feature 13** (OAuth2 Resource Server): `Jwt`, `JwtAuthenticationToken` -- the JWT types that the `jwt()` post-processor creates for resource server testing

This is the final feature (Feature 15) in the simplified Spring Security reimplementation. The test module
is intentionally the last feature because it tests all the security infrastructure built
in Features 1-14. By building testing support last, we ensure that the test module
exercises the actual types and patterns from every previous feature rather than testing
against stubs.

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|---|---|---|
| `@WithMockUser` | `@WithMockUser` | `test/src/main/java/org/springframework/security/test/context/support/WithMockUser.java:58` |
| `@WithSecurityContext` | `@WithSecurityContext` | `test/src/main/java/org/springframework/security/test/context/support/WithSecurityContext.java:57` |
| `WithSecurityContextFactory` | `WithSecurityContextFactory` | `test/src/main/java/org/springframework/security/test/context/support/WithSecurityContextFactory.java:35` |
| `SimpleWithMockUserSecurityContextFactory` | `WithMockUserSecurityContextFactory` | `test/src/main/java/org/springframework/security/test/context/support/WithMockUserSecurityContextFactory.java:42` |
| `WithSecurityContextExtension` | `WithSecurityContextTestExecutionListener` | `test/src/main/java/org/springframework/security/test/context/support/WithSecurityContextTestExecutionListener.java:60` |
| `SimpleSecurityMockMvcRequestPostProcessors` | `SecurityMockMvcRequestPostProcessors` | `test/src/main/java/org/springframework/security/test/web/servlet/request/SecurityMockMvcRequestPostProcessors.java:122` |
| `SimpleSecurityMockMvcConfigurer` | `SecurityMockMvcConfigurers` | `test/src/main/java/org/springframework/security/test/web/servlet/setup/SecurityMockMvcConfigurers.java:31` |

### Key differences from the real framework

**`WithSecurityContextTestExecutionListener` vs. `WithSecurityContextExtension`**: The real
framework uses Spring's `TestExecutionListener` -- a Spring-specific test lifecycle hook
that integrates with the Spring `TestContext` framework. This enables `@Autowired`
injection into factory classes, `setupBefore` timing control (before test method vs. before
test execution), and integration with `SecurityContextRepository` for web-based tests. Our
simplified version uses a plain JUnit 5 extension (`BeforeEachCallback` /
`AfterEachCallback`), which keeps it independent of the Spring test framework but loses
DI support and timing control.

**`SecurityMockMvcRequestPostProcessors`**: The real class is substantially larger (~2000
lines) and includes post-processors for: `httpBasic()`, `x509()`, `digest()`, `opaqueToken()`,
`oauth2Login()`, `oidcLogin()`, `oauth2Client()`, `securityContext()`, and more. Each
post-processor also integrates with `SecurityContextRepository` and
`TestSecurityContextHolder` for proper session/request-scoped context propagation. Our
version covers the three most fundamental post-processors: `csrf()`, `user()`, and `jwt()`.

**`SecurityMockMvcConfigurers`**: The real class resolves the security filter from the
`WebApplicationContext` via bean lookup -- you call `springSecurity()` with no arguments and
it finds `FilterChainProxy` automatically. Our version requires the filter to be passed
explicitly as a constructor argument. The real class also creates a more sophisticated
default post-processor that propagates `TestSecurityContextHolder` into the request and
integrates with `CsrfFilter` to ensure CSRF token consistency.

**`WithMockUserSecurityContextFactory`**: The real class supports
`SecurityContextHolderStrategy` injection (for custom thread-local strategies) and is
instantiated through the Spring application context, enabling `@Autowired` dependencies.
Our version uses `SecurityContextHolder` directly and reflective instantiation.

---

## Complete Code

### [NEW] `simple-security-test/build.gradle`

```groovy
// simple-security-test — @WithMockUser, MockMvc security support
// Mirrors: spring-security-test

dependencies {
    api project(':simple-security-core')
    api project(':simple-security-web')
    api project(':simple-security-oauth2-jose')
    api project(':simple-security-oauth2-resource-server')
    api platform('org.junit:junit-bom:5.11.4')
    api 'org.junit.jupiter:junit-jupiter-api'
    implementation 'org.springframework:spring-test:6.2.3'
    implementation 'org.springframework:spring-web:6.2.3'

    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    testImplementation 'jakarta.servlet:jakarta.servlet-api:6.1.0'
}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/context/support/WithSecurityContextFactory.java`

```java
package simple.security.test.context.support;

import java.lang.annotation.Annotation;

import simple.security.core.context.SecurityContext;

/**
 * A factory for creating a {@link SecurityContext} from a test annotation.
 *
 * <p>Each test annotation (like {@link WithMockUser}) is associated with a
 * {@code WithSecurityContextFactory} implementation via the {@link WithSecurityContext}
 * meta-annotation. The factory reads the annotation's attributes and builds a
 * pre-populated {@link SecurityContext} for the test to run with.
 *
 * <p>Users can create custom factories for custom test annotations:
 * <pre>
 * &#64;WithSecurityContext(factory = WithMockAdminFactory.class)
 * &#64;Retention(RUNTIME)
 * public &#64;interface WithMockAdmin { }
 *
 * public class WithMockAdminFactory implements WithSecurityContextFactory&lt;WithMockAdmin&gt; {
 *     public SecurityContext createSecurityContext(WithMockAdmin annotation) {
 *         // build a SecurityContext with admin authorities
 *     }
 * }
 * </pre>
 *
 * <p>Simplifications vs. real WithSecurityContextFactory:
 * <ul>
 *   <li>No Spring DI / {@code @Autowired} support — factory is instantiated via reflection</li>
 *   <li>No {@code createSecurityContext(Annotation, TestContext)} overload</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.context.support.WithSecurityContextFactory
 * Source:  test/src/main/java/org/springframework/security/test/context/support/WithSecurityContextFactory.java
 *
 * @param <A> the annotation type this factory handles
 */
public interface WithSecurityContextFactory<A extends Annotation> {

	/**
	 * Creates a {@link SecurityContext} from the given test annotation.
	 * @param annotation the annotation instance found on the test method or class
	 * @return a populated SecurityContext to use during the test
	 */
	SecurityContext createSecurityContext(A annotation);

}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/context/support/WithSecurityContext.java`

```java
package simple.security.test.context.support;

import java.lang.annotation.Annotation;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Meta-annotation that connects a test annotation to a {@link WithSecurityContextFactory}.
 *
 * <p>When a test method or class carries an annotation that is itself annotated with
 * {@code @WithSecurityContext}, the {@link WithSecurityContextExtension} JUnit 5
 * extension detects it, instantiates the referenced factory, and uses it to populate
 * the {@link simple.security.core.context.SecurityContextHolder} before the test runs.
 *
 * <p>This is the extensibility point of the test-security-context system. To create
 * a custom mock authentication annotation:
 * <ol>
 *   <li>Create your annotation and meta-annotate it with {@code @WithSecurityContext}</li>
 *   <li>Implement a {@link WithSecurityContextFactory} that reads your annotation</li>
 *   <li>Point the {@link #factory()} attribute to your factory class</li>
 * </ol>
 *
 * <p>Simplifications vs. real {@code @WithSecurityContext}:
 * <ul>
 *   <li>No {@code setupBefore} attribute — context is always set up before each test method</li>
 *   <li>Uses JUnit 5 {@code @ExtendWith} instead of Spring's {@code TestExecutionListener}</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.context.support.WithSecurityContext
 * Source:  test/src/main/java/org/springframework/security/test/context/support/WithSecurityContext.java
 */
@Target(ElementType.ANNOTATION_TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface WithSecurityContext {

	/**
	 * The {@link WithSecurityContextFactory} implementation that will create
	 * the {@link simple.security.core.context.SecurityContext} from the annotated annotation.
	 */
	Class<? extends WithSecurityContextFactory<? extends Annotation>> factory();

}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/context/support/WithMockUser.java`

```java
package simple.security.test.context.support;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.junit.jupiter.api.extension.ExtendWith;

/**
 * Annotate a test method or class to run as a mock user with the given username,
 * password, and roles/authorities.
 *
 * <h2>Basic usage</h2>
 * <pre>
 * &#64;Test
 * &#64;WithMockUser(username = "admin", roles = "ADMIN")
 * void adminCanAccessDashboard() {
 *     Authentication auth = SecurityContextHolder.getContext().getAuthentication();
 *     assertThat(auth.getName()).isEqualTo("admin");
 *     assertThat(auth.getAuthorities()).extracting("authority")
 *             .containsExactly("ROLE_ADMIN");
 * }
 * </pre>
 *
 * <h2>How it works</h2>
 * <ol>
 *   <li>The {@link WithSecurityContext} meta-annotation triggers the
 *       {@link WithSecurityContextExtension} JUnit 5 extension</li>
 *   <li>The extension finds this annotation and instantiates the
 *       {@link SimpleWithMockUserSecurityContextFactory}</li>
 *   <li>The factory creates a {@link simple.security.core.context.SecurityContext}
 *       with a {@link simple.security.authentication.UsernamePasswordAuthenticationToken}
 *       containing a {@link simple.security.core.userdetails.User} principal</li>
 *   <li>The context is set on {@link simple.security.core.context.SecurityContextHolder}
 *       before the test, and cleared after</li>
 * </ol>
 *
 * <h2>Roles vs. authorities</h2>
 * <p>{@link #roles()} auto-prefixes each value with {@code "ROLE_"} — use it for
 * role-based access control. {@link #authorities()} uses the strings verbatim — use
 * it for fine-grained authorities like {@code "ORDER_READ"}.
 * <strong>Do not set both</strong>.
 *
 * <p>Simplifications vs. real {@code @WithMockUser}:
 * <ul>
 *   <li>No {@code setupBefore} attribute</li>
 *   <li>JUnit 5 {@code @ExtendWith} instead of Spring's TestExecutionListener</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.context.support.WithMockUser
 * Source:  test/src/main/java/org/springframework/security/test/context/support/WithMockUser.java
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@WithSecurityContext(factory = SimpleWithMockUserSecurityContextFactory.class)
@ExtendWith(WithSecurityContextExtension.class)
public @interface WithMockUser {

	/**
	 * Convenience alias for {@link #username()}. If {@link #username()} is
	 * specified and non-empty, it takes precedence over this value.
	 */
	String value() default "user";

	/**
	 * The username to use. Defaults to empty, which falls back to {@link #value()}.
	 */
	String username() default "";

	/**
	 * The roles to grant, auto-prefixed with {@code "ROLE_"}.
	 * <p>Defaults to {@code {"USER"}} (i.e., the user will have {@code ROLE_USER}).
	 * <p>Cannot be used together with {@link #authorities()}.
	 */
	String[] roles() default {"USER"};

	/**
	 * The authorities to grant, used verbatim (no prefix added).
	 * <p>If specified, {@link #roles()} must remain at its default.
	 * <p>Defaults to empty (meaning {@link #roles()} will be used instead).
	 */
	String[] authorities() default {};

	/**
	 * The password for the mock user.
	 */
	String password() default "password";

}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/context/support/SimpleWithMockUserSecurityContextFactory.java`

```java
package simple.security.test.context.support;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.userdetails.User;

/**
 * Creates a {@link SecurityContext} from a {@link WithMockUser} annotation.
 *
 * <h2>Resolution logic</h2>
 * <ol>
 *   <li><strong>Username:</strong> uses {@link WithMockUser#username()} if non-empty,
 *       otherwise falls back to {@link WithMockUser#value()}</li>
 *   <li><strong>Authorities:</strong> if {@link WithMockUser#authorities()} is specified,
 *       those strings are used as-is. Otherwise, each role from {@link WithMockUser#roles()}
 *       is prefixed with {@code "ROLE_"}. Specifying both is an error.</li>
 *   <li><strong>Principal:</strong> a {@link User} is created with the resolved username,
 *       password, and authorities</li>
 *   <li><strong>Authentication:</strong> a {@link UsernamePasswordAuthenticationToken#authenticated
 *       authenticated} token wraps the principal</li>
 * </ol>
 *
 * <p>Simplifications vs. real WithMockUserSecurityContextFactory:
 * <ul>
 *   <li>No SecurityContextHolderStrategy injection — uses SecurityContextHolder directly</li>
 *   <li>No Spring DI support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.context.support.WithMockUserSecurityContextFactory
 * Source:  test/src/main/java/org/springframework/security/test/context/support/WithMockUserSecurityContextFactory.java
 */
public final class SimpleWithMockUserSecurityContextFactory
		implements WithSecurityContextFactory<WithMockUser> {

	@Override
	public SecurityContext createSecurityContext(WithMockUser annotation) {
		String username = annotation.username().isEmpty()
				? annotation.value()
				: annotation.username();

		List<GrantedAuthority> authorities = resolveAuthorities(annotation);

		User principal = new User(username, annotation.password(), authorities);

		var authentication = UsernamePasswordAuthenticationToken.authenticated(
				principal, annotation.password(), authorities);

		SecurityContext context = SecurityContextHolder.createEmptyContext();
		context.setAuthentication(authentication);
		return context;
	}

	private List<GrantedAuthority> resolveAuthorities(WithMockUser annotation) {
		String[] authorities = annotation.authorities();
		String[] roles = annotation.roles();

		if (authorities.length > 0 && !(roles.length == 1 && "USER".equals(roles[0]))) {
			throw new IllegalStateException(
					"@WithMockUser cannot have both 'roles' and 'authorities' set. "
					+ "Use roles() for ROLE_-prefixed authorities, or authorities() for raw strings.");
		}

		List<GrantedAuthority> result = new ArrayList<>();
		if (authorities.length > 0) {
			for (String authority : authorities) {
				result.add(new SimpleGrantedAuthority(authority));
			}
		}
		else {
			for (String role : roles) {
				if (role.startsWith("ROLE_")) {
					throw new IllegalArgumentException(
							"roles() should not start with ROLE_ — "
							+ "use authorities() instead. Got: " + role);
				}
				result.add(new SimpleGrantedAuthority("ROLE_" + role));
			}
		}
		return result;
	}

}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/context/support/WithSecurityContextExtension.java`

```java
package simple.security.test.context.support;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;

import org.junit.jupiter.api.extension.AfterEachCallback;
import org.junit.jupiter.api.extension.BeforeEachCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;

/**
 * JUnit 5 extension that processes {@link WithSecurityContext}-annotated test
 * annotations (like {@link WithMockUser}) to populate the
 * {@link SecurityContextHolder} before each test.
 *
 * <h2>How it works</h2>
 * <ol>
 *   <li><strong>Before each test:</strong> scans the test method and class for
 *       annotations that carry {@link WithSecurityContext}</li>
 *   <li>If found, instantiates the referenced {@link WithSecurityContextFactory}</li>
 *   <li>Calls {@link WithSecurityContextFactory#createSecurityContext(Annotation)}
 *       with the annotation instance</li>
 *   <li>Sets the returned context on {@link SecurityContextHolder}</li>
 *   <li><strong>After each test:</strong> clears the SecurityContextHolder to prevent
 *       leakage between tests</li>
 * </ol>
 *
 * <h2>Method vs. class annotations</h2>
 * Method-level annotations take precedence over class-level ones. This lets you
 * set a default user on the class and override it per test method.
 *
 * <p>Simplifications vs. real WithSecurityContextTestExecutionListener:
 * <ul>
 *   <li>JUnit 5 extension instead of Spring TestExecutionListener</li>
 *   <li>No setupBefore (TEST_METHOD vs TEST_EXECUTION) distinction</li>
 *   <li>No Spring DI for factory classes — instantiated via no-arg constructor</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.context.support.WithSecurityContextTestExecutionListener
 * Source:  test/src/main/java/org/springframework/security/test/context/support/WithSecurityContextTestExecutionListener.java
 */
public class WithSecurityContextExtension implements BeforeEachCallback, AfterEachCallback {

	@Override
	public void beforeEach(ExtensionContext context) throws Exception {
		// Method-level annotation takes precedence over class-level
		SecurityContext securityContext = findAndCreateSecurityContext(
				context.getRequiredTestMethod());
		if (securityContext == null) {
			securityContext = findAndCreateSecurityContext(
					context.getRequiredTestClass());
		}
		if (securityContext != null) {
			SecurityContextHolder.setContext(securityContext);
		}
	}

	@Override
	public void afterEach(ExtensionContext context) {
		SecurityContextHolder.clearContext();
	}

	/**
	 * Scans the given element for an annotation that carries {@link WithSecurityContext},
	 * then uses the referenced factory to create a SecurityContext.
	 */
	@SuppressWarnings({"unchecked", "rawtypes"})
	private SecurityContext findAndCreateSecurityContext(AnnotatedElement element) throws Exception {
		for (Annotation annotation : element.getAnnotations()) {
			WithSecurityContext withSecurityContext = annotation.annotationType()
					.getAnnotation(WithSecurityContext.class);
			if (withSecurityContext != null) {
				Class<? extends WithSecurityContextFactory> factoryClass =
						withSecurityContext.factory();
				WithSecurityContextFactory factory = factoryClass.getDeclaredConstructor()
						.newInstance();
				return factory.createSecurityContext(annotation);
			}
		}
		return null;
	}

}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/web/servlet/request/SimpleSecurityMockMvcRequestPostProcessors.java`

```java
package simple.security.test.web.servlet.request;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.List;
import java.util.UUID;
import java.util.function.Consumer;

import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.test.web.servlet.request.RequestPostProcessor;

import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.userdetails.User;
import simple.security.oauth2.jwt.Jwt;
import simple.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import simple.security.web.csrf.CsrfToken;
import simple.security.web.csrf.DefaultCsrfToken;

/**
 * Static factory methods that create {@link RequestPostProcessor} instances
 * for use with MockMvc's {@code .with(...)} method.
 *
 * <h2>Usage examples</h2>
 * <pre>
 * import static simple.security.test.web.servlet.request
 *     .SimpleSecurityMockMvcRequestPostProcessors.*;
 *
 * // Mock a user:
 * mockMvc.perform(get("/api/data").with(user("john").roles("USER")))
 *        .andExpect(status().isOk());
 *
 * // Mock a JWT:
 * mockMvc.perform(get("/api/data")
 *         .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_read"))))
 *        .andExpect(status().isOk());
 *
 * // Add a CSRF token:
 * mockMvc.perform(post("/api/orders")
 *         .with(csrf())
 *         .with(user("john").roles("USER")))
 *        .andExpect(status().isCreated());
 * </pre>
 *
 * <h2>How it works</h2>
 * <p>Each post-processor modifies the {@link MockHttpServletRequest} before it enters
 * the security filter chain:
 * <ul>
 *   <li>{@link #user(String)} / {@link #jwt()} / {@link #authentication(Authentication)}:
 *       set the {@link SecurityContext} on {@link SecurityContextHolder}, so the security
 *       filters see the request as already authenticated</li>
 *   <li>{@link #csrf()}: adds a valid CSRF token as both a request attribute (so
 *       {@code SimpleCsrfFilter} can load it) and a request parameter/header (so
 *       validation passes)</li>
 * </ul>
 *
 * <p>Simplifications vs. real SecurityMockMvcRequestPostProcessors:
 * <ul>
 *   <li>No SecurityContextRepository integration — sets SecurityContextHolder directly</li>
 *   <li>No digest(), x509(), httpBasic(), opaqueToken(), oauth2Login(), oidcLogin()</li>
 *   <li>No CsrfFilter.skipRequest() for JWT — simplified CSRF handling</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors
 * Source:  test/src/main/java/org/springframework/security/test/web/servlet/request/SecurityMockMvcRequestPostProcessors.java
 */
public final class SimpleSecurityMockMvcRequestPostProcessors {

	private SimpleSecurityMockMvcRequestPostProcessors() {
	}

	// ========== CSRF ==========

	/**
	 * Creates a post-processor that adds a valid CSRF token to the request.
	 * By default, the token is added as a request parameter. Call
	 * {@link CsrfRequestPostProcessor#asHeader()} to send it as a header instead.
	 */
	public static CsrfRequestPostProcessor csrf() {
		return new CsrfRequestPostProcessor();
	}

	// ========== User ==========

	/**
	 * Creates a post-processor that authenticates the request as the given user.
	 * <p>Defaults: password {@code "password"}, role {@code ROLE_USER}.
	 * <p>Use the builder methods to customize: {@code user("admin").roles("ADMIN")}
	 */
	public static UserRequestPostProcessor user(String username) {
		return new UserRequestPostProcessor(username);
	}

	/**
	 * Creates a post-processor that authenticates the request with the given
	 * {@link simple.security.core.userdetails.UserDetails}.
	 */
	public static RequestPostProcessor user(simple.security.core.userdetails.UserDetails userDetails) {
		return new AuthenticationRequestPostProcessor(
				UsernamePasswordAuthenticationToken.authenticated(
						userDetails, userDetails.getPassword(),
						userDetails.getAuthorities()));
	}

	// ========== Authentication ==========

	/**
	 * Creates a post-processor that sets the given {@link Authentication} on
	 * the {@link SecurityContextHolder}.
	 */
	public static RequestPostProcessor authentication(Authentication authentication) {
		return new AuthenticationRequestPostProcessor(authentication);
	}

	// ========== JWT ==========

	/**
	 * Creates a post-processor that authenticates the request with a mock JWT.
	 * <p>Defaults: subject {@code "user"}, scope {@code "read"}, algorithm {@code "none"}.
	 * <p>Use the builder methods to customize claims and authorities.
	 */
	public static JwtRequestPostProcessor jwt() {
		return new JwtRequestPostProcessor();
	}

	// ========== Inner classes ==========

	/**
	 * Post-processor that adds a valid CSRF token to a MockHttpServletRequest.
	 */
	public static final class CsrfRequestPostProcessor implements RequestPostProcessor {

		private static final String DEFAULT_HEADER_NAME = "X-CSRF-TOKEN";
		private static final String DEFAULT_PARAMETER_NAME = "_csrf";

		private boolean useHeader = false;
		private boolean useInvalidToken = false;

		private CsrfRequestPostProcessor() {
		}

		/**
		 * Send the CSRF token as a request header instead of a parameter.
		 */
		public CsrfRequestPostProcessor asHeader() {
			this.useHeader = true;
			return this;
		}

		/**
		 * Use an intentionally invalid token value — useful for testing that
		 * CSRF validation correctly rejects bad tokens.
		 */
		public CsrfRequestPostProcessor useInvalidToken() {
			this.useInvalidToken = true;
			return this;
		}

		@Override
		public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
			String tokenValue = this.useInvalidToken
					? "invalid-token"
					: UUID.randomUUID().toString();

			CsrfToken csrfToken = new DefaultCsrfToken(
					DEFAULT_HEADER_NAME, DEFAULT_PARAMETER_NAME, tokenValue);

			// Set as request attribute so SimpleCsrfFilter can load it
			request.setAttribute(CsrfToken.class.getName(), csrfToken);
			request.setAttribute(DEFAULT_PARAMETER_NAME, csrfToken);

			// Set the actual token value as parameter or header so validation passes
			if (this.useHeader) {
				request.addHeader(DEFAULT_HEADER_NAME, tokenValue);
			}
			else {
				request.addParameter(DEFAULT_PARAMETER_NAME, tokenValue);
			}

			return request;
		}

	}

	/**
	 * Post-processor that authenticates a request as a user with configurable
	 * username, password, and roles/authorities.
	 */
	public static final class UserRequestPostProcessor implements RequestPostProcessor {

		private final String username;
		private String password = "password";
		private List<GrantedAuthority> authorities = new ArrayList<>(
				List.of(new SimpleGrantedAuthority("ROLE_USER")));

		private UserRequestPostProcessor(String username) {
			if (username == null || username.isEmpty()) {
				throw new IllegalArgumentException("username cannot be null or empty");
			}
			this.username = username;
		}

		/**
		 * Sets the roles, auto-prefixed with {@code "ROLE_"}.
		 * Replaces any previously set authorities.
		 */
		public UserRequestPostProcessor roles(String... roles) {
			this.authorities = new ArrayList<>();
			for (String role : roles) {
				if (role.startsWith("ROLE_")) {
					throw new IllegalArgumentException(
							"roles() should not start with ROLE_ — "
							+ "use authorities() instead. Got: " + role);
				}
				this.authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
			}
			return this;
		}

		/**
		 * Sets the authorities verbatim (no prefix).
		 * Replaces any previously set authorities.
		 */
		public UserRequestPostProcessor authorities(GrantedAuthority... authorities) {
			this.authorities = new ArrayList<>(Arrays.asList(authorities));
			return this;
		}

		/**
		 * Sets the authorities verbatim (no prefix).
		 * Replaces any previously set authorities.
		 */
		public UserRequestPostProcessor authorities(
				Collection<? extends GrantedAuthority> authorities) {
			this.authorities = new ArrayList<>(authorities);
			return this;
		}

		/**
		 * Sets the password for the mock user.
		 */
		public UserRequestPostProcessor password(String password) {
			this.password = password;
			return this;
		}

		@Override
		public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
			User principal = new User(this.username, this.password, this.authorities);
			Authentication authentication = UsernamePasswordAuthenticationToken.authenticated(
					principal, this.password, this.authorities);

			SecurityContext context = SecurityContextHolder.createEmptyContext();
			context.setAuthentication(authentication);
			SecurityContextHolder.setContext(context);

			return request;
		}

	}

	/**
	 * Post-processor that authenticates a request with a mock JWT.
	 */
	public static final class JwtRequestPostProcessor implements RequestPostProcessor {

		private Jwt jwt;
		private Consumer<Jwt.Builder> jwtCustomizer = builder -> {};
		private List<GrantedAuthority> authorities;

		private JwtRequestPostProcessor() {
		}

		/**
		 * Customize the JWT claims and headers.
		 * <pre>
		 * .with(jwt().jwt(jwt -> jwt
		 *     .subject("admin")
		 *     .claim("scope", "read write")))
		 * </pre>
		 */
		public JwtRequestPostProcessor jwt(Consumer<Jwt.Builder> jwtCustomizer) {
			this.jwtCustomizer = jwtCustomizer;
			return this;
		}

		/**
		 * Use a pre-built {@link Jwt} directly.
		 */
		public JwtRequestPostProcessor jwt(Jwt jwt) {
			this.jwt = jwt;
			return this;
		}

		/**
		 * Sets the authorities for the JWT authentication.
		 * Overrides any authorities that would be derived from claims.
		 */
		public JwtRequestPostProcessor authorities(GrantedAuthority... authorities) {
			this.authorities = Arrays.asList(authorities);
			return this;
		}

		/**
		 * Sets the authorities for the JWT authentication.
		 * Overrides any authorities that would be derived from claims.
		 */
		public JwtRequestPostProcessor authorities(
				Collection<? extends GrantedAuthority> authorities) {
			this.authorities = new ArrayList<>(authorities);
			return this;
		}

		@Override
		public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
			Jwt jwtToken = this.jwt;
			if (jwtToken == null) {
				Jwt.Builder builder = Jwt.withTokenValue("mock-jwt-token")
						.header("alg", "none")
						.subject("user")
						.claim("scope", "read");
				this.jwtCustomizer.accept(builder);
				jwtToken = builder.build();
			}

			Collection<? extends GrantedAuthority> grantedAuthorities = this.authorities;
			if (grantedAuthorities == null) {
				grantedAuthorities = List.of(new SimpleGrantedAuthority("SCOPE_read"));
			}

			JwtAuthenticationToken authentication = new JwtAuthenticationToken(
					jwtToken, grantedAuthorities);

			SecurityContext context = SecurityContextHolder.createEmptyContext();
			context.setAuthentication(authentication);
			SecurityContextHolder.setContext(context);

			return request;
		}

	}

	/**
	 * Post-processor that sets an arbitrary Authentication on SecurityContextHolder.
	 */
	private static final class AuthenticationRequestPostProcessor
			implements RequestPostProcessor {

		private final Authentication authentication;

		AuthenticationRequestPostProcessor(Authentication authentication) {
			this.authentication = authentication;
		}

		@Override
		public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
			SecurityContext context = SecurityContextHolder.createEmptyContext();
			context.setAuthentication(this.authentication);
			SecurityContextHolder.setContext(context);
			return request;
		}

	}

}
```

### [NEW] `simple-security-test/src/main/java/simple/security/test/web/servlet/setup/SimpleSecurityMockMvcConfigurer.java`

```java
package simple.security.test.web.servlet.setup;

import jakarta.servlet.Filter;

import org.springframework.test.web.servlet.request.RequestPostProcessor;
import org.springframework.test.web.servlet.setup.ConfigurableMockMvcBuilder;
import org.springframework.test.web.servlet.setup.MockMvcConfigurerAdapter;
import org.springframework.web.context.WebApplicationContext;

import simple.security.core.context.SecurityContext;
import simple.security.core.context.SecurityContextHolder;

/**
 * Applies Spring Security's filter chain to a MockMvc instance.
 *
 * <h2>Usage</h2>
 * <pre>
 * import static simple.security.test.web.servlet.setup
 *     .SimpleSecurityMockMvcConfigurer.springSecurity;
 *
 * MockMvc mockMvc = MockMvcBuilders
 *         .standaloneSetup(controller)
 *         .apply(springSecurity(filterChainProxy))
 *         .build();
 * </pre>
 *
 * <h2>What it does</h2>
 * <ol>
 *   <li>Adds the security filter (typically a {@code SimpleFilterChainProxy}) to MockMvc's
 *       filter chain so every test request passes through security filters</li>
 *   <li>Returns a default {@link RequestPostProcessor} that applies the current
 *       {@link SecurityContext} from {@link SecurityContextHolder} to each request —
 *       this bridges {@code @WithMockUser} (which sets up SecurityContextHolder) into
 *       the MockMvc request pipeline</li>
 * </ol>
 *
 * <p>Simplifications vs. real SecurityMockMvcConfigurers:
 * <ul>
 *   <li>Requires the filter chain as a constructor argument — no WebApplicationContext lookup</li>
 *   <li>No DelegateFilter lazy resolution — filter is known at construction time</li>
 *   <li>No TestSecurityContextHolder — uses SecurityContextHolder directly</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers
 * Source:  test/src/main/java/org/springframework/security/test/web/servlet/setup/SecurityMockMvcConfigurers.java
 */
public final class SimpleSecurityMockMvcConfigurer extends MockMvcConfigurerAdapter {

	private final Filter securityFilter;

	private SimpleSecurityMockMvcConfigurer(Filter securityFilter) {
		if (securityFilter == null) {
			throw new IllegalArgumentException("securityFilter cannot be null");
		}
		this.securityFilter = securityFilter;
	}

	/**
	 * Creates a configurer that applies the given security filter to MockMvc.
	 * @param securityFilter typically a {@link simple.security.web.SimpleFilterChainProxy}
	 * @return the configurer to pass to {@code MockMvcBuilders...apply(...)}
	 */
	public static SimpleSecurityMockMvcConfigurer springSecurity(Filter securityFilter) {
		return new SimpleSecurityMockMvcConfigurer(securityFilter);
	}

	/**
	 * Adds the security filter chain to the MockMvc builder.
	 */
	@Override
	public void afterConfigurerAdded(ConfigurableMockMvcBuilder<?> builder) {
		builder.addFilters(this.securityFilter);
	}

	/**
	 * Returns a default {@link RequestPostProcessor} that is applied to every request.
	 * This ensures that any {@link SecurityContext} set up by {@code @WithMockUser}
	 * is available to the security filter chain.
	 *
	 * <p>If a per-request post-processor (like {@code .with(user("admin"))}) has already
	 * set a SecurityContext, this default post-processor is a no-op because the
	 * SecurityContextHolder will already be populated.
	 */
	@Override
	public RequestPostProcessor beforeMockMvcCreated(
			ConfigurableMockMvcBuilder<?> builder, WebApplicationContext context) {
		return request -> request;
	}

}
```

### [NEW] `simple-security-test/src/test/java/simple/security/test/context/support/WithMockUserExtensionTest.java`

```java
package simple.security.test.context.support;

import org.junit.jupiter.api.Test;

import simple.security.core.Authentication;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.userdetails.UserDetails;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for {@link WithMockUser}.
 *
 * <p>These tests demonstrate how a client would use {@code @WithMockUser} to simulate
 * authenticated users in their tests. The annotation sets up the SecurityContext before
 * each test method and clears it after, so tests are isolated.
 */
class WithMockUserExtensionTest {

	// ========== Default behavior ==========

	@Test
	@WithMockUser
	void shouldSetUpDefaultUser() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth).isNotNull();
		assertThat(auth.isAuthenticated()).isTrue();
		assertThat(auth.getName()).isEqualTo("user");
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("ROLE_USER");
	}

	@Test
	@WithMockUser
	void shouldHaveDefaultPassword() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getCredentials()).isEqualTo("password");
	}

	@Test
	@WithMockUser
	void shouldHaveUserDetailsPrincipal() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getPrincipal()).isInstanceOf(UserDetails.class);
		UserDetails user = (UserDetails) auth.getPrincipal();
		assertThat(user.getUsername()).isEqualTo("user");
		assertThat(user.getPassword()).isEqualTo("password");
	}

	// ========== Custom username ==========

	@Test
	@WithMockUser("admin")
	void shouldUseValueAsUsername() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("admin");
	}

	@Test
	@WithMockUser(username = "john")
	void shouldUseExplicitUsername() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("john");
	}

	@Test
	@WithMockUser(value = "ignored", username = "preferred")
	void shouldPreferUsernameOverValue() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("preferred");
	}

	// ========== Custom roles ==========

	@Test
	@WithMockUser(roles = "ADMIN")
	void shouldPrefixRolesWithROLE() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("ROLE_ADMIN");
	}

	@Test
	@WithMockUser(roles = {"ADMIN", "MANAGER"})
	void shouldSupportMultipleRoles() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactlyInAnyOrder("ROLE_ADMIN", "ROLE_MANAGER");
	}

	// ========== Custom authorities ==========

	@Test
	@WithMockUser(authorities = "ORDER_READ")
	void shouldUseAuthoritiesVerbatim() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("ORDER_READ");
	}

	@Test
	@WithMockUser(authorities = {"ORDER_READ", "ORDER_WRITE"})
	void shouldSupportMultipleAuthorities() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactlyInAnyOrder("ORDER_READ", "ORDER_WRITE");
	}

	// ========== Custom password ==========

	@Test
	@WithMockUser(password = "secret123")
	void shouldUseCustomPassword() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getCredentials()).isEqualTo("secret123");
		UserDetails user = (UserDetails) auth.getPrincipal();
		assertThat(user.getPassword()).isEqualTo("secret123");
	}

	// ========== Full customization ==========

	@Test
	@WithMockUser(username = "admin", authorities = {"READ", "WRITE"}, password = "s3cret")
	void shouldSupportFullCustomization() {
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("admin");
		assertThat(auth.getCredentials()).isEqualTo("s3cret");
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactlyInAnyOrder("READ", "WRITE");
	}

	// ========== Context isolation ==========

	@Test
	void shouldNotHaveAuthenticationWithoutAnnotation() {
		// No @WithMockUser — SecurityContext should be empty
		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth).isNull();
	}

}
```

### [NEW] `simple-security-test/src/test/java/simple/security/test/web/servlet/request/SimpleSecurityMockMvcRequestPostProcessorsTest.java`

```java
package simple.security.test.web.servlet.request;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;

import simple.security.core.Authentication;
import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContextHolder;
import simple.security.core.userdetails.User;
import simple.security.core.userdetails.UserDetails;
import simple.security.oauth2.jwt.Jwt;
import simple.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import simple.security.web.csrf.CsrfToken;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static simple.security.test.web.servlet.request
		.SimpleSecurityMockMvcRequestPostProcessors.*;

/**
 * Client-perspective tests for {@link SimpleSecurityMockMvcRequestPostProcessors}.
 *
 * <p>These tests demonstrate how a client would use the MockMvc post-processors
 * to simulate authenticated users, CSRF tokens, and JWT-protected requests.
 */
class SimpleSecurityMockMvcRequestPostProcessorsTest {

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	// ========== csrf() ==========

	@Test
	void csrfShouldAddTokenAsRequestAttribute() {
		MockHttpServletRequest request = csrf()
				.postProcessRequest(new MockHttpServletRequest());

		CsrfToken token = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
		assertThat(token).isNotNull();
		assertThat(token.getToken()).isNotEmpty();
		assertThat(token.getHeaderName()).isEqualTo("X-CSRF-TOKEN");
		assertThat(token.getParameterName()).isEqualTo("_csrf");
	}

	@Test
	void csrfShouldAddTokenAsParameter() {
		MockHttpServletRequest request = csrf()
				.postProcessRequest(new MockHttpServletRequest());

		CsrfToken token = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
		String paramValue = request.getParameter("_csrf");
		assertThat(paramValue).isEqualTo(token.getToken());
	}

	@Test
	void csrfAsHeaderShouldAddTokenAsHeader() {
		MockHttpServletRequest request = csrf().asHeader()
				.postProcessRequest(new MockHttpServletRequest());

		CsrfToken token = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
		String headerValue = request.getHeader("X-CSRF-TOKEN");
		assertThat(headerValue).isEqualTo(token.getToken());
		// Should NOT be set as parameter when using asHeader()
		assertThat(request.getParameter("_csrf")).isNull();
	}

	@Test
	void csrfUseInvalidTokenShouldSetInvalidValue() {
		MockHttpServletRequest request = csrf().useInvalidToken()
				.postProcessRequest(new MockHttpServletRequest());

		CsrfToken token = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
		assertThat(token.getToken()).isEqualTo("invalid-token");
	}

	// ========== user(String) ==========

	@Test
	void userShouldSetSecurityContext() {
		user("john").postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth).isNotNull();
		assertThat(auth.isAuthenticated()).isTrue();
		assertThat(auth.getName()).isEqualTo("john");
	}

	@Test
	void userShouldHaveDefaultRoleUser() {
		user("john").postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("ROLE_USER");
	}

	@Test
	void userWithCustomRoles() {
		user("admin").roles("ADMIN", "MANAGER")
				.postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("admin");
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactlyInAnyOrder("ROLE_ADMIN", "ROLE_MANAGER");
	}

	@Test
	void userWithCustomAuthorities() {
		user("john").authorities(
						new SimpleGrantedAuthority("ORDER_READ"),
						new SimpleGrantedAuthority("ORDER_WRITE"))
				.postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactlyInAnyOrder("ORDER_READ", "ORDER_WRITE");
	}

	@Test
	void userWithCustomPassword() {
		user("john").password("secret123")
				.postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getCredentials()).isEqualTo("secret123");
	}

	@Test
	void userShouldHaveUserDetailsPrincipal() {
		user("john").roles("USER")
				.postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getPrincipal()).isInstanceOf(UserDetails.class);
		UserDetails principal = (UserDetails) auth.getPrincipal();
		assertThat(principal.getUsername()).isEqualTo("john");
	}

	@Test
	void userRolesShouldRejectROLE_Prefix() {
		assertThatThrownBy(() -> user("john").roles("ROLE_ADMIN"))
				.isInstanceOf(IllegalArgumentException.class)
				.hasMessageContaining("ROLE_");
	}

	// ========== user(UserDetails) ==========

	@Test
	void userDetailsShouldSetSecurityContext() {
		UserDetails userDetails = User.withUsername("admin")
				.password("secret")
				.roles("ADMIN")
				.build();

		user(userDetails).postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("admin");
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("ROLE_ADMIN");
	}

	// ========== authentication() ==========

	@Test
	void authenticationShouldSetArbitraryAuth() {
		var token = simple.security.authentication.UsernamePasswordAuthenticationToken
				.authenticated("custom-principal", "creds",
						List.of(new SimpleGrantedAuthority("CUSTOM")));

		authentication(token).postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth).isSameAs(token);
		assertThat(auth.getPrincipal()).isEqualTo("custom-principal");
	}

	// ========== jwt() ==========

	@Test
	void jwtShouldSetJwtAuthentication() {
		jwt().postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth).isInstanceOf(JwtAuthenticationToken.class);
		assertThat(auth.isAuthenticated()).isTrue();
	}

	@Test
	void jwtShouldHaveDefaultSubjectAndScope() {
		jwt().postProcessRequest(new MockHttpServletRequest());

		JwtAuthenticationToken auth = (JwtAuthenticationToken) SecurityContextHolder
				.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("user");
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("SCOPE_read");
	}

	@Test
	void jwtWithCustomClaims() {
		jwt().jwt(builder -> builder
						.subject("admin")
						.claim("scope", "read write"))
				.postProcessRequest(new MockHttpServletRequest());

		JwtAuthenticationToken auth = (JwtAuthenticationToken) SecurityContextHolder
				.getContext().getAuthentication();
		assertThat(auth.getName()).isEqualTo("admin");
		Jwt token = auth.getToken();
		assertThat(token.getSubject()).isEqualTo("admin");
		assertThat(token.<String>getClaim("scope")).isEqualTo("read write");
	}

	@Test
	void jwtWithCustomAuthorities() {
		jwt().authorities(
						new SimpleGrantedAuthority("SCOPE_read"),
						new SimpleGrantedAuthority("SCOPE_write"))
				.postProcessRequest(new MockHttpServletRequest());

		Authentication auth = SecurityContextHolder.getContext().getAuthentication();
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactlyInAnyOrder("SCOPE_read", "SCOPE_write");
	}

	@Test
	void jwtWithPreBuiltJwt() {
		Jwt prebuilt = Jwt.withTokenValue("my-custom-token")
				.header("alg", "RS256")
				.subject("service-account")
				.claim("aud", "my-api")
				.build();

		jwt().jwt(prebuilt)
				.authorities(new SimpleGrantedAuthority("ROLE_SERVICE"))
				.postProcessRequest(new MockHttpServletRequest());

		JwtAuthenticationToken auth = (JwtAuthenticationToken) SecurityContextHolder
				.getContext().getAuthentication();
		assertThat(auth.getToken().getTokenValue()).isEqualTo("my-custom-token");
		assertThat(auth.getName()).isEqualTo("service-account");
		assertThat(auth.getAuthorities()).extracting("authority")
				.containsExactly("ROLE_SERVICE");
	}

}
```
