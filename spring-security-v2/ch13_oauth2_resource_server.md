# Chapter 13: OAuth2 Resource Server (JWT)

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure a JWT-based OAuth2 Resource Server:
SimpleHttpSecurity http = new SimpleHttpSecurity();
http.oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt
                .decoder(SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build())));
SecurityFilterChain chain = http.build();

// GET /api/data with Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
// -> extracts Bearer token from Authorization header
// -> decodes and verifies JWT signature using the RSA public key
// -> checks expiration
// -> converts "scope" claim into SCOPE_xxx granted authorities
// -> sets SecurityContext with JwtAuthenticationToken and CONTINUES the chain

// No token? -> passes through (request continues unauthenticated)
// Expired/invalid token? -> 401 with WWW-Authenticate: Bearer error="invalid_token"
```

Questions to guide your thinking:
- Why does `SimpleBearerTokenAuthenticationFilter` implement `Filter` directly instead of extending `SimpleAbstractAuthenticationProcessingFilter`? (Hint: think about Feature 6's HTTP Basic filter.)
- Why does the configurer create its *own* `AuthenticationManager` with a `SimpleJwtAuthenticationProvider` rather than using the shared `AuthenticationManager` from `SimpleHttpSecurity`?
- How does the `JwtDecoder` interface enable the decoder to be swapped (RSA public key, HMAC secret, remote JWK Set URI) without changing the filter or provider?
- Why does `SimpleBearerTokenAuthenticationEntryPoint` include the OAuth2 error details in the `WWW-Authenticate` header rather than in the response body?

---

## The API Contract

Feature 13 introduces JWT-based Bearer token authentication for resource server APIs.
It is exposed as a convenience method on `SimpleHttpSecurity` backed by a configurer
that produces a filter.

### OAuth2 Resource Server DSL

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity oauth2ResourceServer(
        Customizer<SimpleOAuth2ResourceServerConfigurer> oauth2ResourceServerCustomizer) {
    oauth2ResourceServerCustomizer.customize(getOrApply(new SimpleOAuth2ResourceServerConfigurer()));
    return this;
}
```

The configurer exposes these DSL methods:

```java
// simple/security/config/annotation/web/configurers/oauth2/SimpleOAuth2ResourceServerConfigurer.java
public class SimpleOAuth2ResourceServerConfigurer
        extends SimpleAbstractHttpConfigurer<SimpleOAuth2ResourceServerConfigurer> {

    public SimpleOAuth2ResourceServerConfigurer jwt(Customizer<JwtConfigurer> jwtCustomizer) { ... }
    public SimpleOAuth2ResourceServerConfigurer authenticationEntryPoint(AuthenticationEntryPoint ep) { ... }

    // Inner class for JWT-specific configuration
    public static class JwtConfigurer {
        public JwtConfigurer decoder(JwtDecoder decoder) { ... }
        public JwtConfigurer jwtAuthenticationConverter(SimpleJwtAuthenticationConverter converter) { ... }
    }
}
```

### The nested DSL pattern

The `oauth2ResourceServer()` DSL uses a **nested configurer** pattern. The outer configurer
(`SimpleOAuth2ResourceServerConfigurer`) handles the protocol-agnostic parts (entry point,
filter wiring). The inner configurer (`JwtConfigurer`) handles JWT-specific settings (decoder,
authority converter). This mirrors the real framework's design where opaque token support
would be a sibling inner class alongside `JwtConfigurer`:

```java
http.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt
        .decoder(SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build())
        .jwtAuthenticationConverter(customConverter))
    .authenticationEntryPoint(customEntryPoint));
```

### Key types

The OAuth2 resource server feature spans three new modules and introduces several key types:

```java
// Core types: OAuth2-specific error and exception
OAuth2Error error = new OAuth2Error("invalid_token", "JWT expired", null);
throw new OAuth2AuthenticationException(error);

// JWT model: decoded token with headers, claims, and typed accessors
Jwt jwt = decoder.decode("eyJhbGciOiJSUzI1NiJ9...");
String subject = jwt.getSubject();
Instant expiration = jwt.getExpiresAt();

// Decoder: FunctionalInterface that converts raw token string to Jwt
JwtDecoder decoder = SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();

// Authentication tokens: unauthenticated (raw string) and authenticated (decoded JWT)
BearerTokenAuthenticationToken unauthenticated = new BearerTokenAuthenticationToken("eyJ...");
JwtAuthenticationToken authenticated = new JwtAuthenticationToken(jwt, authorities);
```

---

## Client Usage & Tests

The tests in `OAuth2ResourceServerApiTest` define the behavioral contract from a client's
perspective. Each test builds a complete filter chain and sends mock HTTP requests with
JWT Bearer tokens through it.

### Configuring the resource server

```java
@Test
void shouldConfigureOAuth2ResourceServerViaBuilder() throws Exception {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt
                    .decoder(SimpleNimbusJwtDecoder.withPublicKey(publicKey).build())));

    SecurityFilterChain chain = http.build();

    // Verify the Bearer token filter is in the chain
    assertThat(chain.getFilters())
            .anyMatch(f -> f instanceof SimpleBearerTokenAuthenticationFilter);
}
```

### Authenticating a valid JWT

```java
@Test
void shouldAuthenticateValidJwt() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    String jwt = createSignedJwt("user123", "read write", Instant.now().plusSeconds(3600));

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Bearer " + jwt);
    MockHttpServletResponse response = new MockHttpServletResponse();

    // Capture the authentication inside the chain (before proxy clears context)
    AtomicReference<Authentication> captured = new AtomicReference<>();
    FilterChain capturingChain = (req, res) ->
            captured.set(SecurityContextHolder.getContext().getAuthentication());

    proxy.doFilter(request, response, capturingChain);

    // The request should pass through (filter chain continues)
    assertThat(response.getStatus()).isEqualTo(200);

    // SecurityContext contained the authenticated JWT token during processing
    assertThat(captured.get()).isInstanceOf(JwtAuthenticationToken.class);
    assertThat(captured.get().isAuthenticated()).isTrue();
    assertThat(captured.get().getName()).isEqualTo("user123");
}
```

### Scope-based authorities

```java
@Test
void shouldExtractScopeAuthorities() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    String jwt = createSignedJwt("user123", "read write admin", Instant.now().plusSeconds(3600));

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Bearer " + jwt);
    MockHttpServletResponse response = new MockHttpServletResponse();

    AtomicReference<Authentication> captured = new AtomicReference<>();
    FilterChain capturingChain = (req, res) ->
            captured.set(SecurityContextHolder.getContext().getAuthentication());

    proxy.doFilter(request, response, capturingChain);

    assertThat(captured.get().getAuthorities())
            .extracting("authority")
            .containsExactlyInAnyOrder("SCOPE_read", "SCOPE_write", "SCOPE_admin");
}
```

### Accessing JWT claims from the Authentication

```java
@Test
void shouldAccessJwtClaimsFromAuthentication() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    String jwt = createSignedJwt("user123", "read", Instant.now().plusSeconds(3600));

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Bearer " + jwt);
    MockHttpServletResponse response = new MockHttpServletResponse();

    AtomicReference<Authentication> captured = new AtomicReference<>();
    FilterChain capturingChain = (req, res) ->
            captured.set(SecurityContextHolder.getContext().getAuthentication());

    proxy.doFilter(request, response, capturingChain);

    JwtAuthenticationToken jwtAuth = (JwtAuthenticationToken) captured.get();
    assertThat(jwtAuth.getToken().getSubject()).isEqualTo("user123");
    assertThat(jwtAuth.getToken().getIssuer()).isEqualTo("https://idp.example.com");
    assertThat(jwtAuth.getTokenAttributes()).containsKey("sub");
}
```

### Passthrough when no token

```java
@Test
void shouldPassThroughWhenNoAuthorizationHeader() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    // Request with no Authorization header
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();

    proxy.doFilter(request, response, chain);

    // Request should pass through -- no authentication set
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
}

@Test
void shouldPassThroughWhenNonBearerAuthorizationHeader() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Basic dXNlcjpwYXNzd29yZA==");
    MockHttpServletResponse response = new MockHttpServletResponse();
    MockFilterChain chain = new MockFilterChain();

    proxy.doFilter(request, response, chain);

    // Non-Bearer token -> filter passes through
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
}
```

### Invalid tokens produce 401

```java
@Test
void shouldReject401WhenJwtExpired() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    // Create an expired JWT
    String jwt = createSignedJwt("user123", "read", Instant.now().minusSeconds(3600));

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Bearer " + jwt);
    MockHttpServletResponse response = new MockHttpServletResponse();

    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(401);
    assertThat(response.getHeader("WWW-Authenticate")).contains("invalid_token");
    assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
}

@Test
void shouldReject401WhenJwtSignedWithWrongKey() throws Exception {
    SimpleFilterChainProxy proxy = buildProxy();

    // Create a JWT signed with a DIFFERENT key
    KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
    gen.initialize(2048);
    KeyPair wrongKeyPair = gen.generateKeyPair();
    RSAPrivateKey wrongPrivateKey = (RSAPrivateKey) wrongKeyPair.getPrivate();

    JWSSigner wrongSigner = new RSASSASigner(wrongPrivateKey);
    JWTClaimsSet claims = new JWTClaimsSet.Builder()
            .subject("attacker")
            .issuer("https://evil.example.com")
            .expirationTime(Date.from(Instant.now().plusSeconds(3600)))
            .build();
    SignedJWT signedJWT = new SignedJWT(
            new JWSHeader(JWSAlgorithm.RS256), claims);
    signedJWT.sign(wrongSigner);

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Bearer " + signedJWT.serialize());
    MockHttpServletResponse response = new MockHttpServletResponse();

    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(401);
    assertThat(response.getHeader("WWW-Authenticate")).contains("invalid_token");
}
```

### Custom entry point

```java
@Test
void shouldSupportCustomAuthenticationEntryPoint() throws Exception {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt.decoder(SimpleNimbusJwtDecoder.withPublicKey(publicKey).build()))
            .authenticationEntryPoint((req, res, ex) -> {
                res.setStatus(401);
                res.setContentType("application/json");
                res.getWriter().write("{\"error\":\"unauthorized\"}");
            }));

    SecurityFilterChain chain = http.build();
    SimpleFilterChainProxy proxy = new SimpleFilterChainProxy(List.of(chain));

    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
    request.addHeader("Authorization", "Bearer invalid.jwt.token");
    MockHttpServletResponse response = new MockHttpServletResponse();

    proxy.doFilter(request, response, new MockFilterChain());

    assertThat(response.getStatus()).isEqualTo(401);
    assertThat(response.getContentAsString()).contains("unauthorized");
}
```

---

## Implementing the Call Chain

### Call chain recap

```
Client calls http.oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.decoder(myDecoder)))
  |
  +---> SimpleHttpSecurity.oauth2ResourceServer()
  |       getOrApply(new SimpleOAuth2ResourceServerConfigurer()) -- register configurer
  |       customizer.customize(configurer) -- user's lambda calls .jwt(...)
  |         SimpleOAuth2ResourceServerConfigurer.jwt()
  |           creates JwtConfigurer, customizer calls .decoder(myDecoder)
  |
  +---> Client calls http.build()
  |       |
  |       +---> Phase 1: INITIALIZING
  |       |       SimpleOAuth2ResourceServerConfigurer.init(http)
  |       |         Creates SimpleBearerTokenAuthenticationEntryPoint
  |       |         Registers as shared entry point (if none already set by form login)
  |       |
  |       +---> Phase 2: beforeConfigure()
  |       |       Resolves AuthenticationManager -> setSharedObject (if configured)
  |       |
  |       +---> Phase 3: CONFIGURING
  |       |       SimpleOAuth2ResourceServerConfigurer.configure(http)
  |       |         JwtConfigurer.getAuthenticationProvider()
  |       |           Creates SimpleJwtAuthenticationProvider(jwtDecoder)
  |       |           Sets optional jwtAuthenticationConverter
  |       |         Wraps provider in a dedicated SimpleProviderManager
  |       |         Creates SimpleBearerTokenAuthenticationFilter(jwtAuthManager, entryPoint)
  |       |         Calls http.addFilter(filter) -> position 450
  |       |
  |       +---> Phase 4: performBuild()
  |               Sort filters: [..., 450: BearerTokenFilter, ...]
  |               Wrap in SimpleDefaultSecurityFilterChain
  |
  +---> At request time:
          GET /api/data + Authorization: Bearer eyJ...
            SimpleBearerTokenResolver.resolve(request)
              Extracts token from "Authorization: Bearer <token>" header
            BearerTokenAuthenticationToken(token)
              Wraps raw string in unauthenticated Authentication
            AuthenticationManager.authenticate(bearerToken)
              SimpleJwtAuthenticationProvider.authenticate()
                JwtDecoder.decode(token) -- parse, verify signature, validate expiration
                SimpleJwtAuthenticationConverter.convert(jwt) -- extract scope authorities
                Returns JwtAuthenticationToken(jwt, authorities)
            SecurityContextHolder.setContext(authenticated)
            chain.doFilter() -- CONTINUE the chain (resource server always continues)

          GET /api/data (no token) -> resolver returns null -> pass through
          GET /api/data + Bearer <expired> -> decode throws -> entryPoint sends 401
```

### 13a. Core types: OAuth2Error and OAuth2AuthenticationException

Before building the resource server components, we need two foundational types that
represent OAuth2-specific errors.

**`OAuth2Error`** models the structured error from the OAuth2 spec -- an error code
(e.g., `"invalid_token"`), an optional description, and an optional URI:

```java
public class OAuth2Error {
    private final String errorCode;
    private final String description;
    private final String uri;

    public OAuth2Error(String errorCode) {
        this(errorCode, null, null);
    }

    public OAuth2Error(String errorCode, String description, String uri) {
        if (errorCode == null || errorCode.isEmpty()) {
            throw new IllegalArgumentException("errorCode cannot be empty");
        }
        this.errorCode = errorCode;
        this.description = description;
        this.uri = uri;
    }
    // getters...
}
```

**`OAuth2AuthenticationException`** bridges the OAuth2 error model with Spring Security's
exception hierarchy. It extends `AuthenticationException` (from Feature 2) and carries
an `OAuth2Error`:

```java
public class OAuth2AuthenticationException extends AuthenticationException {
    private final OAuth2Error error;

    public OAuth2AuthenticationException(OAuth2Error error) {
        this(error, error.getDescription());
    }
    // overloaded constructors with message and cause...

    public OAuth2Error getError() {
        return this.error;
    }
}
```

**Why a dedicated exception type?** The entry point needs to inspect the error to build
the `WWW-Authenticate` header. A generic `AuthenticationException` has no error code or
URI. By carrying the structured `OAuth2Error`, the exception gives the entry point
everything it needs to produce an RFC 6750-compliant response.

### 13b. The JWT model: Jwt

The `Jwt` class represents a decoded JSON Web Token. Unlike `BearerTokenAuthenticationToken`
(which holds the raw string), `Jwt` contains the parsed headers, claims, and timestamps:

```java
public class Jwt {
    public static final String ISSUER = "iss";
    public static final String SUBJECT = "sub";
    public static final String AUDIENCE = "aud";
    public static final String EXPIRATION = "exp";
    // ... more standard claim names

    private final String tokenValue;
    private final Instant issuedAt;
    private final Instant expiresAt;
    private final Map<String, Object> headers;   // unmodifiable
    private final Map<String, Object> claims;    // unmodifiable
```

Typed accessors provide convenient access to standard claims:

```java
    public String getSubject() { return getClaim(SUBJECT); }
    public String getIssuer()  { ... }
    public List<String> getAudience() { ... }
    public Instant getNotBefore() { ... }
```

The **builder pattern** makes `Jwt` easy to construct in tests without needing a real
JWT signing process:

```java
Jwt jwt = Jwt.withTokenValue("token-value")
    .header("alg", "RS256")
    .subject("bob")
    .issuer("https://issuer.example.com")
    .issuedAt(Instant.now())
    .expiresAt(Instant.now().plusSeconds(3600))
    .claim("custom", "value")
    .build();
```

**Why are headers and claims stored as unmodifiable maps?** Once decoded and verified,
a JWT is immutable data. Allowing mutation after verification would break the security
guarantee that the claims are trustworthy. The constructor copies the maps into
`Collections.unmodifiableMap()` to enforce this.

### 13c. The decoder interface and Nimbus implementation

**`JwtDecoder`** is a `@FunctionalInterface` -- a single method that converts a raw
JWT string into a verified `Jwt`:

```java
@FunctionalInterface
public interface JwtDecoder {
    Jwt decode(String token) throws JwtException;
}
```

**`SimpleNimbusJwtDecoder`** is the concrete implementation backed by the Nimbus JOSE+JWT
library. Its `decode` method follows three steps:

```java
public Jwt decode(String token) throws JwtException {
    JWT parsedJwt = parse(token);         // 1. Parse compact string, reject unsigned JWTs
    Jwt jwt = createJwt(token, parsedJwt); // 2. Verify signature, extract claims
    validateJwt(jwt);                      // 3. Check expiration
    return jwt;
}
```

The decoder rejects unsigned (plain) JWTs early -- this is a security-critical check:

```java
private JWT parse(String token) {
    try {
        JWT jwt = JWTParser.parse(token);
        if (jwt instanceof PlainJWT) {
            throw new BadJwtException("Unsupported algorithm: none (unsigned JWTs are rejected)");
        }
        return jwt;
    }
    catch (ParseException ex) {
        throw new BadJwtException("Failed to parse JWT: " + ex.getMessage(), ex);
    }
}
```

**Three builder patterns** cover the most common key sources:

```java
// RSA public key (e.g., from a PEM file)
JwtDecoder decoder = SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();

// HMAC shared secret
JwtDecoder decoder = SimpleNimbusJwtDecoder.withSecretKey(secretKey).build();

// Remote JWK Set endpoint (e.g., /.well-known/jwks.json)
JwtDecoder decoder = SimpleNimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build();
```

**Why three builders instead of one constructor?** Each key type requires a completely
different `JWSKeySelector` and potentially different `JWSAlgorithm` defaults (RS256 for
RSA, HS256 for HMAC). The builder pattern lets each key type set up its own Nimbus
`DefaultJWTProcessor` pipeline correctly.

### 13d. The resolver: SimpleBearerTokenResolver

This class extracts the Bearer token from the HTTP `Authorization` header per RFC 6750:

```java
public class SimpleBearerTokenResolver {

    private static final Pattern AUTHORIZATION_PATTERN =
            Pattern.compile("^Bearer (?<token>[a-zA-Z0-9-._~+/]+=*)$", Pattern.CASE_INSENSITIVE);

    public String resolve(HttpServletRequest request) {
        String authorization = request.getHeader("Authorization");
        if (authorization == null) {
            return null;
        }
        if (!authorization.regionMatches(true, 0, "bearer", 0, 6)) {
            return null;
        }
        Matcher matcher = AUTHORIZATION_PATTERN.matcher(authorization);
        if (!matcher.matches()) {
            return null;
        }
        return matcher.group("token");
    }
}
```

**Why return `null` instead of throwing?** A missing Bearer token is not an error -- the
request might be using a different authentication mechanism (form login, HTTP Basic) or
might be accessing a public endpoint. Returning `null` tells the filter to pass through
and let downstream filters or authorization rules decide.

**Why `regionMatches` before the regex?** Quick prefix check avoids the cost of regex
matching on every request that uses a different auth scheme (e.g., `Basic`).

### 13e. The filter: SimpleBearerTokenAuthenticationFilter

This filter implements `Filter` directly -- the same design choice as HTTP Basic in Feature 6,
and for the same reasons:

1. **Every request**: It inspects *every* request for a Bearer token (not just one URL).
2. **Continue chain**: On success, it always continues the filter chain -- the request
   proceeds to the actual API resource.
3. **Stateless**: Each request carries its own token. No login ceremony.

```java
public class SimpleBearerTokenAuthenticationFilter implements Filter {

    private final AuthenticationManager authenticationManager;
    private final AuthenticationEntryPoint authenticationEntryPoint;
    private SimpleBearerTokenResolver bearerTokenResolver = new SimpleBearerTokenResolver();

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
            FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        // Step 1: Try to extract Bearer token
        String token = this.bearerTokenResolver.resolve(request);

        // Step 2: No token -> pass through (request continues unauthenticated)
        if (token == null) {
            chain.doFilter(request, response);
            return;
        }

        // Step 3: Wrap in unauthenticated token
        BearerTokenAuthenticationToken authenticationRequest = new BearerTokenAuthenticationToken(token);

        try {
            // Step 4: Authenticate via AuthenticationManager
            Authentication authenticationResult =
                    this.authenticationManager.authenticate(authenticationRequest);

            // Step 5: Store in SecurityContext and continue
            SecurityContext context = new SecurityContextImpl();
            context.setAuthentication(authenticationResult);
            SecurityContextHolder.setContext(context);

            chain.doFilter(request, response);
        }
        catch (AuthenticationException ex) {
            // Step 6: Clear context and send 401
            SecurityContextHolder.clearContext();
            this.authenticationEntryPoint.commence(request, response, ex);
        }
    }
}
```

**Comparison to `SimpleBasicAuthenticationFilter`**: The Bearer filter is structurally
almost identical to the Basic filter -- extract credentials, authenticate, set context,
continue chain. The key difference is what gets extracted (a JWT string vs. username:password)
and what the `AuthenticationManager` does with it (JWT decode vs. password verification).

### 13f. The authentication tokens: BearerTokenAuthenticationToken and JwtAuthenticationToken

These two tokens represent the before and after of JWT authentication:

**`BearerTokenAuthenticationToken`** -- the *unauthenticated* state. Created by the filter
after extracting the raw Bearer token string:

```java
public class BearerTokenAuthenticationToken extends AbstractAuthenticationToken {
    private final String token;

    public BearerTokenAuthenticationToken(String token) {
        super(null);              // no authorities yet
        this.token = token;
        setAuthenticated(false);  // explicitly unauthenticated
    }

    public String getToken() { return this.token; }

    @Override
    public Object getCredentials() { return this.token; }

    @Override
    public Object getPrincipal() { return this.token; }
}
```

**`JwtAuthenticationToken`** -- the *authenticated* state. Created by the converter after
the JWT has been decoded and verified:

```java
public class JwtAuthenticationToken extends AbstractAuthenticationToken {
    private final Jwt jwt;
    private final String name;

    public JwtAuthenticationToken(Jwt jwt, Collection<? extends GrantedAuthority> authorities) {
        this(jwt, authorities, jwt.getSubject());
    }

    public JwtAuthenticationToken(Jwt jwt, Collection<? extends GrantedAuthority> authorities,
            String name) {
        super(authorities);
        this.jwt = jwt;
        this.name = name;
        setAuthenticated(true);   // verified JWT = authenticated
    }

    @Override
    public Object getPrincipal() { return this.jwt; }

    @Override
    public Object getCredentials() { return this.jwt.getTokenValue(); }

    @Override
    public String getName() { return this.name; }  // typically the "sub" claim

    public Jwt getToken() { return this.jwt; }

    public Map<String, Object> getTokenAttributes() { return this.jwt.getClaims(); }
}
```

**Why is the `Jwt` itself the principal?** In username/password authentication, the
principal is a `UserDetails`. For JWT authentication, all the identity information lives
in the token's claims. The `Jwt` object *is* the identity -- it carries the subject,
issuer, audience, custom claims, and more. Wrapping it in a separate principal object
would just add indirection.

### 13g. The converter: SimpleJwtAuthenticationConverter

The converter bridges from decoded `Jwt` to authenticated `JwtAuthenticationToken` by
extracting granted authorities from the token's claims:

```java
public class SimpleJwtAuthenticationConverter {
    private static final String DEFAULT_AUTHORITY_PREFIX = "SCOPE_";

    private Function<Jwt, Collection<GrantedAuthority>> jwtGrantedAuthoritiesConverter;

    public SimpleJwtAuthenticationConverter() {
        this.jwtGrantedAuthoritiesConverter = this::defaultGrantedAuthorities;
    }

    public JwtAuthenticationToken convert(Jwt jwt) {
        Collection<GrantedAuthority> authorities = this.jwtGrantedAuthoritiesConverter.apply(jwt);
        return new JwtAuthenticationToken(jwt, authorities);
    }
```

The default authority extraction reads the `scope` (or `scp`) claim and prefixes each
scope with `SCOPE_`:

```java
    private Collection<GrantedAuthority> defaultGrantedAuthorities(Jwt jwt) {
        Object scopeClaim = jwt.getClaim("scope");
        if (scopeClaim == null) {
            scopeClaim = jwt.getClaim("scp");
        }
        if (scopeClaim == null) {
            return Collections.emptyList();
        }

        Collection<String> scopes;
        if (scopeClaim instanceof String scopeString) {
            scopes = Arrays.asList(scopeString.split(" "));
        }
        else if (scopeClaim instanceof Collection<?> scopeCollection) {
            scopes = scopeCollection.stream().map(Object::toString).collect(Collectors.toList());
        }
        else {
            return Collections.emptyList();
        }

        return scopes.stream()
                .map(scope -> new SimpleGrantedAuthority(DEFAULT_AUTHORITY_PREFIX + scope))
                .collect(Collectors.toUnmodifiableList());
    }
```

**Why both `scope` and `scp`?** Different OAuth2 providers use different claim names.
Auth0 uses `scope` (a space-delimited string), Azure AD uses `scp`, and some providers
use a list. The converter handles all three formats.

**Why is the authority extractor a `Function` rather than a fixed method?** Customization.
Many organizations encode authorities in custom claims (e.g., `"roles": ["ADMIN", "USER"]`
instead of `"scope": "read write"`). The `setJwtGrantedAuthoritiesConverter` method allows
replacing the default extraction logic entirely:

```java
converter.setJwtGrantedAuthoritiesConverter(jwt -> {
    List<String> roles = jwt.getClaim("roles");
    return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .toList();
});
```

### 13h. The provider: SimpleJwtAuthenticationProvider

The provider is the glue between the filter and the JWT infrastructure. It receives an
unauthenticated `BearerTokenAuthenticationToken`, decodes the JWT, converts it, and
returns an authenticated `JwtAuthenticationToken`:

```java
public class SimpleJwtAuthenticationProvider implements AuthenticationProvider {

    private static final OAuth2Error INVALID_TOKEN = new OAuth2Error(
            "invalid_token", "An error occurred while attempting to decode the Jwt", null);

    private final JwtDecoder jwtDecoder;
    private SimpleJwtAuthenticationConverter jwtAuthenticationConverter =
            new SimpleJwtAuthenticationConverter();

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        BearerTokenAuthenticationToken bearer = (BearerTokenAuthenticationToken) authentication;
        Jwt jwt;
        try {
            jwt = this.jwtDecoder.decode(bearer.getToken());
        }
        catch (BadJwtException ex) {
            OAuth2Error error = new OAuth2Error("invalid_token", ex.getMessage(), null);
            throw new OAuth2AuthenticationException(error, error.getDescription(), ex);
        }
        catch (JwtException ex) {
            throw new OAuth2AuthenticationException(INVALID_TOKEN, INVALID_TOKEN.getDescription(), ex);
        }
        JwtAuthenticationToken token = this.jwtAuthenticationConverter.convert(jwt);
        token.setDetails(bearer.getDetails());
        return token;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return BearerTokenAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

**Why two catch blocks?** `BadJwtException` carries a specific error message (e.g.,
"JWT expired at 2024-01-15T...") that should be relayed to the client. A generic
`JwtException` indicates an unexpected processing error -- the generic `INVALID_TOKEN`
message avoids leaking internal details.

**Why `supports(BearerTokenAuthenticationToken.class)`?** This is the `AuthenticationProvider`
contract from Feature 2. The `ProviderManager` loops through all registered providers
and calls `authenticate()` only on the one that supports the given token type. This
ensures the JWT provider is not accidentally called with a `UsernamePasswordAuthenticationToken`.

### 13i. The entry point: SimpleBearerTokenAuthenticationEntryPoint

When authentication fails, this entry point sends a `401 Unauthorized` response with
a `WWW-Authenticate` header per RFC 6750:

```java
public class SimpleBearerTokenAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        String wwwAuthenticate;
        if (authException instanceof OAuth2AuthenticationException oauth2Ex) {
            OAuth2Error error = oauth2Ex.getError();
            StringBuilder sb = new StringBuilder("Bearer");
            sb.append(" error=\"").append(error.getErrorCode()).append("\"");
            if (error.getDescription() != null) {
                sb.append(", error_description=\"").append(error.getDescription()).append("\"");
            }
            if (error.getUri() != null) {
                sb.append(", error_uri=\"").append(error.getUri()).append("\"");
            }
            wwwAuthenticate = sb.toString();
        }
        else {
            wwwAuthenticate = "Bearer";
        }

        response.addHeader("WWW-Authenticate", wwwAuthenticate);
    }
}
```

**Why put error details in the `WWW-Authenticate` header instead of the body?** RFC 6750
Section 3 specifies that Bearer token errors should be communicated via the
`WWW-Authenticate` response header. Clients (including OAuth2 client libraries) expect
to find `error` and `error_description` there. The response body can be empty or contain
additional details, but the header is the standard location.

**Why the `instanceof OAuth2AuthenticationException` check?** If the entry point is
invoked by a non-OAuth2 exception (e.g., from the exception translation filter for an
unauthenticated request with no token), it still returns a valid `Bearer` challenge
without error details.

### 13j. The configurer: SimpleOAuth2ResourceServerConfigurer

The configurer ties everything together through the two-phase lifecycle:

**`init()`** -- registers the entry point:

```java
@Override
public void init(SimpleHttpSecurity builder) throws Exception {
    AuthenticationEntryPoint entryPoint = getAuthenticationEntryPoint();
    // Register as shared entry point if none is set (form login takes priority)
    if (builder.getSharedObject(AuthenticationEntryPoint.class) == null) {
        builder.setSharedObject(AuthenticationEntryPoint.class, entryPoint);
    }
}
```

**`configure()`** -- creates the filter with a **dedicated** `AuthenticationManager`:

```java
@Override
public void configure(SimpleHttpSecurity builder) throws Exception {
    if (this.jwtConfigurer == null) {
        throw new IllegalStateException(
                "oauth2ResourceServer requires JWT configuration. "
                        + "Call .jwt(jwt -> jwt.decoder(myDecoder))");
    }

    // Build a dedicated AuthenticationManager for JWT tokens
    AuthenticationProvider jwtProvider = this.jwtConfigurer.getAuthenticationProvider();
    AuthenticationManager jwtAuthManager = new SimpleProviderManager(List.of(jwtProvider));

    AuthenticationEntryPoint entryPoint = getAuthenticationEntryPoint();

    SimpleBearerTokenAuthenticationFilter filter =
            new SimpleBearerTokenAuthenticationFilter(jwtAuthManager, entryPoint);

    builder.addFilter(filter);
}
```

**Why a dedicated `AuthenticationManager` instead of using the shared one?** The shared
`AuthenticationManager` (from `http.authenticationManager(...)`) handles
`UsernamePasswordAuthenticationToken` for form login and HTTP Basic. The JWT provider
only handles `BearerTokenAuthenticationToken`. Using a separate manager means:

1. The JWT provider is never accidentally given username/password tokens.
2. The form login / HTTP Basic providers are never given Bearer tokens.
3. The resource server works even if no shared `AuthenticationManager` is configured
   (you don't need `http.authenticationManager(...)` for pure resource server setups).

The **nested `JwtConfigurer`** builds the `AuthenticationProvider`:

```java
public static class JwtConfigurer {
    private JwtDecoder decoder;
    private SimpleJwtAuthenticationConverter jwtAuthenticationConverter;

    AuthenticationProvider getAuthenticationProvider() {
        if (this.decoder == null) {
            throw new IllegalStateException(
                    "A JwtDecoder must be configured. Call .decoder(myJwtDecoder)");
        }
        SimpleJwtAuthenticationProvider provider = new SimpleJwtAuthenticationProvider(this.decoder);
        if (this.jwtAuthenticationConverter != null) {
            provider.setJwtAuthenticationConverter(this.jwtAuthenticationConverter);
        }
        return provider;
    }
}
```

### 13k. DSL method on SimpleHttpSecurity

The convenience method follows the same `getOrApply` pattern as form login and HTTP Basic:

```java
public SimpleHttpSecurity oauth2ResourceServer(
        Customizer<SimpleOAuth2ResourceServerConfigurer> oauth2ResourceServerCustomizer) {
    oauth2ResourceServerCustomizer.customize(getOrApply(new SimpleOAuth2ResourceServerConfigurer()));
    return this;
}
```

The filter order map was also updated to include the Bearer token filter at position 450:

```java
static {
    int order = 100;
    FILTER_ORDER_MAP.put("SimpleSecurityContextHolderFilter", order);      // 100
    order += 100;
    FILTER_ORDER_MAP.put("SimpleCsrfFilter", order);                       // 200
    order += 50;
    FILTER_ORDER_MAP.put("SimpleLogoutFilter", order);                     // 250
    order += 50;
    FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", order); // 300
    order += 100;
    FILTER_ORDER_MAP.put("SimpleBasicAuthenticationFilter", order);        // 400
    order += 50;
    FILTER_ORDER_MAP.put("SimpleBearerTokenAuthenticationFilter", order);  // 450   <-- NEW
    order += 50;
    FILTER_ORDER_MAP.put("SimpleExceptionTranslationFilter", order);       // 500
    order += 100;
    FILTER_ORDER_MAP.put("SimpleAuthorizationFilter", order);              // 600
}
```

**Why position 450 (after HTTP Basic at 400)?** Authentication filters should all run
before exception translation (500) and authorization (600). Bearer token authentication
sits after HTTP Basic because they can coexist -- a resource server might accept both
Basic credentials and JWT tokens. The ordering ensures credentials from either mechanism
are processed before access control decisions.

---

## Try It Yourself

**Challenge 1**: Add opaque token support. Create a `SimpleOpaqueTokenIntrospector`
interface with a method `OAuth2AuthenticatedPrincipal introspect(String token)` and
a `SimpleOpaqueTokenAuthenticationProvider` that calls it. Add an `opaqueToken()`
method to `SimpleOAuth2ResourceServerConfigurer` as a sibling to `jwt()`. How does
the provider distinguish between a JWT and an opaque token?

**Challenge 2**: Extend `SimpleNimbusJwtDecoder` to support custom claim validation
beyond just expiration. Add a `setClaimValidator(Predicate<Jwt>)` method that runs
after decoding. Write a test that rejects tokens without a required `aud` (audience)
claim. How does this relate to the real framework's `OAuth2TokenValidator` interface?

**Challenge 3**: Make `SimpleBearerTokenResolver` support extracting tokens from a
query parameter (`?access_token=eyJ...`) in addition to the Authorization header, as
allowed by RFC 6750 Section 2.3. Add a `setAllowUriQueryParameter(boolean)` method.
Why is query parameter transport discouraged for security reasons?

---

## Why This Works

**The `JwtDecoder` as a strategy interface**: By defining a single-method interface for
JWT decoding, the entire authentication pipeline becomes agnostic to how tokens are
verified. The filter does not know whether the token is verified with an RSA public key,
an HMAC secret, or a remote JWK Set endpoint. Swapping the decoder swaps the entire
verification strategy without touching the filter, provider, or converter.

**Dedicated `AuthenticationManager` per authentication mechanism**: The OAuth2 resource
server configurer creates its own `SimpleProviderManager` wrapping a
`SimpleJwtAuthenticationProvider`. This is a key architectural decision. Unlike form
login and HTTP Basic (which share the top-level `AuthenticationManager`), the resource
server brings its own authentication infrastructure. This isolation means JWT
authentication works independently -- you can have a resource server with no
`UserDetailsService`, no password encoder, no shared `AuthenticationManager`.

**The two-token pattern**: `BearerTokenAuthenticationToken` (unauthenticated, raw string)
and `JwtAuthenticationToken` (authenticated, decoded JWT with authorities) mirror the
`UsernamePasswordAuthenticationToken` pattern from Feature 2. The unauthenticated token
is a request. The authenticated token is the result. The provider transforms one into
the other. This pattern makes the `ProviderManager` dispatch mechanism work consistently
across all authentication types.

**RFC 6750 compliance in the entry point**: The entry point writes the `WWW-Authenticate`
header with the `error` and `error_description` attributes per the OAuth2 Bearer Token
spec. This means standard OAuth2 client libraries can parse the error and take appropriate
action (refresh the token, re-authenticate, etc.) without custom error handling.

**Same filter design as HTTP Basic**: `SimpleBearerTokenAuthenticationFilter` has the
exact same structure as `SimpleBasicAuthenticationFilter` from Feature 6 -- inspect every
request, pass through if no credentials, authenticate if present, always continue the
chain on success. The design converged because Bearer tokens and Basic auth have the same
lifecycle: stateless, per-request, continue-to-resource. This is why neither extends
`SimpleAbstractAuthenticationProcessingFilter` (which is for one-time login ceremonies
like form login).

---

## What We Enhanced

| What Changed | Before (Feature 12) | After (Feature 13) | Why |
|---|---|---|---|
| `SimpleHttpSecurity` | No OAuth2 support | Added `oauth2ResourceServer()` DSL method and `SimpleBearerTokenAuthenticationFilter` to filter order map (position 450) | Enables JWT Bearer token authentication via the same DSL pattern |
| New modules | None | `simple-security-oauth2-core`, `simple-security-oauth2-jose`, `simple-security-oauth2-resource-server` | Mirrors Spring Security's multi-module OAuth2 architecture |
| Authentication tokens | `UsernamePasswordAuthenticationToken` only | + `BearerTokenAuthenticationToken` (unauthenticated) and `JwtAuthenticationToken` (authenticated) | Different credential types need different token representations |
| Provider pattern | Providers for username/password | + `SimpleJwtAuthenticationProvider` with dedicated `AuthenticationManager` | JWT authentication is self-contained and does not share the global auth manager |
| Exception hierarchy | `AuthenticationException` subtypes | + `OAuth2AuthenticationException` carrying `OAuth2Error` | RFC 6750 requires structured error responses with error codes |
| Entry points | Login redirect, Basic realm challenge | + `SimpleBearerTokenAuthenticationEntryPoint` with `WWW-Authenticate: Bearer error="..."` | Each authentication mechanism needs its own challenge format |
| `simple-security-config/build.gradle` | No OAuth2 dependencies | Added `implementation project(':simple-security-oauth2-jose')` and `implementation project(':simple-security-oauth2-resource-server')` | Config module needs access to JWT decoder and resource server types for the configurer |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|---|---|---|
| `OAuth2Error` | `OAuth2Error` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/OAuth2Error.java` |
| `OAuth2AuthenticationException` | `OAuth2AuthenticationException` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/OAuth2AuthenticationException.java` |
| `Jwt` | `Jwt` | `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/Jwt.java` |
| `JwtDecoder` | `JwtDecoder` | `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/JwtDecoder.java` |
| `JwtException` | `JwtException` | `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/JwtException.java` |
| `BadJwtException` | `BadJwtException` | `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/BadJwtException.java` |
| `SimpleNimbusJwtDecoder` | `NimbusJwtDecoder` | `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/NimbusJwtDecoder.java` |
| `BearerTokenAuthenticationToken` | `BearerTokenAuthenticationToken` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/BearerTokenAuthenticationToken.java` |
| `JwtAuthenticationToken` | `JwtAuthenticationToken` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationToken.java` |
| `SimpleJwtAuthenticationConverter` | `JwtAuthenticationConverter` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationConverter.java` |
| `SimpleJwtAuthenticationProvider` | `JwtAuthenticationProvider` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationProvider.java` |
| `SimpleBearerTokenResolver` | `DefaultBearerTokenResolver` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/DefaultBearerTokenResolver.java` |
| `SimpleBearerTokenAuthenticationFilter` | `BearerTokenAuthenticationFilter` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/authentication/BearerTokenAuthenticationFilter.java` |
| `SimpleBearerTokenAuthenticationEntryPoint` | `BearerTokenAuthenticationEntryPoint` | `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/BearerTokenAuthenticationEntryPoint.java` |
| `SimpleOAuth2ResourceServerConfigurer` | `OAuth2ResourceServerConfigurer` | `config/src/main/java/org/springframework/security/config/annotation/web/configurers/oauth2/server/resource/OAuth2ResourceServerConfigurer.java` |

### Key differences from the real framework

**`NimbusJwtDecoder`**: The real class supports custom `OAuth2TokenValidator` chains
(issuer validation, audience validation, custom claim checks), pluggable
`JWTClaimsSetConverter`, cache configuration for the JWK Set URI, and custom
`RestOperations` for HTTP calls. Our simplified version validates only expiration
and uses Nimbus's default HTTP client.

**`OAuth2ResourceServerConfigurer`**: The real class supports both JWT and opaque token
authentication, configures an `AuthenticationManagerResolver` for multi-tenant scenarios,
registers a `BearerTokenRequestMatcher` for CSRF exemption, and configures an
`AccessDeniedHandler` for insufficient scope errors. Our version focuses on the JWT
path only.

**`BearerTokenAuthenticationFilter`**: The real class extends `OncePerRequestFilter`,
uses `AuthenticationConverter` (instead of a direct resolver), supports
`SecurityContextHolderStrategy`, integrates with `SecurityContextRepository`, and has
an `AuthenticationFailureHandler`. Our version captures the essential flow: resolve
token, authenticate, set context, continue chain.

**`JwtAuthenticationConverter`**: The real class implements `Converter<Jwt, AbstractAuthenticationToken>`
and supports a pluggable `JwtGrantedAuthoritiesConverter` (which itself is a `Converter`).
It also supports a custom principal converter via `setPrincipalClaimName`. Our version
uses a `Function<Jwt, Collection<GrantedAuthority>>` for authority extraction, which
serves the same purpose with less abstraction.

**`Jwt`**: The real class extends `AbstractOAuth2Token` and implements
`JwtClaimAccessor` (which extends `ClaimAccessor`). These interfaces provide generic
claim access methods. Our version puts typed accessors directly on the `Jwt` class.

---

## Complete Code

### [NEW] `simple-security-oauth2-core/src/main/java/simple/security/oauth2/core/OAuth2Error.java`

```java
package simple.security.oauth2.core;

/**
 * A representation of an OAuth 2.0 Error.
 *
 * <p>OAuth 2.0 errors carry an error code (e.g., "invalid_token"), an optional
 * description, and an optional URI pointing to further information. This model
 * is used throughout the OAuth2 modules to communicate specific error conditions.
 *
 * <p>Simplifications vs. real OAuth2Error:
 * <ul>
 *   <li>No serialization support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.OAuth2Error
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/OAuth2Error.java
 */
public class OAuth2Error {

	private final String errorCode;

	private final String description;

	private final String uri;

	public OAuth2Error(String errorCode) {
		this(errorCode, null, null);
	}

	public OAuth2Error(String errorCode, String description, String uri) {
		if (errorCode == null || errorCode.isEmpty()) {
			throw new IllegalArgumentException("errorCode cannot be empty");
		}
		this.errorCode = errorCode;
		this.description = description;
		this.uri = uri;
	}

	public String getErrorCode() {
		return this.errorCode;
	}

	public String getDescription() {
		return this.description;
	}

	public String getUri() {
		return this.uri;
	}

	@Override
	public String toString() {
		return "[" + this.errorCode + "] "
				+ (this.description != null ? this.description : "");
	}

}
```

### [NEW] `simple-security-oauth2-core/src/main/java/simple/security/oauth2/core/OAuth2AuthenticationException.java`

```java
package simple.security.oauth2.core;

import simple.security.core.AuthenticationException;

/**
 * An {@link AuthenticationException} that carries an {@link OAuth2Error}.
 *
 * <p>Thrown by OAuth2-related components (Bearer token filter, JWT decoder, etc.)
 * when authentication fails due to an OAuth2-specific error such as an invalid token,
 * expired token, or insufficient scope.
 *
 * <p>Simplifications vs. real OAuth2AuthenticationException:
 * <ul>
 *   <li>No serialization support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.OAuth2AuthenticationException
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/OAuth2AuthenticationException.java
 */
public class OAuth2AuthenticationException extends AuthenticationException {

	private final OAuth2Error error;

	public OAuth2AuthenticationException(OAuth2Error error) {
		this(error, error.getDescription());
	}

	public OAuth2AuthenticationException(OAuth2Error error, String message) {
		super(message);
		this.error = error;
	}

	public OAuth2AuthenticationException(OAuth2Error error, Throwable cause) {
		this(error, error.getDescription(), cause);
	}

	public OAuth2AuthenticationException(OAuth2Error error, String message, Throwable cause) {
		super(message, cause);
		this.error = error;
	}

	public OAuth2Error getError() {
		return this.error;
	}

}
```

### [NEW] `simple-security-oauth2-jose/src/main/java/simple/security/oauth2/jwt/Jwt.java`

```java
package simple.security.oauth2.jwt;

import java.time.Instant;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * A decoded JSON Web Token (JWT).
 *
 * <p>A JWT consists of three parts: a JOSE header, a claims set, and a signature.
 * This class represents the decoded form -- the raw token string plus the parsed
 * headers and claims as unmodifiable maps.
 *
 * <p>The class provides typed accessors for standard JWT claims (issuer, subject,
 * audience, expiration, etc.) as well as generic access to any claim via
 * {@link #getClaim(String)}.
 *
 * <p>Simplifications vs. real Jwt:
 * <ul>
 *   <li>No AbstractOAuth2Token base class -- claims and token value stored directly</li>
 *   <li>No JwtClaimAccessor interface -- typed accessors are methods on this class</li>
 *   <li>No serialization support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.jwt.Jwt
 * Source:  oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/Jwt.java
 */
public class Jwt {

	// Standard JWT claim names
	public static final String ISSUER = "iss";
	public static final String SUBJECT = "sub";
	public static final String AUDIENCE = "aud";
	public static final String EXPIRATION = "exp";
	public static final String NOT_BEFORE = "nbf";
	public static final String ISSUED_AT = "iat";
	public static final String ID = "jti";

	private final String tokenValue;

	private final Instant issuedAt;

	private final Instant expiresAt;

	private final Map<String, Object> headers;

	private final Map<String, Object> claims;

	/**
	 * Creates a new Jwt with the given token value, timestamps, headers, and claims.
	 */
	public Jwt(String tokenValue, Instant issuedAt, Instant expiresAt,
			Map<String, Object> headers, Map<String, Object> claims) {
		if (tokenValue == null || tokenValue.isEmpty()) {
			throw new IllegalArgumentException("tokenValue cannot be empty");
		}
		if (headers == null || headers.isEmpty()) {
			throw new IllegalArgumentException("headers cannot be empty");
		}
		if (claims == null || claims.isEmpty()) {
			throw new IllegalArgumentException("claims cannot be empty");
		}
		this.tokenValue = tokenValue;
		this.issuedAt = issuedAt;
		this.expiresAt = expiresAt;
		this.headers = Collections.unmodifiableMap(new HashMap<>(headers));
		this.claims = Collections.unmodifiableMap(new HashMap<>(claims));
	}

	public String getTokenValue() {
		return this.tokenValue;
	}

	public Instant getIssuedAt() {
		return this.issuedAt;
	}

	public Instant getExpiresAt() {
		return this.expiresAt;
	}

	public Map<String, Object> getHeaders() {
		return this.headers;
	}

	public Map<String, Object> getClaims() {
		return this.claims;
	}

	@SuppressWarnings("unchecked")
	public <T> T getClaim(String name) {
		return (T) this.claims.get(name);
	}

	// --- Typed claim accessors ---

	public String getIssuer() {
		Object iss = this.claims.get(ISSUER);
		return iss != null ? iss.toString() : null;
	}

	public String getSubject() {
		return getClaim(SUBJECT);
	}

	@SuppressWarnings("unchecked")
	public List<String> getAudience() {
		Object aud = this.claims.get(AUDIENCE);
		if (aud instanceof List) {
			return (List<String>) aud;
		}
		if (aud instanceof String s) {
			return List.of(s);
		}
		return null;
	}

	public Instant getNotBefore() {
		return getInstantClaim(NOT_BEFORE);
	}

	public String getId() {
		return getClaim(ID);
	}

	private Instant getInstantClaim(String claimName) {
		Object value = this.claims.get(claimName);
		if (value instanceof Instant instant) {
			return instant;
		}
		if (value instanceof Number number) {
			return Instant.ofEpochSecond(number.longValue());
		}
		return null;
	}

	/**
	 * Returns a new builder initialized with the given token value.
	 */
	public static Builder withTokenValue(String tokenValue) {
		return new Builder(tokenValue);
	}

	/**
	 * A fluent builder for {@link Jwt} instances, useful for testing.
	 */
	public static final class Builder {

		private final String tokenValue;
		private final Map<String, Object> headers = new HashMap<>();
		private final Map<String, Object> claims = new HashMap<>();

		private Builder(String tokenValue) {
			this.tokenValue = tokenValue;
		}

		public Builder header(String name, Object value) {
			this.headers.put(name, value);
			return this;
		}

		public Builder headers(Map<String, Object> headers) {
			this.headers.putAll(headers);
			return this;
		}

		public Builder claim(String name, Object value) {
			this.claims.put(name, value);
			return this;
		}

		public Builder claims(Map<String, Object> claims) {
			this.claims.putAll(claims);
			return this;
		}

		public Builder issuer(String issuer) {
			return claim(ISSUER, issuer);
		}

		public Builder subject(String subject) {
			return claim(SUBJECT, subject);
		}

		public Builder audience(List<String> audience) {
			return claim(AUDIENCE, audience);
		}

		public Builder expiresAt(Instant expiresAt) {
			return claim(EXPIRATION, expiresAt);
		}

		public Builder issuedAt(Instant issuedAt) {
			return claim(ISSUED_AT, issuedAt);
		}

		public Builder notBefore(Instant notBefore) {
			return claim(NOT_BEFORE, notBefore);
		}

		public Builder jti(String jti) {
			return claim(ID, jti);
		}

		public Jwt build() {
			Instant iat = toInstant(this.claims.get(ISSUED_AT));
			Instant exp = toInstant(this.claims.get(EXPIRATION));
			return new Jwt(this.tokenValue, iat, exp, this.headers, this.claims);
		}

		private Instant toInstant(Object value) {
			if (value instanceof Instant instant) return instant;
			if (value instanceof Number number) return Instant.ofEpochSecond(number.longValue());
			return null;
		}

	}

}
```

### [NEW] `simple-security-oauth2-jose/src/main/java/simple/security/oauth2/jwt/JwtDecoder.java`

```java
package simple.security.oauth2.jwt;

/**
 * Decodes a JSON Web Token (JWT) from its compact serialized form.
 *
 * <p>Implementations are responsible for parsing the token, verifying the
 * cryptographic signature, and returning a {@link Jwt} with the decoded
 * headers and claims.
 *
 * <p>This is a {@link FunctionalInterface} -- a single method that converts
 * a raw JWT string into a {@link Jwt} object. The primary implementation is
 * {@link SimpleNimbusJwtDecoder}, which uses the Nimbus JOSE+JWT library.
 *
 * <p>Simplifications vs. real JwtDecoder:
 * <ul>
 *   <li>Identical contract -- this is already minimal</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.jwt.JwtDecoder
 * Source:  oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/JwtDecoder.java
 */
@FunctionalInterface
public interface JwtDecoder {

	/**
	 * Decodes the JWT from its compact claims representation format and returns
	 * a {@link Jwt}.
	 *
	 * @param token the JWT value
	 * @return a {@link Jwt}
	 * @throws JwtException if an error occurs while attempting to decode the JWT
	 */
	Jwt decode(String token) throws JwtException;

}
```

### [NEW] `simple-security-oauth2-jose/src/main/java/simple/security/oauth2/jwt/JwtException.java`

```java
package simple.security.oauth2.jwt;

/**
 * Base exception for JWT processing errors.
 *
 * Mirrors: org.springframework.security.oauth2.jwt.JwtException
 * Source:  oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/JwtException.java
 */
public class JwtException extends RuntimeException {

	public JwtException(String message) {
		super(message);
	}

	public JwtException(String message, Throwable cause) {
		super(message, cause);
	}

}
```

### [NEW] `simple-security-oauth2-jose/src/main/java/simple/security/oauth2/jwt/BadJwtException.java`

```java
package simple.security.oauth2.jwt;

/**
 * Thrown when a JWT is invalid -- malformed, unsigned, expired, or fails
 * signature verification.
 *
 * Mirrors: org.springframework.security.oauth2.jwt.BadJwtException
 * Source:  oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/BadJwtException.java
 */
public class BadJwtException extends JwtException {

	public BadJwtException(String message) {
		super(message);
	}

	public BadJwtException(String message, Throwable cause) {
		super(message, cause);
	}

}
```

### [NEW] `simple-security-oauth2-jose/src/main/java/simple/security/oauth2/jwt/SimpleNimbusJwtDecoder.java`

```java
package simple.security.oauth2.jwt;

import java.security.interfaces.RSAPublicKey;
import java.text.ParseException;
import java.time.Instant;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

import javax.crypto.SecretKey;

import com.nimbusds.jose.JOSEException;
import com.nimbusds.jose.JWSAlgorithm;
import com.nimbusds.jose.jwk.source.ImmutableJWKSet;
import com.nimbusds.jose.jwk.source.JWKSource;
import com.nimbusds.jose.jwk.source.RemoteJWKSet;
import com.nimbusds.jose.proc.BadJOSEException;
import com.nimbusds.jose.proc.JWSKeySelector;
import com.nimbusds.jose.proc.JWSVerificationKeySelector;
import com.nimbusds.jose.proc.SecurityContext;
import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jose.jwk.OctetSequenceKey;
import com.nimbusds.jose.jwk.RSAKey;
import com.nimbusds.jwt.JWT;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.JWTParser;
import com.nimbusds.jwt.PlainJWT;
import com.nimbusds.jwt.SignedJWT;
import com.nimbusds.jwt.proc.DefaultJWTProcessor;

/**
 * A {@link JwtDecoder} that uses the Nimbus JOSE+JWT library to parse and verify JWTs.
 *
 * <p>This decoder handles the full JWT verification pipeline:
 * <ol>
 *   <li>Parse the compact JWT string into a Nimbus JWT object</li>
 *   <li>Reject unsigned (plain) JWTs</li>
 *   <li>Verify the cryptographic signature using the configured key source</li>
 *   <li>Convert the Nimbus claims into a Spring Security {@link Jwt}</li>
 *   <li>Validate the token (expiration check)</li>
 * </ol>
 *
 * <p>Three builder patterns are provided for the most common key sources:
 * <ul>
 *   <li>{@link #withPublicKey(RSAPublicKey)} -- for RSA public keys (e.g., from a PEM file)</li>
 *   <li>{@link #withSecretKey(SecretKey)} -- for HMAC shared secrets</li>
 *   <li>{@link #withJwkSetUri(String)} -- for remote JWK Set endpoints (e.g., {@code /.well-known/jwks.json})</li>
 * </ul>
 *
 * <p>Simplifications vs. real NimbusJwtDecoder:
 * <ul>
 *   <li>No custom claim set converters -- uses direct conversion</li>
 *   <li>No custom validators -- only checks expiration</li>
 *   <li>No cache configuration for JWK Set URI</li>
 *   <li>No issuer location discovery</li>
 *   <li>No custom RestOperations -- Nimbus uses its default HTTP client</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.jwt.NimbusJwtDecoder
 * Source:  oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/NimbusJwtDecoder.java
 */
public final class SimpleNimbusJwtDecoder implements JwtDecoder {

	private final DefaultJWTProcessor<SecurityContext> jwtProcessor;

	private SimpleNimbusJwtDecoder(DefaultJWTProcessor<SecurityContext> jwtProcessor) {
		this.jwtProcessor = jwtProcessor;
	}

	@Override
	public Jwt decode(String token) throws JwtException {
		JWT parsedJwt = parse(token);
		Jwt jwt = createJwt(token, parsedJwt);
		validateJwt(jwt);
		return jwt;
	}

	private JWT parse(String token) {
		try {
			JWT jwt = JWTParser.parse(token);
			if (jwt instanceof PlainJWT) {
				throw new BadJwtException("Unsupported algorithm: none (unsigned JWTs are rejected)");
			}
			return jwt;
		}
		catch (ParseException ex) {
			throw new BadJwtException("Failed to parse JWT: " + ex.getMessage(), ex);
		}
	}

	private Jwt createJwt(String token, JWT parsedJwt) {
		try {
			JWTClaimsSet claimsSet = this.jwtProcessor.process(parsedJwt, null);
			Map<String, Object> headers = extractHeaders(parsedJwt);
			Map<String, Object> claims = convertClaims(claimsSet);
			Instant iat = toInstant(claimsSet.getIssueTime());
			Instant exp = toInstant(claimsSet.getExpirationTime());
			return new Jwt(token, iat, exp, headers, claims);
		}
		catch (BadJOSEException ex) {
			throw new BadJwtException("Failed to validate JWT: " + ex.getMessage(), ex);
		}
		catch (JOSEException ex) {
			throw new BadJwtException("Failed to process JWT: " + ex.getMessage(), ex);
		}
	}

	private void validateJwt(Jwt jwt) {
		Instant expiration = jwt.getExpiresAt();
		if (expiration != null && Instant.now().isAfter(expiration)) {
			throw new BadJwtException("JWT expired at " + expiration);
		}
	}

	private Map<String, Object> extractHeaders(JWT jwt) {
		Map<String, Object> headers = new HashMap<>(jwt.getHeader().toJSONObject());
		return headers;
	}

	private Map<String, Object> convertClaims(JWTClaimsSet claimsSet) {
		Map<String, Object> claims = new HashMap<>();
		for (Map.Entry<String, Object> entry : claimsSet.getClaims().entrySet()) {
			Object value = entry.getValue();
			if (value instanceof Date date) {
				claims.put(entry.getKey(), date.toInstant());
			}
			else {
				claims.put(entry.getKey(), value);
			}
		}
		return claims;
	}

	private Instant toInstant(Date date) {
		return date != null ? date.toInstant() : null;
	}

	// === Builders ===

	/**
	 * Returns a builder for creating a decoder that verifies JWTs using the
	 * given RSA public key with the RS256 algorithm.
	 */
	public static PublicKeyJwtDecoderBuilder withPublicKey(RSAPublicKey key) {
		return new PublicKeyJwtDecoderBuilder(key);
	}

	/**
	 * Returns a builder for creating a decoder that verifies JWTs using the
	 * given HMAC secret key with the HS256 algorithm.
	 */
	public static SecretKeyJwtDecoderBuilder withSecretKey(SecretKey key) {
		return new SecretKeyJwtDecoderBuilder(key);
	}

	/**
	 * Returns a builder for creating a decoder that fetches the JWK Set from
	 * the given URI to verify JWT signatures.
	 */
	public static JwkSetUriJwtDecoderBuilder withJwkSetUri(String jwkSetUri) {
		return new JwkSetUriJwtDecoderBuilder(jwkSetUri);
	}

	// --- Public Key Builder ---

	/**
	 * Builder for creating a decoder backed by an RSA public key.
	 */
	public static final class PublicKeyJwtDecoderBuilder {

		private final RSAPublicKey key;
		private JWSAlgorithm algorithm = JWSAlgorithm.RS256;

		private PublicKeyJwtDecoderBuilder(RSAPublicKey key) {
			if (key == null) {
				throw new IllegalArgumentException("key cannot be null");
			}
			this.key = key;
		}

		public PublicKeyJwtDecoderBuilder signatureAlgorithm(JWSAlgorithm algorithm) {
			this.algorithm = algorithm;
			return this;
		}

		public SimpleNimbusJwtDecoder build() {
			RSAKey rsaKey = new RSAKey.Builder(this.key).build();
			JWKSet jwkSet = new JWKSet(rsaKey);
			JWKSource<SecurityContext> jwkSource = new ImmutableJWKSet<>(jwkSet);
			JWSKeySelector<SecurityContext> keySelector =
					new JWSVerificationKeySelector<>(this.algorithm, jwkSource);
			DefaultJWTProcessor<SecurityContext> processor = new DefaultJWTProcessor<>();
			processor.setJWSKeySelector(keySelector);
			// No-op verifier -- Spring Security handles claim validation independently
			processor.setJWTClaimsSetVerifier((claimsSet, context) -> { });
			return new SimpleNimbusJwtDecoder(processor);
		}

	}

	// --- Secret Key Builder ---

	/**
	 * Builder for creating a decoder backed by an HMAC secret key.
	 */
	public static final class SecretKeyJwtDecoderBuilder {

		private final SecretKey key;
		private JWSAlgorithm algorithm = JWSAlgorithm.HS256;

		private SecretKeyJwtDecoderBuilder(SecretKey key) {
			if (key == null) {
				throw new IllegalArgumentException("key cannot be null");
			}
			this.key = key;
		}

		public SecretKeyJwtDecoderBuilder signatureAlgorithm(JWSAlgorithm algorithm) {
			this.algorithm = algorithm;
			return this;
		}

		public SimpleNimbusJwtDecoder build() {
			OctetSequenceKey octKey = new OctetSequenceKey.Builder(this.key).build();
			JWKSet jwkSet = new JWKSet(octKey);
			JWKSource<SecurityContext> jwkSource = new ImmutableJWKSet<>(jwkSet);
			JWSKeySelector<SecurityContext> keySelector =
					new JWSVerificationKeySelector<>(this.algorithm, jwkSource);
			DefaultJWTProcessor<SecurityContext> processor = new DefaultJWTProcessor<>();
			processor.setJWSKeySelector(keySelector);
			// No-op verifier -- Spring Security handles claim validation independently
			processor.setJWTClaimsSetVerifier((claimsSet, context) -> { });
			return new SimpleNimbusJwtDecoder(processor);
		}

	}

	// --- JWK Set URI Builder ---

	/**
	 * Builder for creating a decoder that fetches keys from a remote JWK Set endpoint.
	 */
	public static final class JwkSetUriJwtDecoderBuilder {

		private final String jwkSetUri;
		private JWSAlgorithm algorithm = JWSAlgorithm.RS256;

		private JwkSetUriJwtDecoderBuilder(String jwkSetUri) {
			if (jwkSetUri == null || jwkSetUri.isEmpty()) {
				throw new IllegalArgumentException("jwkSetUri cannot be empty");
			}
			this.jwkSetUri = jwkSetUri;
		}

		public JwkSetUriJwtDecoderBuilder signatureAlgorithm(JWSAlgorithm algorithm) {
			this.algorithm = algorithm;
			return this;
		}

		public SimpleNimbusJwtDecoder build() {
			try {
				java.net.URL url = java.net.URI.create(this.jwkSetUri).toURL();
				JWKSource<SecurityContext> jwkSource = new RemoteJWKSet<>(url);
				JWSKeySelector<SecurityContext> keySelector =
						new JWSVerificationKeySelector<>(this.algorithm, jwkSource);
				DefaultJWTProcessor<SecurityContext> processor = new DefaultJWTProcessor<>();
				processor.setJWSKeySelector(keySelector);
				// No-op verifier -- Spring Security handles claim validation independently
			processor.setJWTClaimsSetVerifier((claimsSet, context) -> { });
				return new SimpleNimbusJwtDecoder(processor);
			}
			catch (java.net.MalformedURLException ex) {
				throw new IllegalArgumentException("Invalid JWK Set URI: " + this.jwkSetUri, ex);
			}
		}

	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/authentication/BearerTokenAuthenticationToken.java`

```java
package simple.security.oauth2.server.resource.authentication;

import simple.security.authentication.AbstractAuthenticationToken;

/**
 * An unauthenticated {@link simple.security.core.Authentication} that holds the raw
 * Bearer token string extracted from the HTTP request.
 *
 * <p>This token is created by the {@link simple.security.oauth2.server.resource.web.SimpleBearerTokenAuthenticationFilter}
 * and passed to the {@link simple.security.authentication.AuthenticationManager} for
 * authentication. The provider will decode the JWT, verify it, and produce a
 * {@link JwtAuthenticationToken} as the authenticated result.
 *
 * <p>Simplifications vs. real BearerTokenAuthenticationToken:
 * <ul>
 *   <li>No serialization</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.BearerTokenAuthenticationToken
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/BearerTokenAuthenticationToken.java
 */
public class BearerTokenAuthenticationToken extends AbstractAuthenticationToken {

	private final String token;

	/**
	 * Creates an unauthenticated token holding the raw Bearer token string.
	 *
	 * @param token the bearer token value
	 */
	public BearerTokenAuthenticationToken(String token) {
		super(null);
		if (token == null || token.isEmpty()) {
			throw new IllegalArgumentException("token cannot be empty");
		}
		this.token = token;
		setAuthenticated(false);
	}

	/**
	 * Returns the raw Bearer token string.
	 */
	public String getToken() {
		return this.token;
	}

	@Override
	public Object getCredentials() {
		return this.token;
	}

	@Override
	public Object getPrincipal() {
		return this.token;
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/authentication/JwtAuthenticationToken.java`

```java
package simple.security.oauth2.server.resource.authentication;

import java.util.Collection;

import simple.security.authentication.AbstractAuthenticationToken;
import simple.security.core.GrantedAuthority;
import simple.security.oauth2.jwt.Jwt;

/**
 * An authenticated {@link simple.security.core.Authentication} that wraps a decoded
 * {@link Jwt} as its principal, along with the granted authorities extracted from
 * the token's claims.
 *
 * <p>This is the authentication object stored in the {@link simple.security.core.context.SecurityContext}
 * after a Bearer token has been successfully decoded and validated.
 *
 * <p>{@link #getName()} returns the JWT {@code sub} (subject) claim by default,
 * or a custom name if provided.
 *
 * <p>Simplifications vs. real JwtAuthenticationToken:
 * <ul>
 *   <li>No AbstractOAuth2TokenAuthenticationToken base class</li>
 *   <li>No toBuilder() / builder pattern</li>
 *   <li>No separate principal object -- the Jwt itself is the principal</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationToken.java
 */
public class JwtAuthenticationToken extends AbstractAuthenticationToken {

	private final Jwt jwt;

	private final String name;

	/**
	 * Creates an authenticated token with the given JWT and authorities.
	 * The name defaults to the JWT's {@code sub} claim.
	 */
	public JwtAuthenticationToken(Jwt jwt, Collection<? extends GrantedAuthority> authorities) {
		this(jwt, authorities, jwt.getSubject());
	}

	/**
	 * Creates an authenticated token with the given JWT, authorities, and name.
	 */
	public JwtAuthenticationToken(Jwt jwt, Collection<? extends GrantedAuthority> authorities,
			String name) {
		super(authorities);
		if (jwt == null) {
			throw new IllegalArgumentException("jwt cannot be null");
		}
		this.jwt = jwt;
		this.name = name;
		setAuthenticated(true);
	}

	/**
	 * Returns the decoded JWT.
	 */
	public Jwt getToken() {
		return this.jwt;
	}

	/**
	 * Returns the decoded JWT as the principal.
	 */
	@Override
	public Object getPrincipal() {
		return this.jwt;
	}

	/**
	 * Returns the raw token value as the credentials.
	 */
	@Override
	public Object getCredentials() {
		return this.jwt.getTokenValue();
	}

	/**
	 * Returns the name of the authenticated principal -- typically the {@code sub} claim.
	 */
	@Override
	public String getName() {
		return this.name;
	}

	/**
	 * Returns the JWT claims as token attributes.
	 */
	public java.util.Map<String, Object> getTokenAttributes() {
		return this.jwt.getClaims();
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/authentication/SimpleJwtAuthenticationConverter.java`

```java
package simple.security.oauth2.server.resource.authentication;

import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.stream.Collectors;

import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.oauth2.jwt.Jwt;

/**
 * Converts a decoded {@link Jwt} into a {@link JwtAuthenticationToken} by extracting
 * granted authorities from the token's claims.
 *
 * <p>By default, authorities are extracted from the {@code scope} or {@code scp} claim
 * and prefixed with {@code "SCOPE_"}. For example, a JWT with
 * {@code "scope": "read write"} produces authorities {@code SCOPE_read} and
 * {@code SCOPE_write}.
 *
 * <p>The authority extraction can be customized by providing a custom
 * {@code jwtGrantedAuthoritiesConverter} function.
 *
 * <p>Simplifications vs. real JwtAuthenticationConverter:
 * <ul>
 *   <li>No Converter interface -- just a direct convert method</li>
 *   <li>No custom principal converter</li>
 *   <li>No FactorGrantedAuthority (BEARER_AUTHORITY)</li>
 *   <li>Default authority prefix is always "SCOPE_"</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationConverter.java
 */
public class SimpleJwtAuthenticationConverter {

	private static final String DEFAULT_AUTHORITY_PREFIX = "SCOPE_";

	private java.util.function.Function<Jwt, Collection<GrantedAuthority>> jwtGrantedAuthoritiesConverter;

	public SimpleJwtAuthenticationConverter() {
		this.jwtGrantedAuthoritiesConverter = this::defaultGrantedAuthorities;
	}

	/**
	 * Converts the given {@link Jwt} into a {@link JwtAuthenticationToken}.
	 */
	public JwtAuthenticationToken convert(Jwt jwt) {
		Collection<GrantedAuthority> authorities = this.jwtGrantedAuthoritiesConverter.apply(jwt);
		return new JwtAuthenticationToken(jwt, authorities);
	}

	/**
	 * Sets a custom function for extracting granted authorities from a JWT.
	 */
	public void setJwtGrantedAuthoritiesConverter(
			java.util.function.Function<Jwt, Collection<GrantedAuthority>> converter) {
		this.jwtGrantedAuthoritiesConverter = converter;
	}

	/**
	 * Default authority extraction: reads the "scope" or "scp" claim, splits by space,
	 * and prefixes each scope with "SCOPE_".
	 */
	private Collection<GrantedAuthority> defaultGrantedAuthorities(Jwt jwt) {
		Object scopeClaim = jwt.getClaim("scope");
		if (scopeClaim == null) {
			scopeClaim = jwt.getClaim("scp");
		}
		if (scopeClaim == null) {
			return Collections.emptyList();
		}

		Collection<String> scopes;
		if (scopeClaim instanceof String scopeString) {
			scopes = Arrays.asList(scopeString.split(" "));
		}
		else if (scopeClaim instanceof Collection<?> scopeCollection) {
			scopes = scopeCollection.stream()
					.map(Object::toString)
					.collect(Collectors.toList());
		}
		else {
			return Collections.emptyList();
		}

		return scopes.stream()
				.map(scope -> new SimpleGrantedAuthority(DEFAULT_AUTHORITY_PREFIX + scope))
				.collect(Collectors.toUnmodifiableList());
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/authentication/SimpleJwtAuthenticationProvider.java`

```java
package simple.security.oauth2.server.resource.authentication;

import simple.security.authentication.AuthenticationProvider;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.oauth2.core.OAuth2AuthenticationException;
import simple.security.oauth2.core.OAuth2Error;
import simple.security.oauth2.jwt.BadJwtException;
import simple.security.oauth2.jwt.Jwt;
import simple.security.oauth2.jwt.JwtDecoder;
import simple.security.oauth2.jwt.JwtException;

/**
 * An {@link AuthenticationProvider} that authenticates Bearer tokens by decoding
 * them as JWTs and converting them to {@link JwtAuthenticationToken}.
 *
 * <p>This is the core authentication logic for JWT-based resource servers:
 * <ol>
 *   <li>Receives a {@link BearerTokenAuthenticationToken} (raw token string)</li>
 *   <li>Calls {@link JwtDecoder#decode(String)} to parse and verify the JWT</li>
 *   <li>Calls {@link SimpleJwtAuthenticationConverter#convert(Jwt)} to produce an authenticated token</li>
 * </ol>
 *
 * <p>If decoding fails (bad signature, expired, malformed), this provider wraps the
 * error in an {@link OAuth2AuthenticationException} with the {@code invalid_token} error code.
 *
 * <p>Simplifications vs. real JwtAuthenticationProvider:
 * <ul>
 *   <li>No custom Converter interface -- uses SimpleJwtAuthenticationConverter directly</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationProvider
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationProvider.java
 */
public class SimpleJwtAuthenticationProvider implements AuthenticationProvider {

	private static final OAuth2Error INVALID_TOKEN = new OAuth2Error(
			"invalid_token", "An error occurred while attempting to decode the Jwt", null);

	private final JwtDecoder jwtDecoder;

	private SimpleJwtAuthenticationConverter jwtAuthenticationConverter = new SimpleJwtAuthenticationConverter();

	public SimpleJwtAuthenticationProvider(JwtDecoder jwtDecoder) {
		if (jwtDecoder == null) {
			throw new IllegalArgumentException("jwtDecoder cannot be null");
		}
		this.jwtDecoder = jwtDecoder;
	}

	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		BearerTokenAuthenticationToken bearer = (BearerTokenAuthenticationToken) authentication;
		Jwt jwt;
		try {
			jwt = this.jwtDecoder.decode(bearer.getToken());
		}
		catch (BadJwtException ex) {
			OAuth2Error error = new OAuth2Error("invalid_token", ex.getMessage(), null);
			throw new OAuth2AuthenticationException(error, error.getDescription(), ex);
		}
		catch (JwtException ex) {
			throw new OAuth2AuthenticationException(INVALID_TOKEN, INVALID_TOKEN.getDescription(), ex);
		}
		JwtAuthenticationToken token = this.jwtAuthenticationConverter.convert(jwt);
		token.setDetails(bearer.getDetails());
		return token;
	}

	@Override
	public boolean supports(Class<?> authentication) {
		return BearerTokenAuthenticationToken.class.isAssignableFrom(authentication);
	}

	public void setJwtAuthenticationConverter(SimpleJwtAuthenticationConverter converter) {
		this.jwtAuthenticationConverter = converter;
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/web/SimpleBearerTokenResolver.java`

```java
package simple.security.oauth2.server.resource.web;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves a Bearer token from the {@code Authorization} header of an HTTP request.
 *
 * <p>Implements the Bearer Token extraction as described in RFC 6750 Section 2.1.
 * Looks for the pattern {@code Authorization: Bearer <token>} in the request header
 * and extracts the token value.
 *
 * <p>Simplifications vs. real DefaultBearerTokenResolver:
 * <ul>
 *   <li>Only supports Authorization header -- no query parameter or form body fallback</li>
 *   <li>No configurable allowFormEncodedBodyParameter or allowUriQueryParameter</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.web.DefaultBearerTokenResolver
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/DefaultBearerTokenResolver.java
 */
public class SimpleBearerTokenResolver {

	private static final Pattern AUTHORIZATION_PATTERN =
			Pattern.compile("^Bearer (?<token>[a-zA-Z0-9-._~+/]+=*)$", Pattern.CASE_INSENSITIVE);

	/**
	 * Resolves the Bearer token from the request's {@code Authorization} header.
	 *
	 * @param request the HTTP request
	 * @return the token string, or {@code null} if no Bearer token is present
	 */
	public String resolve(HttpServletRequest request) {
		String authorization = request.getHeader("Authorization");
		if (authorization == null) {
			return null;
		}
		if (!authorization.regionMatches(true, 0, "bearer", 0, 6)) {
			return null;
		}
		Matcher matcher = AUTHORIZATION_PATTERN.matcher(authorization);
		if (!matcher.matches()) {
			return null;
		}
		return matcher.group("token");
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/web/SimpleBearerTokenAuthenticationFilter.java`

```java
package simple.security.oauth2.server.resource.web;

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
import simple.security.core.context.SecurityContextImpl;
import simple.security.oauth2.server.resource.authentication.BearerTokenAuthenticationToken;
import simple.security.web.AuthenticationEntryPoint;

/**
 * A {@link Filter} that authenticates requests carrying a Bearer token in the
 * {@code Authorization} header.
 *
 * <h2>Processing flow</h2>
 * <ol>
 *   <li>Extract the Bearer token from the request via {@link SimpleBearerTokenResolver}</li>
 *   <li>If no token is found, pass through -- the request continues unauthenticated</li>
 *   <li>Wrap the raw token string in a {@link BearerTokenAuthenticationToken}</li>
 *   <li>Delegate to the {@link AuthenticationManager} which calls {@link simple.security.oauth2.server.resource.authentication.SimpleJwtAuthenticationProvider}</li>
 *   <li>On success: store the authenticated token in the {@link SecurityContext} and continue the chain</li>
 *   <li>On failure: clear the SecurityContext and delegate to the {@link AuthenticationEntryPoint}</li>
 * </ol>
 *
 * <p>This filter implements {@link Filter} directly (not extending an abstract
 * processing filter) because, like HTTP Basic, it inspects <em>every</em> request
 * for a token and always continues the filter chain on success.
 *
 * <p>Simplifications vs. real BearerTokenAuthenticationFilter:
 * <ul>
 *   <li>No AuthenticationConverter abstraction -- uses SimpleBearerTokenResolver directly</li>
 *   <li>No AuthenticationFailureHandler -- delegates directly to entry point</li>
 *   <li>No SecurityContextRepository -- stores in SecurityContextHolder only</li>
 *   <li>No DPoP support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationFilter
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/authentication/BearerTokenAuthenticationFilter.java
 */
public class SimpleBearerTokenAuthenticationFilter implements Filter {

	private final AuthenticationManager authenticationManager;

	private final AuthenticationEntryPoint authenticationEntryPoint;

	private SimpleBearerTokenResolver bearerTokenResolver = new SimpleBearerTokenResolver();

	public SimpleBearerTokenAuthenticationFilter(AuthenticationManager authenticationManager,
			AuthenticationEntryPoint authenticationEntryPoint) {
		if (authenticationManager == null) {
			throw new IllegalArgumentException("authenticationManager cannot be null");
		}
		if (authenticationEntryPoint == null) {
			throw new IllegalArgumentException("authenticationEntryPoint cannot be null");
		}
		this.authenticationManager = authenticationManager;
		this.authenticationEntryPoint = authenticationEntryPoint;
	}

	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
			FilterChain chain) throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) servletRequest;
		HttpServletResponse response = (HttpServletResponse) servletResponse;

		// Step 1: Try to extract Bearer token
		String token = this.bearerTokenResolver.resolve(request);

		// Step 2: No token -> pass through (request continues unauthenticated)
		if (token == null) {
			chain.doFilter(request, response);
			return;
		}

		// Step 3: Wrap in unauthenticated token
		BearerTokenAuthenticationToken authenticationRequest = new BearerTokenAuthenticationToken(token);

		try {
			// Step 4: Authenticate via AuthenticationManager
			Authentication authenticationResult = this.authenticationManager.authenticate(authenticationRequest);

			// Step 5: Store in SecurityContext and continue
			SecurityContext context = new SecurityContextImpl();
			context.setAuthentication(authenticationResult);
			SecurityContextHolder.setContext(context);

			chain.doFilter(request, response);
		}
		catch (AuthenticationException ex) {
			// Step 6: Clear context and send 401
			SecurityContextHolder.clearContext();
			this.authenticationEntryPoint.commence(request, response, ex);
		}
	}

	public void setBearerTokenResolver(SimpleBearerTokenResolver bearerTokenResolver) {
		this.bearerTokenResolver = bearerTokenResolver;
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/main/java/simple/security/oauth2/server/resource/web/SimpleBearerTokenAuthenticationEntryPoint.java`

```java
package simple.security.oauth2.server.resource.web;

import java.io.IOException;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.AuthenticationException;
import simple.security.oauth2.core.OAuth2AuthenticationException;
import simple.security.oauth2.core.OAuth2Error;
import simple.security.web.AuthenticationEntryPoint;

/**
 * An {@link AuthenticationEntryPoint} for OAuth2 Resource Servers that sends a
 * {@code 401 Unauthorized} response with a {@code WWW-Authenticate} header.
 *
 * <p>When a request fails authentication (missing or invalid Bearer token), this
 * entry point returns the appropriate HTTP 401 response per RFC 6750 Section 3.
 * If the exception carries an {@link OAuth2Error}, the error code and description
 * are included in the {@code WWW-Authenticate} header.
 *
 * <p>Simplifications vs. real BearerTokenAuthenticationEntryPoint:
 * <ul>
 *   <li>No custom realm name</li>
 *   <li>Simpler WWW-Authenticate header formatting</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.server.resource.web.BearerTokenAuthenticationEntryPoint
 * Source:  oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/BearerTokenAuthenticationEntryPoint.java
 */
public class SimpleBearerTokenAuthenticationEntryPoint implements AuthenticationEntryPoint {

	@Override
	public void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException {

		response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

		String wwwAuthenticate;
		if (authException instanceof OAuth2AuthenticationException oauth2Ex) {
			OAuth2Error error = oauth2Ex.getError();
			StringBuilder sb = new StringBuilder("Bearer");
			sb.append(" error=\"").append(error.getErrorCode()).append("\"");
			if (error.getDescription() != null) {
				sb.append(", error_description=\"").append(error.getDescription()).append("\"");
			}
			if (error.getUri() != null) {
				sb.append(", error_uri=\"").append(error.getUri()).append("\"");
			}
			wwwAuthenticate = sb.toString();
		}
		else {
			wwwAuthenticate = "Bearer";
		}

		response.addHeader("WWW-Authenticate", wwwAuthenticate);
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/oauth2/SimpleOAuth2ResourceServerConfigurer.java`

```java
package simple.security.config.annotation.web.configurers.oauth2;

import simple.security.authentication.AuthenticationManager;
import simple.security.authentication.AuthenticationProvider;
import simple.security.authentication.SimpleProviderManager;
import simple.security.config.Customizer;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.config.annotation.web.configurers.SimpleAbstractHttpConfigurer;
import simple.security.oauth2.jwt.JwtDecoder;
import simple.security.oauth2.server.resource.authentication.SimpleJwtAuthenticationConverter;
import simple.security.oauth2.server.resource.authentication.SimpleJwtAuthenticationProvider;
import simple.security.oauth2.server.resource.web.SimpleBearerTokenAuthenticationEntryPoint;
import simple.security.oauth2.server.resource.web.SimpleBearerTokenAuthenticationFilter;
import simple.security.web.AuthenticationEntryPoint;

import java.util.List;

/**
 * Configures OAuth2 Resource Server support via the {@link SimpleHttpSecurity} DSL.
 *
 * <h2>Usage</h2>
 * <pre>
 * http.oauth2ResourceServer(oauth2 -> oauth2
 *     .jwt(jwt -> jwt
 *         .decoder(myJwtDecoder)
 *         .jwtAuthenticationConverter(myConverter)));
 * </pre>
 *
 * <h2>What it does</h2>
 * <p>During {@code init()}, this configurer registers a
 * {@link SimpleBearerTokenAuthenticationEntryPoint} as the shared
 * {@link AuthenticationEntryPoint} (if none is already registered).
 *
 * <p>During {@code configure()}, it creates a
 * {@link SimpleBearerTokenAuthenticationFilter} wired with an
 * {@link AuthenticationManager} that uses a
 * {@link SimpleJwtAuthenticationProvider} and adds it to the
 * filter chain at position 450 (after HTTP Basic at 400, before
 * Exception Translation at 500).
 *
 * <p>Simplifications vs. real OAuth2ResourceServerConfigurer:
 * <ul>
 *   <li>JWT only -- no opaque token support</li>
 *   <li>No AuthenticationManagerResolver</li>
 *   <li>No BearerTokenRequestMatcher for CSRF exemption</li>
 *   <li>No AccessDeniedHandler configuration</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.oauth2.server.resource.OAuth2ResourceServerConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/oauth2/server/resource/OAuth2ResourceServerConfigurer.java
 */
public class SimpleOAuth2ResourceServerConfigurer
		extends SimpleAbstractHttpConfigurer<SimpleOAuth2ResourceServerConfigurer> {

	private JwtConfigurer jwtConfigurer;

	private AuthenticationEntryPoint authenticationEntryPoint;

	// === Lifecycle ===

	@Override
	public void init(SimpleHttpSecurity builder) throws Exception {
		AuthenticationEntryPoint entryPoint = getAuthenticationEntryPoint();
		// Register as shared entry point if none is set (form login takes priority)
		if (builder.getSharedObject(AuthenticationEntryPoint.class) == null) {
			builder.setSharedObject(AuthenticationEntryPoint.class, entryPoint);
		}
	}

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		if (this.jwtConfigurer == null) {
			throw new IllegalStateException(
					"oauth2ResourceServer requires JWT configuration. "
							+ "Call .jwt(jwt -> jwt.decoder(myDecoder))");
		}

		// Build a dedicated AuthenticationManager for JWT tokens
		AuthenticationProvider jwtProvider = this.jwtConfigurer.getAuthenticationProvider();
		AuthenticationManager jwtAuthManager = new SimpleProviderManager(List.of(jwtProvider));

		AuthenticationEntryPoint entryPoint = getAuthenticationEntryPoint();

		SimpleBearerTokenAuthenticationFilter filter =
				new SimpleBearerTokenAuthenticationFilter(jwtAuthManager, entryPoint);

		builder.addFilter(filter);
	}

	// === DSL methods ===

	/**
	 * Configures JWT-based Bearer token authentication.
	 *
	 * @param jwtCustomizer customizer for the JWT configuration
	 * @return this configurer for chaining
	 */
	public SimpleOAuth2ResourceServerConfigurer jwt(Customizer<JwtConfigurer> jwtCustomizer) {
		if (this.jwtConfigurer == null) {
			this.jwtConfigurer = new JwtConfigurer();
		}
		jwtCustomizer.customize(this.jwtConfigurer);
		return this;
	}

	/**
	 * Sets a custom authentication entry point for Bearer token errors.
	 */
	public SimpleOAuth2ResourceServerConfigurer authenticationEntryPoint(
			AuthenticationEntryPoint entryPoint) {
		this.authenticationEntryPoint = entryPoint;
		return this;
	}

	// === Internal ===

	private AuthenticationEntryPoint getAuthenticationEntryPoint() {
		if (this.authenticationEntryPoint != null) {
			return this.authenticationEntryPoint;
		}
		return new SimpleBearerTokenAuthenticationEntryPoint();
	}

	// === JWT sub-configurer ===

	/**
	 * Configuration for JWT-based authentication within the OAuth2 resource server.
	 *
	 * <pre>
	 * http.oauth2ResourceServer(oauth2 -> oauth2
	 *     .jwt(jwt -> jwt
	 *         .decoder(SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build())
	 *         .jwtAuthenticationConverter(customConverter)));
	 * </pre>
	 */
	public static class JwtConfigurer {

		private JwtDecoder decoder;

		private SimpleJwtAuthenticationConverter jwtAuthenticationConverter;

		/**
		 * Sets the {@link JwtDecoder} used to decode and verify JWTs.
		 */
		public JwtConfigurer decoder(JwtDecoder decoder) {
			this.decoder = decoder;
			return this;
		}

		/**
		 * Sets a custom converter for extracting authorities from the JWT.
		 */
		public JwtConfigurer jwtAuthenticationConverter(SimpleJwtAuthenticationConverter converter) {
			this.jwtAuthenticationConverter = converter;
			return this;
		}

		AuthenticationProvider getAuthenticationProvider() {
			if (this.decoder == null) {
				throw new IllegalStateException(
						"A JwtDecoder must be configured. Call .decoder(myJwtDecoder)");
			}
			SimpleJwtAuthenticationProvider provider = new SimpleJwtAuthenticationProvider(this.decoder);
			if (this.jwtAuthenticationConverter != null) {
				provider.setJwtAuthenticationConverter(this.jwtAuthenticationConverter);
			}
			return provider;
		}

	}

}
```

### [MODIFIED] `simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

Changes:
1. Added `import simple.security.config.annotation.web.configurers.oauth2.SimpleOAuth2ResourceServerConfigurer`
2. Added `SimpleBearerTokenAuthenticationFilter` at position 450 in the filter order map
3. Added `oauth2ResourceServer()` DSL method

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
import simple.security.config.annotation.web.configurers.SimpleAuthorizeHttpRequestsConfigurer;
import simple.security.config.annotation.web.configurers.SimpleCsrfConfigurer;
import simple.security.config.annotation.web.configurers.SimpleExceptionHandlingConfigurer;
import simple.security.config.annotation.web.configurers.SimpleFormLoginConfigurer;
import simple.security.config.annotation.web.configurers.SimpleHttpBasicConfigurer;
import simple.security.config.annotation.web.configurers.SimpleLogoutConfigurer;
import simple.security.config.annotation.web.configurers.SimpleSessionManagementConfigurer;
import simple.security.config.annotation.web.configurers.oauth2.SimpleOAuth2ResourceServerConfigurer;
import simple.security.core.userdetails.UserDetailsService;
import simple.security.web.SecurityFilterChain;
import simple.security.web.SimpleDefaultSecurityFilterChain;
import simple.security.web.util.matcher.RequestMatcher;
import simple.security.web.util.matcher.SimpleAnyRequestMatcher;
import simple.security.web.util.matcher.SimplePathPatternRequestMatcher;

public final class SimpleHttpSecurity implements SecurityBuilder<SecurityFilterChain> {

    private static final Map<String, Integer> FILTER_ORDER_MAP = new LinkedHashMap<>();

    static {
        int order = 100;
        FILTER_ORDER_MAP.put("SimpleSecurityContextHolderFilter", order);      // 100
        order += 100;
        FILTER_ORDER_MAP.put("SimpleCsrfFilter", order);                       // 200
        order += 50;
        FILTER_ORDER_MAP.put("SimpleLogoutFilter", order);                     // 250
        order += 50;
        FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", order); // 300
        order += 100;
        FILTER_ORDER_MAP.put("SimpleBasicAuthenticationFilter", order);        // 400
        order += 50;
        FILTER_ORDER_MAP.put("SimpleBearerTokenAuthenticationFilter", order);  // 450  <-- NEW
        order += 50;
        FILTER_ORDER_MAP.put("SimpleExceptionTranslationFilter", order);       // 500
        order += 100;
        FILTER_ORDER_MAP.put("SimpleAuthorizationFilter", order);              // 600
    }

    // ... (existing fields and methods unchanged) ...

    /**
     * Configures OAuth2 Resource Server support with JWT Bearer Token authentication.
     */
    public SimpleHttpSecurity oauth2ResourceServer(
            Customizer<SimpleOAuth2ResourceServerConfigurer> oauth2ResourceServerCustomizer) {
        oauth2ResourceServerCustomizer.customize(getOrApply(new SimpleOAuth2ResourceServerConfigurer()));
        return this;
    }

    // ... (rest of existing methods unchanged) ...
}
```

### [MODIFIED] `simple-security-config/build.gradle`

Added OAuth2 module dependencies:

```groovy
// simple-security-config -- @EnableWebSecurity, HttpSecurity DSL, configurers
// Mirrors: spring-security-config

dependencies {
    api project(':simple-security-core')
    api project(':simple-security-web')
    implementation project(':simple-security-oauth2-jose')              // <-- NEW
    implementation project(':simple-security-oauth2-resource-server')   // <-- NEW
    implementation 'org.springframework:spring-context:6.2.3'
    implementation 'org.springframework:spring-web:6.2.3'
    implementation 'org.springframework:spring-aop:6.2.3'
    implementation 'org.springframework:spring-expression:6.2.3'

    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    compileOnly 'org.springframework:spring-webmvc:6.2.3'

    testImplementation 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    testImplementation 'org.springframework:spring-test:6.2.3'
    testImplementation 'org.springframework:spring-webmvc:6.2.3'
    testImplementation 'com.nimbusds:nimbus-jose-jwt:10.0.2'
}
```

### [NEW] `simple-security-config/src/test/java/simple/security/config/OAuth2ResourceServerApiTest.java`

```java
package simple.security.config;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.time.Instant;
import java.util.Date;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicReference;

import jakarta.servlet.FilterChain;

import com.nimbusds.jose.JWSAlgorithm;
import com.nimbusds.jose.JWSHeader;
import com.nimbusds.jose.JWSSigner;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockFilterChain;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.core.Authentication;
import simple.security.core.context.SecurityContextHolder;
import simple.security.oauth2.jwt.SimpleNimbusJwtDecoder;
import simple.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import simple.security.oauth2.server.resource.web.SimpleBearerTokenAuthenticationFilter;
import simple.security.web.SecurityFilterChain;
import simple.security.web.SimpleFilterChainProxy;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for Feature 13: OAuth2 Resource Server (JWT).
 *
 * <p>These tests demonstrate how a client configures JWT-based Bearer token
 * authentication via the {@link SimpleHttpSecurity} DSL and sends HTTP requests
 * with JWT tokens through the filter chain.
 */
class OAuth2ResourceServerApiTest {

	private static RSAPublicKey publicKey;
	private static RSAPrivateKey privateKey;

	@BeforeAll
	static void generateKeyPair() throws Exception {
		KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
		gen.initialize(2048);
		KeyPair keyPair = gen.generateKeyPair();
		publicKey = (RSAPublicKey) keyPair.getPublic();
		privateKey = (RSAPrivateKey) keyPair.getPrivate();
	}

	@AfterEach
	void clearSecurityContext() {
		SecurityContextHolder.clearContext();
	}

	// ========== Client-perspective: DSL Configuration ==========

	@Test
	void shouldConfigureOAuth2ResourceServerViaBuilder() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2ResourceServer(oauth2 -> oauth2
				.jwt(jwt -> jwt
						.decoder(SimpleNimbusJwtDecoder.withPublicKey(publicKey).build())));

		SecurityFilterChain chain = http.build();

		// Verify the Bearer token filter is in the chain
		assertThat(chain.getFilters())
				.anyMatch(f -> f instanceof SimpleBearerTokenAuthenticationFilter);
	}

	// ========== Client-perspective: Valid JWT Authentication ==========

	@Test
	void shouldAuthenticateValidJwt() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		String jwt = createSignedJwt("user123", "read write", Instant.now().plusSeconds(3600));

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer " + jwt);
		MockHttpServletResponse response = new MockHttpServletResponse();

		// Capture the authentication inside the chain (before proxy clears context)
		AtomicReference<Authentication> captured = new AtomicReference<>();
		FilterChain capturingChain = (req, res) ->
				captured.set(SecurityContextHolder.getContext().getAuthentication());

		proxy.doFilter(request, response, capturingChain);

		// The request should pass through (filter chain continues)
		assertThat(response.getStatus()).isEqualTo(200);

		// SecurityContext contained the authenticated JWT token during processing
		assertThat(captured.get()).isInstanceOf(JwtAuthenticationToken.class);
		assertThat(captured.get().isAuthenticated()).isTrue();
		assertThat(captured.get().getName()).isEqualTo("user123");
	}

	@Test
	void shouldExtractScopeAuthorities() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		String jwt = createSignedJwt("user123", "read write admin", Instant.now().plusSeconds(3600));

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer " + jwt);
		MockHttpServletResponse response = new MockHttpServletResponse();

		AtomicReference<Authentication> captured = new AtomicReference<>();
		FilterChain capturingChain = (req, res) ->
				captured.set(SecurityContextHolder.getContext().getAuthentication());

		proxy.doFilter(request, response, capturingChain);

		assertThat(captured.get().getAuthorities())
				.extracting("authority")
				.containsExactlyInAnyOrder("SCOPE_read", "SCOPE_write", "SCOPE_admin");
	}

	@Test
	void shouldAccessJwtClaimsFromAuthentication() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		String jwt = createSignedJwt("user123", "read", Instant.now().plusSeconds(3600));

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer " + jwt);
		MockHttpServletResponse response = new MockHttpServletResponse();

		AtomicReference<Authentication> captured = new AtomicReference<>();
		FilterChain capturingChain = (req, res) ->
				captured.set(SecurityContextHolder.getContext().getAuthentication());

		proxy.doFilter(request, response, capturingChain);

		JwtAuthenticationToken jwtAuth = (JwtAuthenticationToken) captured.get();
		assertThat(jwtAuth.getToken().getSubject()).isEqualTo("user123");
		assertThat(jwtAuth.getToken().getIssuer()).isEqualTo("https://idp.example.com");
		assertThat(jwtAuth.getTokenAttributes()).containsKey("sub");
	}

	// ========== Client-perspective: No Token (passthrough) ==========

	@Test
	void shouldPassThroughWhenNoAuthorizationHeader() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		// Request with no Authorization header
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		MockHttpServletResponse response = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();

		proxy.doFilter(request, response, chain);

		// Request should pass through -- no authentication set
		assertThat(response.getStatus()).isEqualTo(200);
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldPassThroughWhenNonBearerAuthorizationHeader() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Basic dXNlcjpwYXNzd29yZA==");
		MockHttpServletResponse response = new MockHttpServletResponse();
		MockFilterChain chain = new MockFilterChain();

		proxy.doFilter(request, response, chain);

		// Non-Bearer token -> filter passes through
		assertThat(response.getStatus()).isEqualTo(200);
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	// ========== Client-perspective: Invalid JWT (401) ==========

	@Test
	void shouldReject401WhenJwtExpired() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		// Create an expired JWT
		String jwt = createSignedJwt("user123", "read", Instant.now().minusSeconds(3600));

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer " + jwt);
		MockHttpServletResponse response = new MockHttpServletResponse();

		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getStatus()).isEqualTo(401);
		assertThat(response.getHeader("WWW-Authenticate")).contains("invalid_token");
		assertThat(SecurityContextHolder.getContext().getAuthentication()).isNull();
	}

	@Test
	void shouldReject401WhenJwtMalformed() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer not.a.valid.jwt");
		MockHttpServletResponse response = new MockHttpServletResponse();

		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getStatus()).isEqualTo(401);
		assertThat(response.getHeader("WWW-Authenticate")).contains("invalid_token");
	}

	@Test
	void shouldReject401WhenJwtSignedWithWrongKey() throws Exception {
		SimpleFilterChainProxy proxy = buildProxy();

		// Create a JWT signed with a DIFFERENT key
		KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
		gen.initialize(2048);
		KeyPair wrongKeyPair = gen.generateKeyPair();
		RSAPrivateKey wrongPrivateKey = (RSAPrivateKey) wrongKeyPair.getPrivate();

		JWSSigner wrongSigner = new RSASSASigner(wrongPrivateKey);
		JWTClaimsSet claims = new JWTClaimsSet.Builder()
				.subject("attacker")
				.issuer("https://evil.example.com")
				.expirationTime(Date.from(Instant.now().plusSeconds(3600)))
				.build();
		SignedJWT signedJWT = new SignedJWT(
				new JWSHeader(JWSAlgorithm.RS256), claims);
		signedJWT.sign(wrongSigner);

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer " + signedJWT.serialize());
		MockHttpServletResponse response = new MockHttpServletResponse();

		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getStatus()).isEqualTo(401);
		assertThat(response.getHeader("WWW-Authenticate")).contains("invalid_token");
	}

	// ========== Client-perspective: DSL with Customization ==========

	@Test
	void shouldSupportCustomAuthenticationEntryPoint() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2ResourceServer(oauth2 -> oauth2
				.jwt(jwt -> jwt.decoder(SimpleNimbusJwtDecoder.withPublicKey(publicKey).build()))
				.authenticationEntryPoint((req, res, ex) -> {
					res.setStatus(401);
					res.setContentType("application/json");
					res.getWriter().write("{\"error\":\"unauthorized\"}");
				}));

		SecurityFilterChain chain = http.build();
		SimpleFilterChainProxy proxy = new SimpleFilterChainProxy(List.of(chain));

		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		request.addHeader("Authorization", "Bearer invalid.jwt.token");
		MockHttpServletResponse response = new MockHttpServletResponse();

		proxy.doFilter(request, response, new MockFilterChain());

		assertThat(response.getStatus()).isEqualTo(401);
		assertThat(response.getContentAsString()).contains("unauthorized");
	}

	// ========== Helpers ==========

	private SimpleFilterChainProxy buildProxy() throws Exception {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2ResourceServer(oauth2 -> oauth2
				.jwt(jwt -> jwt
						.decoder(SimpleNimbusJwtDecoder.withPublicKey(publicKey).build())));

		SecurityFilterChain chain = http.build();
		return new SimpleFilterChainProxy(List.of(chain));
	}

	/**
	 * Creates a signed JWT with the given subject, scope, and expiration.
	 */
	private String createSignedJwt(String subject, String scope, Instant expiration) throws Exception {
		JWSSigner signer = new RSASSASigner(privateKey);

		JWTClaimsSet claims = new JWTClaimsSet.Builder()
				.subject(subject)
				.issuer("https://idp.example.com")
				.audience("my-api")
				.claim("scope", scope)
				.issueTime(Date.from(Instant.now()))
				.expirationTime(Date.from(expiration))
				.jwtID(UUID.randomUUID().toString())
				.build();

		SignedJWT signedJWT = new SignedJWT(
				new JWSHeader(JWSAlgorithm.RS256), claims);
		signedJWT.sign(signer);

		return signedJWT.serialize();
	}

}
```

### [NEW] `simple-security-oauth2-jose/src/test/java/simple/security/oauth2/jwt/SimpleNimbusJwtDecoderTest.java`

```java
package simple.security.oauth2.jwt;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.time.Instant;
import java.util.Date;
import java.util.UUID;

import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;

import com.nimbusds.jose.JWSAlgorithm;
import com.nimbusds.jose.JWSHeader;
import com.nimbusds.jose.JWSSigner;
import com.nimbusds.jose.crypto.MACSigner;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Internal unit tests for {@link SimpleNimbusJwtDecoder}.
 *
 * <p>Tests the decoder's ability to parse, verify, and validate JWTs
 * using RSA and HMAC key types.
 */
class SimpleNimbusJwtDecoderTest {

	private static RSAPublicKey rsaPublicKey;
	private static RSAPrivateKey rsaPrivateKey;

	@BeforeAll
	static void generateKeys() throws Exception {
		KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
		gen.initialize(2048);
		KeyPair keyPair = gen.generateKeyPair();
		rsaPublicKey = (RSAPublicKey) keyPair.getPublic();
		rsaPrivateKey = (RSAPrivateKey) keyPair.getPrivate();
	}

	// ========== RSA (RS256) ==========

	@Test
	void shouldDecodeValidRsaSignedJwt() throws Exception {
		JwtDecoder decoder = SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();

		String token = createRsaSignedJwt("alice", "read write",
				Instant.now().plusSeconds(3600));

		Jwt jwt = decoder.decode(token);

		assertThat(jwt.getSubject()).isEqualTo("alice");
		assertThat(jwt.getIssuer()).isEqualTo("https://idp.example.com");
		assertThat(jwt.getTokenValue()).isEqualTo(token);
		assertThat(jwt.getHeaders()).containsKey("alg");
		assertThat(jwt.<String>getClaim("scope")).isEqualTo("read write");
		assertThat(jwt.getExpiresAt()).isNotNull();
		assertThat(jwt.getIssuedAt()).isNotNull();
	}

	@Test
	void shouldRejectExpiredJwt() throws Exception {
		JwtDecoder decoder = SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();

		String token = createRsaSignedJwt("alice", "read",
				Instant.now().minusSeconds(60));

		assertThatThrownBy(() -> decoder.decode(token))
				.isInstanceOf(BadJwtException.class)
				.hasMessageContaining("expired");
	}

	@Test
	void shouldRejectJwtWithWrongSignature() throws Exception {
		JwtDecoder decoder = SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();

		// Sign with a different key
		KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
		gen.initialize(2048);
		KeyPair wrongPair = gen.generateKeyPair();

		JWSSigner wrongSigner = new RSASSASigner((RSAPrivateKey) wrongPair.getPrivate());
		JWTClaimsSet claims = new JWTClaimsSet.Builder()
				.subject("eve")
				.expirationTime(Date.from(Instant.now().plusSeconds(3600)))
				.build();
		SignedJWT signedJWT = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claims);
		signedJWT.sign(wrongSigner);

		assertThatThrownBy(() -> decoder.decode(signedJWT.serialize()))
				.isInstanceOf(BadJwtException.class);
	}

	@Test
	void shouldRejectMalformedToken() {
		JwtDecoder decoder = SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();

		assertThatThrownBy(() -> decoder.decode("not.a.jwt"))
				.isInstanceOf(BadJwtException.class);
	}

	// ========== HMAC (HS256) ==========

	@Test
	void shouldDecodeHmacSignedJwt() throws Exception {
		// HMAC requires at least 256 bits (32 bytes)
		byte[] secretBytes = new byte[32];
		java.security.SecureRandom.getInstanceStrong().nextBytes(secretBytes);
		SecretKey secretKey = new SecretKeySpec(secretBytes, "HmacSHA256");

		JwtDecoder decoder = SimpleNimbusJwtDecoder.withSecretKey(secretKey).build();

		// Sign with HMAC
		JWSSigner signer = new MACSigner(secretKey);
		JWTClaimsSet claims = new JWTClaimsSet.Builder()
				.subject("service-account")
				.claim("scope", "internal")
				.expirationTime(Date.from(Instant.now().plusSeconds(3600)))
				.issueTime(new Date())
				.build();
		SignedJWT signedJWT = new SignedJWT(new JWSHeader(JWSAlgorithm.HS256), claims);
		signedJWT.sign(signer);

		Jwt jwt = decoder.decode(signedJWT.serialize());

		assertThat(jwt.getSubject()).isEqualTo("service-account");
		assertThat(jwt.<String>getClaim("scope")).isEqualTo("internal");
	}

	// ========== Jwt model ==========

	@Test
	void shouldBuildJwtViaBuilder() {
		Instant now = Instant.now();
		Jwt jwt = Jwt.withTokenValue("token-value")
				.header("alg", "RS256")
				.subject("bob")
				.issuer("https://issuer.example.com")
				.issuedAt(now)
				.expiresAt(now.plusSeconds(3600))
				.claim("custom", "value")
				.build();

		assertThat(jwt.getTokenValue()).isEqualTo("token-value");
		assertThat(jwt.getSubject()).isEqualTo("bob");
		assertThat(jwt.getIssuer()).isEqualTo("https://issuer.example.com");
		assertThat(jwt.getIssuedAt()).isEqualTo(now);
		assertThat(jwt.getExpiresAt()).isEqualTo(now.plusSeconds(3600));
		assertThat(jwt.<String>getClaim("custom")).isEqualTo("value");
		assertThat(jwt.getHeaders()).containsEntry("alg", "RS256");
	}

	@Test
	void shouldReturnUnmodifiableMaps() {
		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.claim("sub", "user")
				.build();

		assertThatThrownBy(() -> jwt.getHeaders().put("new", "value"))
				.isInstanceOf(UnsupportedOperationException.class);
		assertThatThrownBy(() -> jwt.getClaims().put("new", "value"))
				.isInstanceOf(UnsupportedOperationException.class);
	}

	// ========== Helpers ==========

	private String createRsaSignedJwt(String subject, String scope, Instant expiration) throws Exception {
		JWSSigner signer = new RSASSASigner(rsaPrivateKey);

		JWTClaimsSet claims = new JWTClaimsSet.Builder()
				.subject(subject)
				.issuer("https://idp.example.com")
				.claim("scope", scope)
				.issueTime(new Date())
				.expirationTime(Date.from(expiration))
				.jwtID(UUID.randomUUID().toString())
				.build();

		SignedJWT signedJWT = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claims);
		signedJWT.sign(signer);
		return signedJWT.serialize();
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/test/java/simple/security/oauth2/server/resource/authentication/SimpleJwtAuthenticationConverterTest.java`

```java
package simple.security.oauth2.server.resource.authentication;

import java.time.Instant;
import java.util.Collection;
import java.util.List;

import org.junit.jupiter.api.Test;

import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.oauth2.jwt.Jwt;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Internal tests for {@link SimpleJwtAuthenticationConverter}.
 *
 * <p>Tests authority extraction from JWT scope/scp claims.
 */
class SimpleJwtAuthenticationConverterTest {

	private final SimpleJwtAuthenticationConverter converter = new SimpleJwtAuthenticationConverter();

	@Test
	void shouldExtractAuthoritiesFromScopeClaim_WhenSpaceDelimited() {
		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.subject("user")
				.claim("scope", "read write admin")
				.expiresAt(Instant.now().plusSeconds(3600))
				.build();

		JwtAuthenticationToken result = converter.convert(jwt);

		assertThat(result.getAuthorities())
				.extracting("authority")
				.containsExactlyInAnyOrder("SCOPE_read", "SCOPE_write", "SCOPE_admin");
	}

	@Test
	void shouldExtractAuthoritiesFromScpClaim_WhenScopeNotPresent() {
		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.subject("user")
				.claim("scp", List.of("read", "write"))
				.expiresAt(Instant.now().plusSeconds(3600))
				.build();

		JwtAuthenticationToken result = converter.convert(jwt);

		assertThat(result.getAuthorities())
				.extracting("authority")
				.containsExactlyInAnyOrder("SCOPE_read", "SCOPE_write");
	}

	@Test
	void shouldReturnEmptyAuthorities_WhenNoScopeClaim() {
		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.subject("user")
				.expiresAt(Instant.now().plusSeconds(3600))
				.build();

		JwtAuthenticationToken result = converter.convert(jwt);

		assertThat(result.getAuthorities()).isEmpty();
	}

	@Test
	void shouldUseSubjectAsName() {
		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.subject("alice")
				.expiresAt(Instant.now().plusSeconds(3600))
				.build();

		JwtAuthenticationToken result = converter.convert(jwt);

		assertThat(result.getName()).isEqualTo("alice");
	}

	@Test
	void shouldBeAuthenticated() {
		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.subject("user")
				.expiresAt(Instant.now().plusSeconds(3600))
				.build();

		JwtAuthenticationToken result = converter.convert(jwt);

		assertThat(result.isAuthenticated()).isTrue();
		assertThat(result.getPrincipal()).isSameAs(jwt);
		assertThat(result.getCredentials()).isEqualTo("token");
	}

	@Test
	void shouldSupportCustomAuthoritiesConverter() {
		SimpleJwtAuthenticationConverter customConverter = new SimpleJwtAuthenticationConverter();
		customConverter.setJwtGrantedAuthoritiesConverter(jwt -> {
			List<String> roles = jwt.getClaim("roles");
			if (roles == null) return List.of();
			return roles.stream()
					.map(role -> (GrantedAuthority) new SimpleGrantedAuthority("ROLE_" + role))
					.toList();
		});

		Jwt jwt = Jwt.withTokenValue("token")
				.header("alg", "RS256")
				.subject("admin-user")
				.claim("roles", List.of("ADMIN", "USER"))
				.expiresAt(Instant.now().plusSeconds(3600))
				.build();

		JwtAuthenticationToken result = customConverter.convert(jwt);

		assertThat(result.getAuthorities())
				.extracting("authority")
				.containsExactlyInAnyOrder("ROLE_ADMIN", "ROLE_USER");
	}

}
```

### [NEW] `simple-security-oauth2-resource-server/src/test/java/simple/security/oauth2/server/resource/web/SimpleBearerTokenResolverTest.java`

```java
package simple.security.oauth2.server.resource.web;

import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Internal tests for {@link SimpleBearerTokenResolver}.
 *
 * <p>Tests RFC 6750 Bearer token extraction from the Authorization header.
 */
class SimpleBearerTokenResolverTest {

	private final SimpleBearerTokenResolver resolver = new SimpleBearerTokenResolver();

	@Test
	void shouldResolveBearerToken_WhenValidAuthorizationHeader() {
		MockHttpServletRequest request = new MockHttpServletRequest();
		request.addHeader("Authorization", "Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.signature");

		String token = resolver.resolve(request);

		assertThat(token).isEqualTo("eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.signature");
	}

	@Test
	void shouldReturnNull_WhenNoAuthorizationHeader() {
		MockHttpServletRequest request = new MockHttpServletRequest();

		String token = resolver.resolve(request);

		assertThat(token).isNull();
	}

	@Test
	void shouldReturnNull_WhenBasicAuthorizationHeader() {
		MockHttpServletRequest request = new MockHttpServletRequest();
		request.addHeader("Authorization", "Basic dXNlcjpwYXNzd29yZA==");

		String token = resolver.resolve(request);

		assertThat(token).isNull();
	}

	@Test
	void shouldBeCaseInsensitive_WhenBearerPrefix() {
		MockHttpServletRequest request = new MockHttpServletRequest();
		request.addHeader("Authorization", "bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.sig");

		String token = resolver.resolve(request);

		assertThat(token).isEqualTo("eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.sig");
	}

	@Test
	void shouldReturnNull_WhenBearerTokenEmpty() {
		MockHttpServletRequest request = new MockHttpServletRequest();
		request.addHeader("Authorization", "Bearer ");

		String token = resolver.resolve(request);

		// "Bearer " with a trailing space but no token doesn't match the pattern
		assertThat(token).isNull();
	}

	@Test
	void shouldSupportTokenWithDashes() {
		MockHttpServletRequest request = new MockHttpServletRequest();
		request.addHeader("Authorization", "Bearer token-with-dashes_and.dots+plus/slash==");

		String token = resolver.resolve(request);

		assertThat(token).isEqualTo("token-with-dashes_and.dots+plus/slash==");
	}

}
```
