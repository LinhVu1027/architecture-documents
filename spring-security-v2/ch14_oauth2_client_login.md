# Chapter 14: OAuth2 Client & Login

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Configure OAuth2 Login ("Login with Google"):
ClientRegistration google = ClientRegistration.withRegistrationId("google")
        .clientId("google-client-id")
        .clientSecret("google-secret")
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
        .scope("openid", "profile", "email")
        .authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
        .tokenUri("https://oauth2.googleapis.com/token")
        .userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
        .userNameAttributeName("sub")
        .build();

SimpleInMemoryClientRegistrationRepository repository =
        new SimpleInMemoryClientRegistrationRepository(google);

SimpleHttpSecurity http = new SimpleHttpSecurity();
http.oauth2Login(oauth2 -> oauth2
        .clientRegistrationRepository(repository)
        .defaultSuccessUrl("/home")
        .userInfoEndpoint(userInfo -> userInfo
                .userService(customOAuth2UserService)));
SecurityFilterChain chain = http.build();

// GET /oauth2/authorization/google
// -> looks up "google" ClientRegistration
// -> generates random state parameter
// -> saves OAuth2AuthorizationRequest in session
// -> redirects browser to https://accounts.google.com/o/oauth2/v2/auth?response_type=code&client_id=...&state=...

// GET /login/oauth2/code/google?code=abc123&state=xyz
// -> retrieves saved authorization request from session
// -> validates state parameter matches (CSRF protection)
// -> exchanges authorization code for access token (POST to token endpoint)
// -> loads user info from UserInfo endpoint (GET with Bearer token)
// -> creates OAuth2AuthenticationToken with OAuth2User principal
// -> stores in SecurityContext, redirects to success URL
```

Questions to guide your thinking:
- Why does the OAuth2 login flow need *two* separate filters (redirect filter and login filter) instead of one? (Hint: the redirect filter initiates the flow, the login filter completes it.)
- Why does `OAuth2LoginAuthenticationToken` have two constructors -- one for "unauthenticated" and one for "authenticated"? How does this parallel `UsernamePasswordAuthenticationToken` from Feature 2?
- Why is the `state` parameter critical for security? What attack does it prevent?
- Why does the login filter extend `SimpleAbstractAuthenticationProcessingFilter` while the redirect filter implements `Filter` directly?
- How does the `OAuth2UserService` interface enable custom user loading (e.g., enriching user attributes from a local database)?

---

## The API Contract

Feature 14 introduces the OAuth2 Authorization Code login flow -- "Login with Google/GitHub/etc."
It is exposed as a convenience method on `SimpleHttpSecurity` backed by a configurer that
produces two filters.

### OAuth2 Login DSL

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public SimpleHttpSecurity oauth2Login(Customizer<SimpleOAuth2LoginConfigurer> oauth2LoginCustomizer) {
    oauth2LoginCustomizer.customize(getOrApply(new SimpleOAuth2LoginConfigurer()));
    return this;
}
```

The configurer exposes these DSL methods:

```java
// simple/security/config/annotation/web/configurers/oauth2/client/SimpleOAuth2LoginConfigurer.java
public class SimpleOAuth2LoginConfigurer extends SimpleAbstractHttpConfigurer<SimpleOAuth2LoginConfigurer> {

    public SimpleOAuth2LoginConfigurer clientRegistrationRepository(ClientRegistrationRepository repo) { ... }
    public SimpleOAuth2LoginConfigurer loginPage(String loginPage) { ... }
    public SimpleOAuth2LoginConfigurer defaultSuccessUrl(String defaultSuccessUrl) { ... }
    public SimpleOAuth2LoginConfigurer failureUrl(String failureUrl) { ... }
    public SimpleOAuth2LoginConfigurer successHandler(AuthenticationSuccessHandler handler) { ... }
    public SimpleOAuth2LoginConfigurer failureHandler(AuthenticationFailureHandler handler) { ... }
    public SimpleOAuth2LoginConfigurer authenticationEntryPoint(AuthenticationEntryPoint ep) { ... }
    public SimpleOAuth2LoginConfigurer userInfoEndpoint(Customizer<UserInfoEndpointConfig> customizer) { ... }

    // Inner class for UserInfo endpoint configuration
    public static class UserInfoEndpointConfig {
        public UserInfoEndpointConfig userService(OAuth2UserService userService) { ... }
    }
}
```

### The nested DSL pattern

The `oauth2Login()` DSL uses a **nested configurer** pattern. The outer configurer
(`SimpleOAuth2LoginConfigurer`) handles the login lifecycle (success/failure handlers,
entry point, filter wiring). The inner configurer (`UserInfoEndpointConfig`) handles
how user attributes are loaded from the provider. This mirrors the real framework's
design where OIDC user service would be a sibling alongside the plain OAuth2 user service:

```java
http.oauth2Login(oauth2 -> oauth2
    .clientRegistrationRepository(repository)
    .loginPage("/custom-login")
    .defaultSuccessUrl("/home")
    .userInfoEndpoint(userInfo -> userInfo
        .userService(customOAuth2UserService)));
```

### Key types

The OAuth2 client login feature spans two new modules and introduces several key types:

```java
// Client registration: all the details about an OAuth2 provider
ClientRegistration google = ClientRegistration.withRegistrationId("google")
        .clientId("google-client-id")
        .clientSecret("google-secret")
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
        .scope("openid", "profile", "email")
        .authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
        .tokenUri("https://oauth2.googleapis.com/token")
        .userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
        .userNameAttributeName("sub")
        .build();

// Repository: looks up registrations by ID
ClientRegistrationRepository repository = new SimpleInMemoryClientRegistrationRepository(google);

// Authorization exchange: pairs the request sent to the provider with the response received
OAuth2AuthorizationRequest request = OAuth2AuthorizationRequest.create(...);
OAuth2AuthorizationResponse response = OAuth2AuthorizationResponse.success("code123", "state123");
OAuth2AuthorizationExchange exchange = new OAuth2AuthorizationExchange(request, response);

// Authentication tokens: unauthenticated (during flow) and authenticated (final result)
OAuth2LoginAuthenticationToken unauthenticated = new OAuth2LoginAuthenticationToken(registration, exchange);
OAuth2AuthenticationToken authenticated = new OAuth2AuthenticationToken(oauth2User, authorities, "google");
```

---

## Client Usage & Tests

The tests define the behavioral contract from a client's perspective. Each test builds
a filter chain and verifies the correct behavior at various points in the OAuth2 login flow.

### Configuring OAuth2 Login

```java
@Test
void shouldBuildFilterChainWithOAuth2Filters() throws Exception {
    ClientRegistration google = createGoogleRegistration();
    SimpleInMemoryClientRegistrationRepository repository =
            new SimpleInMemoryClientRegistrationRepository(google);

    OAuth2UserService mockUserService = userRequest ->
            new DefaultOAuth2User(
                    List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
                    Map.of("sub", "user123", "name", "John"),
                    "sub");

    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.oauth2Login(oauth2 -> oauth2
            .clientRegistrationRepository(repository)
            .defaultSuccessUrl("/home")
            .userInfoEndpoint(userInfo -> userInfo
                    .userService(mockUserService)));

    SecurityFilterChain chain = http.build();

    assertNotNull(chain);
    List<Filter> filters = chain.getFilters();

    // Verify that both OAuth2 filters are present
    boolean hasRedirectFilter = filters.stream()
            .anyMatch(f -> f.getClass().getSimpleName().contains("OAuth2AuthorizationRequestRedirect"));
    boolean hasLoginFilter = filters.stream()
            .anyMatch(f -> f.getClass().getSimpleName().contains("OAuth2LoginAuthentication"));
    assertTrue(hasRedirectFilter, "Should contain OAuth2 redirect filter");
    assertTrue(hasLoginFilter, "Should contain OAuth2 login filter");
}
```

### Redirecting to the authorization endpoint

```java
@Test
void shouldRedirectToAuthorizationEndpoint_WhenAccessingOAuth2AuthorizationUrl()
        throws Exception {
    // ... build chain with OAuth2 login configured ...

    MockHttpServletRequest request = new MockHttpServletRequest("GET",
            "/oauth2/authorization/google");
    request.setServerName("localhost");
    request.setServerPort(8080);
    request.setScheme("http");
    MockHttpServletResponse response = new MockHttpServletResponse();

    // Run through the filter chain
    for (Filter filter : chain.getFilters()) {
        if (response.isCommitted()) break;
        filter.doFilter(request, response, (req, res) -> {});
    }

    assertNotNull(response.getRedirectedUrl());
    assertTrue(response.getRedirectedUrl().startsWith(
            "https://accounts.google.com/o/oauth2/v2/auth?"));
}
```

### Handling the OAuth2 callback

```java
@Test
void shouldAuthenticate_WhenCallbackContainsCodeAndState() throws Exception {
    // Simulate a saved authorization request (as if redirect filter already ran)
    String savedState = "test-state-123";
    OAuth2AuthorizationRequest savedRequest = OAuth2AuthorizationRequest.create(
            "https://accounts.google.com/o/oauth2/v2/auth",
            "google-client-id",
            "http://localhost/login/oauth2/code/google",
            Set.of("openid", "profile"),
            savedState,
            "google");

    MockHttpServletRequest request = new MockHttpServletRequest("GET",
            "/login/oauth2/code/google");
    request.setParameter("code", "auth-code-xyz");
    request.setParameter("state", savedState);
    request.getSession().setAttribute(
            SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR,
            savedRequest);

    MockHttpServletResponse response = new MockHttpServletResponse();

    // Mock the authentication manager to return an authenticated token
    DefaultOAuth2User oauth2User = new DefaultOAuth2User(
            List.of(new SimpleGrantedAuthority("OAUTH2_USER"),
                    new SimpleGrantedAuthority("SCOPE_openid")),
            Map.of("sub", "user123", "name", "John Doe"),
            "sub");

    when(authenticationManager.authenticate(any())).thenAnswer(invocation -> {
        OAuth2LoginAuthenticationToken inputToken = invocation.getArgument(0);
        return new OAuth2LoginAuthenticationToken(
                inputToken.getClientRegistration(),
                inputToken.getAuthorizationExchange(),
                oauth2User,
                List.of(new SimpleGrantedAuthority("OAUTH2_USER"),
                        new SimpleGrantedAuthority("SCOPE_openid")),
                new OAuth2AccessToken("access-token-value", Set.of("openid", "profile")));
    });

    Authentication result = filter.attemptAuthentication(request, response);

    // Returns an OAuth2AuthenticationToken (the final principal)
    assertInstanceOf(OAuth2AuthenticationToken.class, result);
    OAuth2AuthenticationToken authToken = (OAuth2AuthenticationToken) result;
    assertEquals("user123", authToken.getName());
    assertEquals("google", authToken.getAuthorizedClientRegistrationId());
    assertTrue(authToken.isAuthenticated());

    // Authorization request removed from session (one-time use)
    assertNull(request.getSession().getAttribute(
            SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR));
}
```

### State mismatch detection

```java
@Test
void shouldRejectCallback_WhenStateMismatch() throws Exception {
    OAuth2AuthorizationRequest savedRequest = OAuth2AuthorizationRequest.create(
            "https://accounts.google.com/o/oauth2/v2/auth",
            "google-client-id",
            "http://localhost/login/oauth2/code/google",
            Set.of("openid"),
            "expected-state",
            "google");

    MockHttpServletRequest request = new MockHttpServletRequest("GET",
            "/login/oauth2/code/google");
    request.setParameter("code", "auth-code-xyz");
    request.setParameter("state", "wrong-state");
    request.getSession().setAttribute(
            SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR,
            savedRequest);

    MockHttpServletResponse response = new MockHttpServletResponse();

    assertThrows(OAuth2AuthenticationException.class,
            () -> filter.attemptAuthentication(request, response));
}
```

### Error response from provider

```java
@Test
void shouldHandleErrorResponseFromProvider() throws Exception {
    // ... save authorization request in session ...

    MockHttpServletRequest request = new MockHttpServletRequest("GET",
            "/login/oauth2/code/google");
    request.setParameter("error", "access_denied");
    request.setParameter("error_description", "User denied access");
    request.setParameter("state", "test-state");
    // ... set session attribute ...

    MockHttpServletResponse response = new MockHttpServletResponse();

    when(authenticationManager.authenticate(any())).thenAnswer(invocation -> {
        OAuth2LoginAuthenticationToken token = invocation.getArgument(0);
        OAuth2AuthorizationResponse authResponse =
                token.getAuthorizationExchange().getAuthorizationResponse();
        assertFalse(authResponse.isSuccess());
        assertEquals("access_denied", authResponse.getError());
        throw new OAuth2AuthenticationException(
                new OAuth2Error("access_denied", "User denied access", null));
    });

    assertThrows(OAuth2AuthenticationException.class,
            () -> filter.attemptAuthentication(request, response));
}
```

---

## Implementing the Call Chain

### Call chain recap

```
Client calls http.oauth2Login(oauth2 -> oauth2.clientRegistrationRepository(repo))
  |
  +---> SimpleHttpSecurity.oauth2Login()
  |       getOrApply(new SimpleOAuth2LoginConfigurer()) -- register configurer
  |       customizer.customize(configurer) -- user's lambda calls .clientRegistrationRepository(...)
  |
  +---> Client calls http.build()
  |       |
  |       +---> Phase 1: INITIALIZING
  |       |       SimpleOAuth2LoginConfigurer.init(http)
  |       |         Resolves failureUrl default (loginPage + "?error")
  |       |         Creates SimpleLoginUrlAuthenticationEntryPoint
  |       |         Registers as shared entry point
  |       |         Creates OAuth2LoginAuthenticationProvider(userService)
  |       |         Registers via builder.authenticationProvider(provider)
  |       |
  |       +---> Phase 2: beforeConfigure()
  |       |       Resolves AuthenticationManager -> setSharedObject (if providers exist)
  |       |
  |       +---> Phase 3: CONFIGURING
  |       |       SimpleOAuth2LoginConfigurer.configure(http)
  |       |         Resolves ClientRegistrationRepository (from configurer or shared objects)
  |       |         Creates SimpleOAuth2AuthorizationRequestRedirectFilter(repository)
  |       |           addFilterBefore(redirectFilter, UsernamePasswordAuthFilter) -> position 299
  |       |         Creates SimpleOAuth2LoginAuthenticationFilter(repository)
  |       |           Builds dedicated AuthenticationManager with OAuth2LoginAuthenticationProvider
  |       |           Wires success/failure handlers
  |       |           Wires SecurityContextRepository
  |       |           addFilterAfter(loginFilter, RedirectFilter) -> position 300 (approx)
  |       |
  |       +---> Phase 4: performBuild()
  |               Sort filters: [..., 299: RedirectFilter, 300: LoginFilter, ...]
  |               Wrap in SimpleDefaultSecurityFilterChain
  |
  +---> At request time (Step 1 -- Initiate):
  |       GET /oauth2/authorization/google
  |         SimpleOAuth2AuthorizationRequestRedirectFilter.doFilter()
  |           resolveRegistrationId("google")
  |           clientRegistrationRepository.findByRegistrationId("google")
  |           Generate random state (UUID)
  |           resolveRedirectUri({baseUrl}/login/oauth2/code/{registrationId})
  |             -> http://localhost:8080/login/oauth2/code/google
  |           OAuth2AuthorizationRequest.create(...)
  |           session.setAttribute(AUTHORIZATION_REQUEST_ATTR, request)
  |           response.sendRedirect(authorizationRequestUri)
  |             -> 302 to https://accounts.google.com/o/oauth2/v2/auth?
  |                response_type=code&client_id=google-client-id&redirect_uri=...&state=...
  |
  +---> At request time (Step 2 -- Callback):
          GET /login/oauth2/code/google?code=abc123&state=xyz
            SimpleOAuth2LoginAuthenticationFilter.attemptAuthentication()
              Extract registrationId from path: "google"
              Look up ClientRegistration for "google"
              Retrieve saved OAuth2AuthorizationRequest from session
              Remove from session (one-time use)
              Validate state matches (CSRF protection)
              Build OAuth2AuthorizationResponse from query params
              Build OAuth2AuthorizationExchange(request, response)
              Create unauthenticated OAuth2LoginAuthenticationToken(registration, exchange)
              AuthenticationManager.authenticate(token)
                OAuth2LoginAuthenticationProvider.authenticate()
                  Validate authorization response is success
                  exchangeCodeForToken(registration, code, redirectUri)
                    POST to token endpoint with code + client credentials
                    Parse access_token from JSON response
                  userService.loadUser(new OAuth2UserRequest(registration, accessToken))
                    GET userInfoUri with Bearer accessToken
                    Parse JSON attributes into DefaultOAuth2User
                  Return authenticated OAuth2LoginAuthenticationToken(user, authorities, token)
              Convert to OAuth2AuthenticationToken(oauth2User, authorities, "google")
              Parent class stores in SecurityContext
              Redirect to defaultSuccessUrl ("/home")
```

### 14a. Core types: AuthorizationGrantType and OAuth2AccessToken

Before building the client components, we need foundational types from the `simple-security-oauth2-core`
module.

**`AuthorizationGrantType`** is a simple value object wrapping the grant type string.
Static constants are provided for common grant types:

```java
public final class AuthorizationGrantType {

    public static final AuthorizationGrantType AUTHORIZATION_CODE =
            new AuthorizationGrantType("authorization_code");

    public static final AuthorizationGrantType CLIENT_CREDENTIALS =
            new AuthorizationGrantType("client_credentials");

    private final String value;

    public AuthorizationGrantType(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("value cannot be empty");
        }
        this.value = value;
    }
    // equals, hashCode, toString...
}
```

**`OAuth2AccessToken`** holds the access token value and associated scopes. This is
returned after the authorization code exchange and used to call the UserInfo endpoint:

```java
public class OAuth2AccessToken {
    private final String tokenValue;
    private final Set<String> scopes;

    public OAuth2AccessToken(String tokenValue, Set<String> scopes) {
        // validates tokenValue is not blank
        this.tokenValue = tokenValue;
        this.scopes = (scopes != null) ? Collections.unmodifiableSet(scopes) : Collections.emptySet();
    }
}
```

**Why is `AuthorizationGrantType` a value object instead of an enum?** Custom grant types
exist in practice (e.g., `urn:ietf:params:oauth:grant-type:jwt-bearer`). A `final class`
with `equals`/`hashCode` based on the string value allows custom types while still providing
well-known constants.

### 14b. The user model: OAuth2User and DefaultOAuth2User

**`OAuth2User`** is the interface representing a user authenticated via OAuth2. It combines
user attributes (from the UserInfo endpoint) with granted authorities:

```java
public interface OAuth2User {
    String getName();
    Map<String, Object> getAttributes();
    Collection<? extends GrantedAuthority> getAuthorities();
}
```

**`DefaultOAuth2User`** is the default implementation. The key design decision is the
`nameAttributeKey` -- different providers use different attribute names for the user's
identifier (GitHub uses `"login"`, Google uses `"sub"`), so the key is configurable:

```java
public class DefaultOAuth2User implements OAuth2User {
    private final Collection<GrantedAuthority> authorities;
    private final Map<String, Object> attributes;
    private final String nameAttributeKey;

    public DefaultOAuth2User(Collection<? extends GrantedAuthority> authorities,
            Map<String, Object> attributes, String nameAttributeKey) {
        // validates attributes not empty, nameAttributeKey exists in attributes
        this.authorities = Collections.unmodifiableList(new ArrayList<>(authorities));
        this.attributes = Collections.unmodifiableMap(attributes);
        this.nameAttributeKey = nameAttributeKey;
    }

    @Override
    public String getName() {
        return this.attributes.get(this.nameAttributeKey).toString();
    }
}
```

**Why is `nameAttributeKey` required at construction time?** Without it, `getName()` would
have no way to know which attribute is the user's identifier. Making it a constructor
parameter forces every `DefaultOAuth2User` to have a well-defined identity, which the
rest of the framework relies on (e.g., for logging, session tracking, display names).

### 14c. Client registration: ClientRegistration and its Builder

`ClientRegistration` contains all the information needed to interact with an OAuth2 provider:
client credentials, endpoint URIs, scopes, and a redirect URI template.

```java
public final class ClientRegistration {
    private final String registrationId;
    private final String clientId;
    private final String clientSecret;
    private final AuthorizationGrantType authorizationGrantType;
    private final String redirectUri;
    private final Set<String> scopes;
    private final String authorizationUri;
    private final String tokenUri;
    private final String userInfoUri;
    private final String userNameAttributeName;
    private final String clientName;
}
```

The builder enforces invariants specific to each grant type:

```java
public ClientRegistration build() {
    if (this.clientId == null || this.clientId.isBlank()) {
        throw new IllegalArgumentException("clientId is required");
    }
    if (AuthorizationGrantType.AUTHORIZATION_CODE.equals(this.authorizationGrantType)) {
        if (this.redirectUri == null || this.redirectUri.isBlank()) {
            throw new IllegalArgumentException(
                    "redirectUri is required for authorization_code grant type");
        }
        if (this.authorizationUri == null || this.authorizationUri.isBlank()) {
            throw new IllegalArgumentException(
                    "authorizationUri is required for authorization_code grant type");
        }
        // tokenUri also required for authorization_code...
    }
    return new ClientRegistration(this);
}
```

**Why does the builder flatten provider details?** In the real framework, fields like
`authorizationUri`, `tokenUri`, and `userInfoUri` are nested inside a `ProviderDetails`
inner class. This simplified version flattens them directly onto `ClientRegistration` for
easier construction and access. The tradeoff is that adding a new provider field requires
adding it to the main class, but the gain is much simpler code.

**Why is `redirectUri` a template with `{baseUrl}` and `{registrationId}`?** The base URL
varies between environments (localhost:8080 in development, myapp.com in production). Using
template variables lets you define one registration that works everywhere -- the redirect
filter resolves the templates at request time.

### 14d. The repository: ClientRegistrationRepository and SimpleInMemoryClientRegistrationRepository

**`ClientRegistrationRepository`** is the OAuth2 analog of `UserDetailsService` -- a
single-method lookup:

```java
public interface ClientRegistrationRepository {
    ClientRegistration findByRegistrationId(String registrationId);
}
```

**`SimpleInMemoryClientRegistrationRepository`** stores registrations in a `LinkedHashMap`
and implements `Iterable<ClientRegistration>` so configurers can enumerate all registrations:

```java
public final class SimpleInMemoryClientRegistrationRepository
        implements ClientRegistrationRepository, Iterable<ClientRegistration> {

    private final Map<String, ClientRegistration> registrations;

    public SimpleInMemoryClientRegistrationRepository(ClientRegistration... registrations) {
        this(Arrays.asList(registrations));
    }

    public SimpleInMemoryClientRegistrationRepository(List<ClientRegistration> registrations) {
        Map<String, ClientRegistration> map = new LinkedHashMap<>();
        for (ClientRegistration registration : registrations) {
            if (map.containsKey(registration.getRegistrationId())) {
                throw new IllegalStateException(
                        "Duplicate registration ID: " + registration.getRegistrationId());
            }
            map.put(registration.getRegistrationId(), registration);
        }
        this.registrations = Collections.unmodifiableMap(map);
    }
}
```

**Why `Iterable`?** The login page needs to list all configured OAuth2 providers as links.
`Iterable` gives the configurer a way to enumerate them without exposing the internal map.

### 14e. The authorization exchange model

Three types model the OAuth2 authorization exchange:

**`OAuth2AuthorizationRequest`** -- created before redirecting to the provider. It builds
the full authorization URL with query parameters:

```java
public final class OAuth2AuthorizationRequest {
    private final String authorizationUri;
    private final String clientId;
    private final String redirectUri;
    private final Set<String> scopes;
    private final String state;
    private final String registrationId;
    private final String authorizationRequestUri;  // computed

    private String buildAuthorizationRequestUri() {
        StringBuilder sb = new StringBuilder(this.authorizationUri);
        sb.append("?response_type=code");
        sb.append("&client_id=").append(urlEncode(this.clientId));
        if (this.redirectUri != null) {
            sb.append("&redirect_uri=").append(urlEncode(this.redirectUri));
        }
        if (!this.scopes.isEmpty()) {
            sb.append("&scope=").append(urlEncode(String.join(" ", this.scopes)));
        }
        if (this.state != null) {
            sb.append("&state=").append(urlEncode(this.state));
        }
        return sb.toString();
    }
}
```

**`OAuth2AuthorizationResponse`** -- parsed from the provider's callback URL. Uses factory
methods for success vs. error responses:

```java
public final class OAuth2AuthorizationResponse {
    private final String code;
    private final String state;
    private final String error;
    private final String errorDescription;

    public static OAuth2AuthorizationResponse success(String code, String state) { ... }
    public static OAuth2AuthorizationResponse error(String error, String errorDescription, String state) { ... }

    public boolean isSuccess() {
        return this.code != null && this.error == null;
    }
}
```

**`OAuth2AuthorizationExchange`** -- pairs the request and response together. This is
passed to the authentication provider so it has both the original request (with state
and redirect URI) and the response (with the authorization code):

```java
public final class OAuth2AuthorizationExchange {
    private final OAuth2AuthorizationRequest authorizationRequest;
    private final OAuth2AuthorizationResponse authorizationResponse;
}
```

**Why pair request and response into an exchange?** The provider needs both. The request
contains the `state` (for validation), `redirectUri` (for the token exchange), and
`registrationId`. The response contains the `code`. Bundling them avoids passing many
individual parameters through the call chain.

### 14f. The redirect filter: SimpleOAuth2AuthorizationRequestRedirectFilter

This filter initiates the OAuth2 flow by redirecting the user's browser to the provider's
authorization endpoint. It matches requests to `/oauth2/authorization/{registrationId}`:

```java
public class SimpleOAuth2AuthorizationRequestRedirectFilter implements Filter {

    public static final String DEFAULT_AUTHORIZATION_REQUEST_BASE_URI = "/oauth2/authorization";

    static final String AUTHORIZATION_REQUEST_ATTR =
            "simple.security.oauth2.client.web.OAuth2AuthorizationRequest";

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String registrationId = resolveRegistrationId(request);
        if (registrationId == null) {
            chain.doFilter(request, response);  // not our request, pass through
            return;
        }

        ClientRegistration clientRegistration =
                this.clientRegistrationRepository.findByRegistrationId(registrationId);
        if (clientRegistration == null) {
            response.sendError(400, "Unknown client registration: " + registrationId);
            return;
        }

        // Generate state and resolve redirect URI
        String state = UUID.randomUUID().toString();
        String redirectUri = resolveRedirectUri(request, clientRegistration);

        // Build the authorization request
        OAuth2AuthorizationRequest authorizationRequest = OAuth2AuthorizationRequest.create(
                clientRegistration.getAuthorizationUri(),
                clientRegistration.getClientId(),
                redirectUri,
                clientRegistration.getScopes(),
                state,
                registrationId);

        // Save in session for the callback filter to retrieve
        request.getSession().setAttribute(AUTHORIZATION_REQUEST_ATTR, authorizationRequest);

        // Redirect to the authorization server
        response.sendRedirect(authorizationRequest.getAuthorizationRequestUri());
    }
}
```

The redirect URI resolution replaces template variables with runtime values:

```java
private String resolveRedirectUri(HttpServletRequest request,
        ClientRegistration clientRegistration) {
    String redirectUri = clientRegistration.getRedirectUri();
    String baseUrl = getBaseUrl(request);
    redirectUri = redirectUri.replace("{baseUrl}", baseUrl);
    redirectUri = redirectUri.replace("{registrationId}",
            clientRegistration.getRegistrationId());
    return redirectUri;
}
```

**Why does this filter implement `Filter` directly instead of extending `SimpleAbstractAuthenticationProcessingFilter`?**
This filter does not authenticate anyone -- it just redirects. It does not interact
with `AuthenticationManager`, does not set `SecurityContext`, and does not need
success/failure handlers. It is a pure redirect, not an authentication ceremony.
`SimpleAbstractAuthenticationProcessingFilter` would add unnecessary lifecycle methods.

**Why store the authorization request in the session?** The callback (step 2) needs to
correlate the response with the original request. The `state` parameter must match, and the
`redirectUri` is needed for the token exchange. The session is the natural place to store
this short-lived data -- it survives the redirect but does not require a database.

### 14g. The login filter: SimpleOAuth2LoginAuthenticationFilter

This filter handles the callback from the authorization server. It extends
`SimpleAbstractAuthenticationProcessingFilter` because it performs a one-time login ceremony:

```java
public class SimpleOAuth2LoginAuthenticationFilter
        extends SimpleAbstractAuthenticationProcessingFilter {

    public static final String DEFAULT_FILTER_PROCESSES_URI = "/login/oauth2/code/";

    public SimpleOAuth2LoginAuthenticationFilter(
            ClientRegistrationRepository clientRegistrationRepository) {
        super(request -> getRequestPath(request).startsWith(DEFAULT_FILTER_PROCESSES_URI));
        this.clientRegistrationRepository = clientRegistrationRepository;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request,
            HttpServletResponse response) throws AuthenticationException {

        // 1. Extract registration ID from URL path
        String registrationId = path.substring(DEFAULT_FILTER_PROCESSES_URI.length());

        // 2. Look up the client registration
        ClientRegistration clientRegistration =
                this.clientRegistrationRepository.findByRegistrationId(registrationId);

        // 3. Retrieve and remove the saved authorization request from session
        OAuth2AuthorizationRequest authorizationRequest =
                (OAuth2AuthorizationRequest) request.getSession()
                        .getAttribute(AUTHORIZATION_REQUEST_ATTR);
        request.getSession().removeAttribute(AUTHORIZATION_REQUEST_ATTR);

        // 4. Build the authorization response from query parameters
        String code = request.getParameter("code");
        String state = request.getParameter("state");
        OAuth2AuthorizationResponse authorizationResponse =
                OAuth2AuthorizationResponse.success(code, state);

        // 5. Validate state parameter
        if (!authorizationRequest.getState().equals(state)) {
            throw new OAuth2AuthenticationException(
                    new OAuth2Error("invalid_state", "State parameter mismatch", null));
        }

        // 6. Create the exchange and unauthenticated token
        OAuth2AuthorizationExchange exchange =
                new OAuth2AuthorizationExchange(authorizationRequest, authorizationResponse);
        OAuth2LoginAuthenticationToken authenticationRequest =
                new OAuth2LoginAuthenticationToken(clientRegistration, exchange);

        // 7. Delegate to AuthenticationManager
        OAuth2LoginAuthenticationToken authResult =
                (OAuth2LoginAuthenticationToken) getAuthenticationManager().authenticate(authenticationRequest);

        // 8. Convert to OAuth2AuthenticationToken (the final principal)
        OAuth2User oauth2User = (OAuth2User) authResult.getPrincipal();
        return new OAuth2AuthenticationToken(
                oauth2User, oauth2User.getAuthorities(),
                clientRegistration.getRegistrationId());
    }
}
```

**Why does the login filter extend `SimpleAbstractAuthenticationProcessingFilter`?**
Unlike the redirect filter, this filter performs a full authentication ceremony:
match a URL, attempt authentication, handle success/failure, store the result in
`SecurityContext`. This is exactly the lifecycle that `SimpleAbstractAuthenticationProcessingFilter`
provides -- the same base class used by `SimpleUsernamePasswordAuthenticationFilter` in
Feature 6. The only difference is what `attemptAuthentication()` does.

**Why convert from `OAuth2LoginAuthenticationToken` to `OAuth2AuthenticationToken`?**
`OAuth2LoginAuthenticationToken` is a transient token used only during the authentication
process. It carries the authorization exchange (code, state, redirect URI) which is no
longer needed after authentication completes. `OAuth2AuthenticationToken` is the final,
lightweight token stored in the `SecurityContext` -- it carries just the `OAuth2User`
principal, authorities, and the registration ID.

### 14h. The authentication tokens: dual-constructor pattern

**`OAuth2LoginAuthenticationToken`** follows the same dual-constructor pattern as
`UsernamePasswordAuthenticationToken` from Feature 2:

```java
public class OAuth2LoginAuthenticationToken extends AbstractAuthenticationToken {

    // Constructor 1: UNAUTHENTICATED -- created by the login filter
    public OAuth2LoginAuthenticationToken(ClientRegistration clientRegistration,
            OAuth2AuthorizationExchange authorizationExchange) {
        super(null);            // no authorities yet
        this.principal = null;  // no user yet
        this.accessToken = null;
        setAuthenticated(false);
    }

    // Constructor 2: AUTHENTICATED -- created by the authentication provider
    public OAuth2LoginAuthenticationToken(ClientRegistration clientRegistration,
            OAuth2AuthorizationExchange authorizationExchange,
            OAuth2User principal,
            Collection<? extends GrantedAuthority> authorities,
            OAuth2AccessToken accessToken) {
        super(authorities);
        this.principal = principal;
        this.accessToken = accessToken;
        setAuthenticated(true);
    }
}
```

**`OAuth2AuthenticationToken`** is the final token stored in the SecurityContext. It is
always authenticated:

```java
public class OAuth2AuthenticationToken extends AbstractAuthenticationToken {

    private final OAuth2User principal;
    private final String authorizedClientRegistrationId;

    public OAuth2AuthenticationToken(OAuth2User principal,
            Collection<? extends GrantedAuthority> authorities,
            String authorizedClientRegistrationId) {
        super(authorities);
        this.principal = principal;
        this.authorizedClientRegistrationId = authorizedClientRegistrationId;
        setAuthenticated(true);
    }

    @Override
    public String getName() {
        return this.principal.getName();
    }
}
```

**Why two separate token types (`OAuth2LoginAuthenticationToken` and `OAuth2AuthenticationToken`)?**
They serve different purposes. The login token carries the full authorization exchange
(needed by the provider to exchange the code for tokens). The authentication token is
the clean result (just user + authorities + registration ID). Keeping them separate means
the SecurityContext does not carry transient data (codes, states, redirect URIs) that is
only relevant during the login flow.

### 14i. The user service: OAuth2UserService and DefaultOAuth2UserService

**`OAuth2UserService`** is a `@FunctionalInterface` that loads user attributes from the
provider's UserInfo endpoint:

```java
@FunctionalInterface
public interface OAuth2UserService {
    OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException;
}
```

**`OAuth2UserRequest`** bundles the `ClientRegistration` (which has the UserInfo URI) and
the `OAuth2AccessToken` (needed for the Bearer authorization header):

```java
public class OAuth2UserRequest {
    private final ClientRegistration clientRegistration;
    private final OAuth2AccessToken accessToken;
}
```

**`DefaultOAuth2UserService`** makes an HTTP GET to the UserInfo endpoint with the access
token, parses the JSON response, and builds a `DefaultOAuth2User`:

```java
public class DefaultOAuth2UserService implements OAuth2UserService {

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) {
        String userInfoUri = userRequest.getClientRegistration().getUserInfoUri();
        String userNameAttributeName = userRequest.getClientRegistration().getUserNameAttributeName();

        Map<String, Object> attributes = fetchUserAttributes(userInfoUri,
                userRequest.getAccessToken().getTokenValue());

        List<GrantedAuthority> authorities = new ArrayList<>();
        authorities.add(new SimpleGrantedAuthority("OAUTH2_USER"));
        for (String scope : userRequest.getAccessToken().getScopes()) {
            authorities.add(new SimpleGrantedAuthority("SCOPE_" + scope));
        }

        return new DefaultOAuth2User(authorities, attributes, userNameAttributeName);
    }

    private Map<String, Object> fetchUserAttributes(String userInfoUri, String accessToken) {
        HttpURLConnection connection = (HttpURLConnection) URI.create(userInfoUri).toURL().openConnection();
        connection.setRequestMethod("GET");
        connection.setRequestProperty("Authorization", "Bearer " + accessToken);
        connection.setRequestProperty("Accept", "application/json");
        // ... read response, parse JSON ...
    }
}
```

**Why is `OAuth2UserService` a `@FunctionalInterface`?** This is the primary extension
point for custom user loading. Applications commonly need to:
- Enrich provider attributes with data from a local database
- Map provider-specific attributes to a custom `OAuth2User` subclass
- Merge authorities from the provider with application-specific roles

Making it a functional interface means you can supply a lambda:

```java
http.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(userInfo -> userInfo
        .userService(request -> {
            OAuth2User defaultUser = new DefaultOAuth2UserService().loadUser(request);
            // Enrich with local roles from database
            return enrichWithLocalRoles(defaultUser);
        })));
```

### 14j. The authentication provider: OAuth2LoginAuthenticationProvider

The provider performs the two-step OAuth2 dance: token exchange followed by user loading:

```java
public class OAuth2LoginAuthenticationProvider implements AuthenticationProvider {

    private final OAuth2UserService userService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        OAuth2LoginAuthenticationToken authToken = (OAuth2LoginAuthenticationToken) authentication;

        // Validate the authorization response
        OAuth2AuthorizationResponse authorizationResponse =
                authToken.getAuthorizationExchange().getAuthorizationResponse();
        if (!authorizationResponse.isSuccess()) {
            throw new OAuth2AuthenticationException(
                    new OAuth2Error(authorizationResponse.getError(), ...));
        }

        // Step 1: Exchange the authorization code for an access token
        OAuth2AccessToken accessToken = exchangeCodeForToken(
                authToken.getClientRegistration(),
                authorizationResponse.getCode(),
                authToken.getAuthorizationExchange().getAuthorizationRequest().getRedirectUri());

        // Step 2: Load user info using the access token
        OAuth2UserRequest userRequest = new OAuth2UserRequest(
                authToken.getClientRegistration(), accessToken);
        OAuth2User oauth2User = this.userService.loadUser(userRequest);

        // Build the authenticated token
        return new OAuth2LoginAuthenticationToken(
                authToken.getClientRegistration(),
                authToken.getAuthorizationExchange(),
                oauth2User,
                new ArrayList<>(oauth2User.getAuthorities()),
                accessToken);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return OAuth2LoginAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

The token exchange uses `HttpURLConnection` to POST to the token endpoint with client
credentials via HTTP Basic authentication:

```java
private OAuth2AccessToken exchangeCodeForToken(ClientRegistration clientRegistration,
        String code, String redirectUri) {
    HttpURLConnection connection = (HttpURLConnection) URI.create(
            clientRegistration.getTokenUri()).toURL().openConnection();
    connection.setRequestMethod("POST");
    connection.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");

    // Client authentication via Basic auth
    String credentials = clientRegistration.getClientId() + ":"
            + clientRegistration.getClientSecret();
    String encoded = Base64.getEncoder().encodeToString(credentials.getBytes(UTF_8));
    connection.setRequestProperty("Authorization", "Basic " + encoded);

    // Form body: grant_type=authorization_code&code=...&redirect_uri=...
    // ... write body, read response, parse access_token from JSON ...
}
```

**Why does the provider accept `OAuth2UserService` as a constructor parameter?** This
is dependency injection. The provider does not know how to load users -- it delegates
to whatever `OAuth2UserService` the configurer provides. This allows the configurer's
`userInfoEndpoint()` DSL to plug in a custom service without modifying the provider.

### 14k. The configurer: SimpleOAuth2LoginConfigurer

The configurer ties everything together through the two-phase lifecycle:

**`init()`** -- registers the entry point and authentication provider:

```java
@Override
public void init(SimpleHttpSecurity builder) throws Exception {
    if (this.failureUrl == null) {
        this.failureUrl = this.loginPage + "?error";
    }
    if (this.authenticationEntryPoint == null) {
        this.authenticationEntryPoint = new SimpleLoginUrlAuthenticationEntryPoint(this.loginPage);
    }

    // Register entry point so ExceptionTranslationFilter can use it
    builder.setSharedObject(AuthenticationEntryPoint.class, this.authenticationEntryPoint);

    // Register the OAuth2 login authentication provider
    OAuth2UserService userService = getUserService();
    AuthenticationProvider provider = new OAuth2LoginAuthenticationProvider(userService);
    builder.authenticationProvider(provider);
}
```

**`configure()`** -- creates and adds both filters:

```java
@Override
public void configure(SimpleHttpSecurity builder) throws Exception {
    ClientRegistrationRepository repository = getClientRegistrationRepository(builder);

    // 1. Create and add the redirect filter (before the login filter)
    SimpleOAuth2AuthorizationRequestRedirectFilter redirectFilter =
            new SimpleOAuth2AuthorizationRequestRedirectFilter(repository);
    builder.addFilterBefore(redirectFilter,
            SimpleUsernamePasswordAuthenticationFilter.class);

    // 2. Create and configure the login filter
    SimpleOAuth2LoginAuthenticationFilter loginFilter =
            new SimpleOAuth2LoginAuthenticationFilter(repository);

    // Build a dedicated AuthenticationManager with only the OAuth2 provider
    OAuth2UserService userService = getUserService();
    AuthenticationProvider provider = new OAuth2LoginAuthenticationProvider(userService);
    AuthenticationManager oauthAuthManager = new SimpleProviderManager(List.of(provider));
    loginFilter.setAuthenticationManager(oauthAuthManager);

    // Wire success/failure handlers
    loginFilter.setAuthenticationSuccessHandler(
            new SimpleRedirectAuthenticationSuccessHandler(this.defaultSuccessUrl));
    loginFilter.setAuthenticationFailureHandler(
            new SimpleRedirectAuthenticationFailureHandler(this.failureUrl));

    // Wire SecurityContextRepository
    SecurityContextRepository contextRepository =
            builder.getSharedObject(SecurityContextRepository.class);
    if (contextRepository != null) {
        loginFilter.setSecurityContextRepository(contextRepository);
    }

    builder.addFilterAfter(loginFilter,
            SimpleOAuth2AuthorizationRequestRedirectFilter.class);
}
```

**Why a dedicated `AuthenticationManager` for the login filter?** The same reasoning as
Feature 13's resource server. The OAuth2 login provider only handles
`OAuth2LoginAuthenticationToken`. Using a separate `SimpleProviderManager` means the login
filter is self-contained -- it does not depend on a shared `AuthenticationManager` that
might contain `DaoAuthenticationProvider` for form login.

**Why two filters instead of one?** The redirect (step 1) and the callback (step 2)
are separated by a browser redirect. They happen on different HTTP requests, match different
URL patterns, and have completely different behaviors. The redirect filter just saves state
and redirects. The login filter performs authentication. Combining them into one filter
would require handling two unrelated request patterns with different lifecycles in the
same `doFilter` method -- a violation of the Single Responsibility Principle.

### 14l. DSL method on SimpleHttpSecurity

The convenience method follows the same `getOrApply` pattern as all other DSL methods:

```java
public SimpleHttpSecurity oauth2Login(Customizer<SimpleOAuth2LoginConfigurer> oauth2LoginCustomizer) {
    oauth2LoginCustomizer.customize(getOrApply(new SimpleOAuth2LoginConfigurer()));
    return this;
}
```

The filter order map was updated to include the OAuth2 redirect filter at position 270
(after logout at 250, before username/password authentication at 300):

```java
static {
    int order = 100;
    FILTER_ORDER_MAP.put("SimpleSecurityContextHolderFilter", order);       // 100
    order += 100;
    FILTER_ORDER_MAP.put("SimpleCsrfFilter", order);                        // 200
    order += 50;
    FILTER_ORDER_MAP.put("SimpleLogoutFilter", order);                      // 250
    order += 20;
    FILTER_ORDER_MAP.put("SimpleOAuth2AuthorizationRequestRedirectFilter", order); // 270  <-- NEW
    order += 30;
    FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", order);     // 300
    order += 100;
    FILTER_ORDER_MAP.put("SimpleBasicAuthenticationFilter", order);         // 400
    order += 50;
    FILTER_ORDER_MAP.put("SimpleBearerTokenAuthenticationFilter", order);   // 450
    order += 50;
    FILTER_ORDER_MAP.put("SimpleExceptionTranslationFilter", order);        // 500
    order += 100;
    FILTER_ORDER_MAP.put("SimpleAuthorizationFilter", order);               // 600
}
```

**Why position 270 (after logout at 250, before username/password at 300)?** The OAuth2
redirect filter should run after CSRF and logout (which handle their own request patterns),
but before the authentication filters. The login filter is added via `addFilterAfter`
relative to the redirect filter, placing it just after at approximately position 271.
This mirrors the real framework where `OAuth2AuthorizationRequestRedirectFilter` sits
between logout and the authentication filter stack.

---

## Try It Yourself

**Challenge 1**: Add PKCE (Proof Key for Code Exchange) support. Create a `code_verifier`
(random string) and `code_challenge` (SHA-256 hash of the verifier) when building the
authorization request. Include `code_challenge` and `code_challenge_method=S256` in the
redirect URL. Send the `code_verifier` in the token exchange. Why is PKCE important for
public clients (mobile apps, SPAs) that cannot securely store a client secret?

**Challenge 2**: Add an auto-generated login page. When no custom `loginPage` is configured,
have the configurer generate an HTML page that lists all registered OAuth2 providers as
clickable links. Each link should point to `/oauth2/authorization/{registrationId}`.
Use `SimpleInMemoryClientRegistrationRepository`'s `Iterable` interface to enumerate
all registrations. How does the real framework's `DefaultLoginPageGeneratingFilter` do this?

**Challenge 3**: Add an `OAuth2AuthorizedClient` model that stores the access token (and
optionally refresh token) associated with a principal and client registration. Create an
`OAuth2AuthorizedClientRepository` interface with `loadAuthorizedClient` and
`saveAuthorizedClient` methods. Wire the login filter to save the authorized client after
successful authentication. How would this enable downstream filters to make API calls to
the OAuth2 provider on behalf of the user?

---

## Why This Works

**The authorization code flow as two separate HTTP requests**: The OAuth2 login flow is
fundamentally split across two HTTP requests separated by a browser redirect. The redirect
filter (request 1) and the login filter (request 2) model this split naturally. The session
bridges the two requests by storing the authorization request (including the `state`
parameter). This clean separation makes each filter simple and testable in isolation --
you can test the redirect filter without an authentication manager, and test the login
filter by injecting a saved authorization request directly into the session.

**The state parameter as CSRF protection**: The `state` parameter is a random UUID generated
by the redirect filter and saved in the session. When the callback arrives, the login filter
verifies that the `state` in the callback matches the one in the session. Without this check,
an attacker could craft a callback URL with a malicious authorization code and trick the
user into visiting it -- the login filter would blindly exchange the code and authenticate
the user with the attacker's account. This is the OAuth2-specific analog of the CSRF token
from Feature 9.

**Injectable `OAuth2UserService` for custom user loading**: The `OAuth2UserService` is a
functional interface that is wired through the configurer's `userInfoEndpoint()` DSL.
The default implementation (`DefaultOAuth2UserService`) calls the provider's UserInfo
endpoint, but applications can replace it entirely. This is the most common extension
point in OAuth2 login -- applications often need to merge provider attributes with local
database records, assign application-specific roles, or create local user accounts on
first login.

**The dual-constructor pattern for `OAuth2LoginAuthenticationToken`**: This pattern -- one
constructor for the unauthenticated request, one for the authenticated result -- is the
same pattern used by `UsernamePasswordAuthenticationToken` in Feature 2. It makes the
`AuthenticationManager` contract work consistently: the filter creates an unauthenticated
token, passes it to the manager, and gets back an authenticated token. The type system
enforces this -- the unauthenticated constructor sets `authenticated=false` and
`principal=null`, while the authenticated constructor requires `OAuth2User`, authorities,
and `accessToken`.

**ClientRegistration's builder with grant-type-specific validation**: The builder validates
different fields depending on the grant type. Authorization code requires `redirectUri`,
`authorizationUri`, and `tokenUri`. Client credentials might not need any of those. This
compile-time validation catches configuration errors early -- before the first login
attempt -- rather than at runtime when the redirect filter tries to use a missing URI.

---

## What We Enhanced

| What Changed | Before (Feature 13) | After (Feature 14) | Why |
|---|---|---|---|
| `SimpleHttpSecurity` | `oauth2ResourceServer()` only | Added `oauth2Login()` DSL method and `SimpleOAuth2AuthorizationRequestRedirectFilter` to filter order map (position 270) | Enables "Login with Google/GitHub" via the same DSL pattern |
| New modules | `simple-security-oauth2-core` (shared) | + `simple-security-oauth2-client` with registration, web, authentication, and userinfo packages | Mirrors Spring Security's separate oauth2-client module |
| Authentication tokens | `BearerTokenAuthenticationToken` / `JwtAuthenticationToken` | + `OAuth2LoginAuthenticationToken` (unauthenticated/authenticated) and `OAuth2AuthenticationToken` (final SecurityContext principal) | Different flows need different token representations |
| Provider pattern | `SimpleJwtAuthenticationProvider` | + `OAuth2LoginAuthenticationProvider` that exchanges code for tokens and loads user info | The authorization code flow requires a multi-step provider (token exchange + user loading) |
| User model | JWT claims as principal | + `OAuth2User` / `DefaultOAuth2User` carrying provider attributes and configurable name attribute | OAuth2 users are identified by provider-specific attributes, not JWT claims |
| Filter chain | Single auth filter per mechanism | Two coordinated filters: redirect filter (initiates) + login filter (completes) | The authorization code flow spans two HTTP requests separated by a browser redirect |
| `simple-security-config/build.gradle` | `simple-security-oauth2-jose` and `simple-security-oauth2-resource-server` | + `implementation project(':simple-security-oauth2-client')` | Config module needs access to OAuth2 client types for the configurer |
| `simple-security-oauth2-core` | `OAuth2Error`, `OAuth2AuthenticationException` | + `AuthorizationGrantType`, `OAuth2AccessToken`, `OAuth2User`, `DefaultOAuth2User` | Shared types used by both resource server and client modules |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|---|---|---|
| `AuthorizationGrantType` | `AuthorizationGrantType` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/AuthorizationGrantType.java` |
| `OAuth2AccessToken` | `OAuth2AccessToken` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/OAuth2AccessToken.java` |
| `OAuth2User` | `OAuth2User` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/user/OAuth2User.java` |
| `DefaultOAuth2User` | `DefaultOAuth2User` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/user/DefaultOAuth2User.java` |
| `ClientRegistration` | `ClientRegistration` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/ClientRegistration.java` |
| `ClientRegistrationRepository` | `ClientRegistrationRepository` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/ClientRegistrationRepository.java` |
| `SimpleInMemoryClientRegistrationRepository` | `InMemoryClientRegistrationRepository` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/InMemoryClientRegistrationRepository.java` |
| `OAuth2AuthorizationRequest` | `OAuth2AuthorizationRequest` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/endpoint/OAuth2AuthorizationRequest.java` |
| `OAuth2AuthorizationResponse` | `OAuth2AuthorizationResponse` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/endpoint/OAuth2AuthorizationResponse.java` |
| `OAuth2AuthorizationExchange` | `OAuth2AuthorizationExchange` | `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/endpoint/OAuth2AuthorizationExchange.java` |
| `SimpleOAuth2AuthorizationRequestRedirectFilter` | `OAuth2AuthorizationRequestRedirectFilter` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/web/OAuth2AuthorizationRequestRedirectFilter.java` |
| `SimpleOAuth2LoginAuthenticationFilter` | `OAuth2LoginAuthenticationFilter` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/web/OAuth2LoginAuthenticationFilter.java` |
| `OAuth2LoginAuthenticationToken` | `OAuth2LoginAuthenticationToken` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/authentication/OAuth2LoginAuthenticationToken.java` |
| `OAuth2AuthenticationToken` | `OAuth2AuthenticationToken` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/authentication/OAuth2AuthenticationToken.java` |
| `OAuth2LoginAuthenticationProvider` | `OAuth2LoginAuthenticationProvider` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/authentication/OAuth2LoginAuthenticationProvider.java` |
| `OAuth2UserRequest` | `OAuth2UserRequest` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/userinfo/OAuth2UserRequest.java` |
| `OAuth2UserService` | `OAuth2UserService` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/userinfo/OAuth2UserService.java` |
| `DefaultOAuth2UserService` | `DefaultOAuth2UserService` | `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/userinfo/DefaultOAuth2UserService.java` |
| `SimpleOAuth2LoginConfigurer` | `OAuth2LoginConfigurer` | `config/src/main/java/org/springframework/security/config/annotation/web/configurers/oauth2/client/OAuth2LoginConfigurer.java` |

### Key differences from the real framework

**`ClientRegistration`**: The real class nests provider details inside a `ProviderDetails`
inner class (with its own `UserInfoEndpoint` sub-class). It supports
`ClientAuthenticationMethod` (client_secret_basic, client_secret_post, etc.),
`ClientSettings` for PKCE, and a copy builder via `ClientRegistration.withClientRegistration()`.
Our simplified version flattens all provider fields directly onto `ClientRegistration` and
assumes `client_secret_basic` authentication.

**`OAuth2AuthorizationRequestRedirectFilter`**: The real class delegates to a separate
`OAuth2AuthorizationRequestResolver` interface (with a default implementation). It also
uses an `AuthorizationRequestRepository` interface for storing/retrieving authorization
requests (defaulting to `HttpSessionOAuth2AuthorizationRequestRepository`). It supports
handling `ClientAuthorizationRequiredException` for triggering re-authorization. Our version
inlines the resolution logic and stores directly in the session.

**`OAuth2LoginAuthenticationFilter`**: The real class extends
`AbstractAuthenticationProcessingFilter` and integrates with
`OAuth2AuthorizedClientRepository` for saving authorized clients. It uses a configurable
`AuthenticationResultConverter` to transform the authentication result. Our version
directly converts to `OAuth2AuthenticationToken` in `attemptAuthentication()`.

**`OAuth2LoginAuthenticationProvider`**: The real class delegates token exchange to a
separate `OAuth2AccessTokenResponseClient` interface (with a default
`DefaultAuthorizationCodeTokenResponseClient` implementation that uses `RestOperations`).
It also detects the `"openid"` scope to hand off to the OIDC authentication provider.
Our version uses `HttpURLConnection` directly and does not support OIDC.

**`DefaultOAuth2UserService`**: The real class uses `RestOperations` (with configurable
`RequestEntityConverter` and `HttpMessageConverters`) for HTTP communication. It uses
Jackson for JSON parsing. Our version uses `HttpURLConnection` with a simple hand-written
JSON parser for flat objects.

---

## Complete Code

### [NEW] `simple-security-oauth2-core/src/main/java/simple/security/oauth2/core/AuthorizationGrantType.java`

```java
package simple.security.oauth2.core;

import java.util.Objects;

/**
 * Represents an OAuth 2.0 Authorization Grant Type.
 *
 * <p>A simple value object wrapping the grant type string (e.g., "authorization_code").
 * Static constants are provided for common grant types; custom types can be created
 * via the constructor.
 *
 * <p>Simplifications vs. real AuthorizationGrantType:
 * <ul>
 *   <li>No serialization support</li>
 *   <li>Only AUTHORIZATION_CODE and CLIENT_CREDENTIALS constants</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.AuthorizationGrantType
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/AuthorizationGrantType.java
 */
public final class AuthorizationGrantType {

	public static final AuthorizationGrantType AUTHORIZATION_CODE =
			new AuthorizationGrantType("authorization_code");

	public static final AuthorizationGrantType CLIENT_CREDENTIALS =
			new AuthorizationGrantType("client_credentials");

	private final String value;

	public AuthorizationGrantType(String value) {
		if (value == null || value.isBlank()) {
			throw new IllegalArgumentException("value cannot be empty");
		}
		this.value = value;
	}

	public String getValue() {
		return this.value;
	}

	@Override
	public boolean equals(Object o) {
		if (this == o) return true;
		if (!(o instanceof AuthorizationGrantType that)) return false;
		return Objects.equals(this.value, that.value);
	}

	@Override
	public int hashCode() {
		return Objects.hashCode(this.value);
	}

	@Override
	public String toString() {
		return this.value;
	}

}
```

### [NEW] `simple-security-oauth2-core/src/main/java/simple/security/oauth2/core/OAuth2AccessToken.java`

```java
package simple.security.oauth2.core;

import java.util.Collections;
import java.util.Set;

/**
 * Represents an OAuth 2.0 Access Token.
 *
 * <p>Holds the token value and the scopes associated with the token.
 * This is returned after the authorization code exchange and used to
 * call protected resource endpoints (e.g., the UserInfo endpoint).
 *
 * <p>Simplifications vs. real OAuth2AccessToken:
 * <ul>
 *   <li>No token type (always assumed Bearer)</li>
 *   <li>No issuedAt/expiresAt timestamps</li>
 *   <li>Does not extend AbstractOAuth2Token</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.OAuth2AccessToken
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/OAuth2AccessToken.java
 */
public class OAuth2AccessToken {

	private final String tokenValue;

	private final Set<String> scopes;

	public OAuth2AccessToken(String tokenValue, Set<String> scopes) {
		if (tokenValue == null || tokenValue.isBlank()) {
			throw new IllegalArgumentException("tokenValue cannot be empty");
		}
		this.tokenValue = tokenValue;
		this.scopes = (scopes != null) ? Collections.unmodifiableSet(scopes) : Collections.emptySet();
	}

	public String getTokenValue() {
		return this.tokenValue;
	}

	public Set<String> getScopes() {
		return this.scopes;
	}

	@Override
	public String toString() {
		return "OAuth2AccessToken [scopes=" + this.scopes + "]";
	}

}
```

### [NEW] `simple-security-oauth2-core/src/main/java/simple/security/oauth2/core/user/OAuth2User.java`

```java
package simple.security.oauth2.core.user;

import simple.security.core.GrantedAuthority;

import java.util.Collection;
import java.util.Map;

/**
 * Represents a user authenticated via an OAuth 2.0 provider.
 *
 * <p>Combines user attributes (returned from the UserInfo endpoint) with
 * granted authorities. This is the principal type stored in the
 * {@link simple.security.core.context.SecurityContext} after OAuth2 login.
 *
 * <p>Simplifications vs. real OAuth2User:
 * <ul>
 *   <li>Combines OAuth2User and OAuth2AuthenticatedPrincipal into a single interface</li>
 *   <li>No AuthenticatedPrincipal hierarchy</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.user.OAuth2User
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/user/OAuth2User.java
 */
public interface OAuth2User {

	/**
	 * Returns the name of the user (the value of the name attribute).
	 */
	String getName();

	/**
	 * Returns the user attributes from the UserInfo endpoint response.
	 */
	Map<String, Object> getAttributes();

	/**
	 * Returns the authorities granted to the user.
	 */
	Collection<? extends GrantedAuthority> getAuthorities();

}
```

### [NEW] `simple-security-oauth2-core/src/main/java/simple/security/oauth2/core/user/DefaultOAuth2User.java`

```java
package simple.security.oauth2.core.user;

import simple.security.core.GrantedAuthority;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Map;
import java.util.Objects;

/**
 * Default implementation of {@link OAuth2User}.
 *
 * <p>Stores the user's authorities, attributes (from the UserInfo endpoint),
 * and a {@code nameAttributeKey} that identifies which attribute holds the user's
 * name. Different OAuth2 providers use different attribute names for the user's
 * identifier (e.g., GitHub uses "login", Google uses "sub"), so the key is
 * configurable.
 *
 * <p>Simplifications vs. real DefaultOAuth2User:
 * <ul>
 *   <li>No serialization support</li>
 *   <li>Authorities not sorted — stored in insertion order</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.user.DefaultOAuth2User
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/user/DefaultOAuth2User.java
 */
public class DefaultOAuth2User implements OAuth2User {

	private final Collection<GrantedAuthority> authorities;

	private final Map<String, Object> attributes;

	private final String nameAttributeKey;

	public DefaultOAuth2User(Collection<? extends GrantedAuthority> authorities,
			Map<String, Object> attributes, String nameAttributeKey) {
		if (attributes == null || attributes.isEmpty()) {
			throw new IllegalArgumentException("attributes cannot be empty");
		}
		if (nameAttributeKey == null || nameAttributeKey.isBlank()) {
			throw new IllegalArgumentException("nameAttributeKey cannot be empty");
		}
		if (!attributes.containsKey(nameAttributeKey)) {
			throw new IllegalArgumentException(
					"Missing attribute '" + nameAttributeKey + "' in attributes");
		}
		this.authorities = (authorities != null)
				? Collections.unmodifiableList(new ArrayList<>(authorities))
				: Collections.emptyList();
		this.attributes = Collections.unmodifiableMap(attributes);
		this.nameAttributeKey = nameAttributeKey;
	}

	@Override
	public String getName() {
		return this.attributes.get(this.nameAttributeKey).toString();
	}

	@Override
	public Map<String, Object> getAttributes() {
		return this.attributes;
	}

	@Override
	public Collection<? extends GrantedAuthority> getAuthorities() {
		return this.authorities;
	}

	@Override
	public boolean equals(Object o) {
		if (this == o) return true;
		if (!(o instanceof DefaultOAuth2User that)) return false;
		return Objects.equals(this.authorities, that.authorities)
				&& Objects.equals(this.attributes, that.attributes)
				&& Objects.equals(this.nameAttributeKey, that.nameAttributeKey);
	}

	@Override
	public int hashCode() {
		return Objects.hash(this.authorities, this.attributes, this.nameAttributeKey);
	}

	@Override
	public String toString() {
		return "DefaultOAuth2User [name=" + getName()
				+ ", authorities=" + this.authorities + "]";
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/registration/ClientRegistration.java`

```java
package simple.security.oauth2.client.registration;

import simple.security.oauth2.core.AuthorizationGrantType;

import java.util.Collections;
import java.util.LinkedHashSet;
import java.util.Set;

/**
 * Represents a client registration with an OAuth 2.0 / OpenID Connect provider.
 *
 * <p>Contains all the information needed to initiate an authorization request and
 * exchange the authorization code for tokens: client ID/secret, the provider's
 * endpoint URIs, scopes, and a redirect URI template.
 *
 * <p>Immutable after construction. Use {@link #withRegistrationId(String)} to
 * create a {@link Builder}.
 *
 * <h2>Builder flattening</h2>
 * <p>In the real framework, provider details are nested in a ProviderDetails inner class.
 * This simplified version flattens those fields directly onto ClientRegistration for
 * easier construction and access.
 *
 * <p>Simplifications vs. real ClientRegistration:
 * <ul>
 *   <li>No ProviderDetails inner class — fields are flattened</li>
 *   <li>No ClientAuthenticationMethod — assumes client_secret_basic</li>
 *   <li>No ClientSettings (PKCE)</li>
 *   <li>No configuration metadata</li>
 *   <li>No copy builder (withClientRegistration)</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.registration.ClientRegistration
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/ClientRegistration.java
 */
public final class ClientRegistration {

	private final String registrationId;

	private final String clientId;

	private final String clientSecret;

	private final AuthorizationGrantType authorizationGrantType;

	private final String redirectUri;

	private final Set<String> scopes;

	private final String authorizationUri;

	private final String tokenUri;

	private final String userInfoUri;

	private final String userNameAttributeName;

	private final String clientName;

	private ClientRegistration(Builder builder) {
		this.registrationId = builder.registrationId;
		this.clientId = builder.clientId;
		this.clientSecret = builder.clientSecret;
		this.authorizationGrantType = builder.authorizationGrantType;
		this.redirectUri = builder.redirectUri;
		this.scopes = Collections.unmodifiableSet(new LinkedHashSet<>(builder.scopes));
		this.authorizationUri = builder.authorizationUri;
		this.tokenUri = builder.tokenUri;
		this.userInfoUri = builder.userInfoUri;
		this.userNameAttributeName = builder.userNameAttributeName;
		this.clientName = (builder.clientName != null) ? builder.clientName : builder.registrationId;
	}

	public static Builder withRegistrationId(String registrationId) {
		return new Builder(registrationId);
	}

	public String getRegistrationId() {
		return this.registrationId;
	}

	public String getClientId() {
		return this.clientId;
	}

	public String getClientSecret() {
		return this.clientSecret;
	}

	public AuthorizationGrantType getAuthorizationGrantType() {
		return this.authorizationGrantType;
	}

	public String getRedirectUri() {
		return this.redirectUri;
	}

	public Set<String> getScopes() {
		return this.scopes;
	}

	public String getAuthorizationUri() {
		return this.authorizationUri;
	}

	public String getTokenUri() {
		return this.tokenUri;
	}

	public String getUserInfoUri() {
		return this.userInfoUri;
	}

	public String getUserNameAttributeName() {
		return this.userNameAttributeName;
	}

	public String getClientName() {
		return this.clientName;
	}

	@Override
	public String toString() {
		return "ClientRegistration [registrationId=" + this.registrationId
				+ ", clientId=" + this.clientId + "]";
	}

	/**
	 * Builder for {@link ClientRegistration}.
	 *
	 * <p>Provider detail fields (authorizationUri, tokenUri, userInfoUri) are
	 * set directly on the builder — no nested ProviderDetails builder needed.
	 */
	public static final class Builder {

		private final String registrationId;

		private String clientId;

		private String clientSecret;

		private AuthorizationGrantType authorizationGrantType;

		private String redirectUri;

		private final Set<String> scopes = new LinkedHashSet<>();

		private String authorizationUri;

		private String tokenUri;

		private String userInfoUri;

		private String userNameAttributeName;

		private String clientName;

		private Builder(String registrationId) {
			if (registrationId == null || registrationId.isBlank()) {
				throw new IllegalArgumentException("registrationId cannot be empty");
			}
			this.registrationId = registrationId;
		}

		public Builder clientId(String clientId) {
			this.clientId = clientId;
			return this;
		}

		public Builder clientSecret(String clientSecret) {
			this.clientSecret = clientSecret;
			return this;
		}

		public Builder authorizationGrantType(AuthorizationGrantType authorizationGrantType) {
			this.authorizationGrantType = authorizationGrantType;
			return this;
		}

		public Builder redirectUri(String redirectUri) {
			this.redirectUri = redirectUri;
			return this;
		}

		public Builder scope(String... scopes) {
			Collections.addAll(this.scopes, scopes);
			return this;
		}

		public Builder authorizationUri(String authorizationUri) {
			this.authorizationUri = authorizationUri;
			return this;
		}

		public Builder tokenUri(String tokenUri) {
			this.tokenUri = tokenUri;
			return this;
		}

		public Builder userInfoUri(String userInfoUri) {
			this.userInfoUri = userInfoUri;
			return this;
		}

		public Builder userNameAttributeName(String userNameAttributeName) {
			this.userNameAttributeName = userNameAttributeName;
			return this;
		}

		public Builder clientName(String clientName) {
			this.clientName = clientName;
			return this;
		}

		public ClientRegistration build() {
			if (this.clientId == null || this.clientId.isBlank()) {
				throw new IllegalArgumentException("clientId is required");
			}
			if (this.authorizationGrantType == null) {
				throw new IllegalArgumentException("authorizationGrantType is required");
			}
			if (AuthorizationGrantType.AUTHORIZATION_CODE.equals(this.authorizationGrantType)) {
				if (this.redirectUri == null || this.redirectUri.isBlank()) {
					throw new IllegalArgumentException(
							"redirectUri is required for authorization_code grant type");
				}
				if (this.authorizationUri == null || this.authorizationUri.isBlank()) {
					throw new IllegalArgumentException(
							"authorizationUri is required for authorization_code grant type");
				}
				if (this.tokenUri == null || this.tokenUri.isBlank()) {
					throw new IllegalArgumentException(
							"tokenUri is required for authorization_code grant type");
				}
			}
			return new ClientRegistration(this);
		}

	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/registration/ClientRegistrationRepository.java`

```java
package simple.security.oauth2.client.registration;

/**
 * Repository for looking up {@link ClientRegistration} by registration ID.
 *
 * <p>Abstracts storage of OAuth2 client registrations — whether they are
 * hardcoded in memory, loaded from a database, or fetched from a configuration
 * server.
 *
 * <p>This is the OAuth2 analog of {@code UserDetailsService}: a single-method
 * lookup that the security filter chain uses to find client details.
 *
 * Mirrors: org.springframework.security.oauth2.client.registration.ClientRegistrationRepository
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/ClientRegistrationRepository.java
 */
public interface ClientRegistrationRepository {

	/**
	 * Returns the {@link ClientRegistration} for the given registration ID,
	 * or {@code null} if not found.
	 */
	ClientRegistration findByRegistrationId(String registrationId);

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/registration/SimpleInMemoryClientRegistrationRepository.java`

```java
package simple.security.oauth2.client.registration;

import java.util.Arrays;
import java.util.Collections;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * In-memory implementation of {@link ClientRegistrationRepository}.
 *
 * <p>Stores registrations in a {@code Map<String, ClientRegistration>} keyed
 * by registration ID. Also implements {@code Iterable<ClientRegistration>}
 * so the login configurer can enumerate all registrations to build login page links.
 *
 * <p>Immutable after construction — the internal map cannot be modified.
 *
 * <p>Simplifications vs. real InMemoryClientRegistrationRepository:
 * <ul>
 *   <li>No ConcurrentHashMap — uses LinkedHashMap (no concurrent modification expected)</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/InMemoryClientRegistrationRepository.java
 */
public final class SimpleInMemoryClientRegistrationRepository
		implements ClientRegistrationRepository, Iterable<ClientRegistration> {

	private final Map<String, ClientRegistration> registrations;

	public SimpleInMemoryClientRegistrationRepository(ClientRegistration... registrations) {
		this(Arrays.asList(registrations));
	}

	public SimpleInMemoryClientRegistrationRepository(List<ClientRegistration> registrations) {
		Map<String, ClientRegistration> map = new LinkedHashMap<>();
		for (ClientRegistration registration : registrations) {
			if (map.containsKey(registration.getRegistrationId())) {
				throw new IllegalStateException(
						"Duplicate registration ID: " + registration.getRegistrationId());
			}
			map.put(registration.getRegistrationId(), registration);
		}
		this.registrations = Collections.unmodifiableMap(map);
	}

	@Override
	public ClientRegistration findByRegistrationId(String registrationId) {
		return this.registrations.get(registrationId);
	}

	@Override
	public Iterator<ClientRegistration> iterator() {
		return this.registrations.values().iterator();
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/web/OAuth2AuthorizationRequest.java`

```java
package simple.security.oauth2.client.web;

import java.util.Collections;
import java.util.Set;

/**
 * Represents the authorization request sent to the OAuth2 provider.
 *
 * <p>This object is created by the redirect filter before sending the user
 * to the authorization endpoint, and saved in the HTTP session so the callback
 * filter can correlate the response (using the {@code state} parameter).
 *
 * <p>Simplifications vs. real OAuth2AuthorizationRequest:
 * <ul>
 *   <li>No PKCE (code_challenge, code_verifier)</li>
 *   <li>No nonce</li>
 *   <li>No additionalParameters map</li>
 *   <li>No response_type field (always "code")</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.endpoint.OAuth2AuthorizationRequest
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/endpoint/OAuth2AuthorizationRequest.java
 */
public final class OAuth2AuthorizationRequest {

	private final String authorizationUri;

	private final String clientId;

	private final String redirectUri;

	private final Set<String> scopes;

	private final String state;

	private final String registrationId;

	private final String authorizationRequestUri;

	private OAuth2AuthorizationRequest(String authorizationUri, String clientId,
			String redirectUri, Set<String> scopes, String state,
			String registrationId) {
		this.authorizationUri = authorizationUri;
		this.clientId = clientId;
		this.redirectUri = redirectUri;
		this.scopes = Collections.unmodifiableSet(scopes);
		this.state = state;
		this.registrationId = registrationId;
		this.authorizationRequestUri = buildAuthorizationRequestUri();
	}

	public static OAuth2AuthorizationRequest create(String authorizationUri, String clientId,
			String redirectUri, Set<String> scopes, String state, String registrationId) {
		return new OAuth2AuthorizationRequest(authorizationUri, clientId,
				redirectUri, scopes, state, registrationId);
	}

	public String getAuthorizationUri() {
		return this.authorizationUri;
	}

	public String getClientId() {
		return this.clientId;
	}

	public String getRedirectUri() {
		return this.redirectUri;
	}

	public Set<String> getScopes() {
		return this.scopes;
	}

	public String getState() {
		return this.state;
	}

	public String getRegistrationId() {
		return this.registrationId;
	}

	/**
	 * Returns the fully assembled authorization request URI with query parameters.
	 * This is the URL the user's browser is redirected to.
	 */
	public String getAuthorizationRequestUri() {
		return this.authorizationRequestUri;
	}

	private String buildAuthorizationRequestUri() {
		StringBuilder sb = new StringBuilder(this.authorizationUri);
		sb.append("?response_type=code");
		sb.append("&client_id=").append(urlEncode(this.clientId));
		if (this.redirectUri != null) {
			sb.append("&redirect_uri=").append(urlEncode(this.redirectUri));
		}
		if (!this.scopes.isEmpty()) {
			sb.append("&scope=").append(urlEncode(String.join(" ", this.scopes)));
		}
		if (this.state != null) {
			sb.append("&state=").append(urlEncode(this.state));
		}
		return sb.toString();
	}

	private static String urlEncode(String value) {
		try {
			return java.net.URLEncoder.encode(value, java.nio.charset.StandardCharsets.UTF_8);
		}
		catch (Exception ex) {
			return value;
		}
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/web/OAuth2AuthorizationResponse.java`

```java
package simple.security.oauth2.client.web;

/**
 * Represents the authorization response received from the OAuth2 provider callback.
 *
 * <p>Contains the authorization code and state parameter from the callback URL.
 * If the provider returned an error instead of a code, the error fields are populated.
 *
 * <p>Simplifications vs. real OAuth2AuthorizationResponse:
 * <ul>
 *   <li>No redirectUri field</li>
 *   <li>Simplified error handling</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.core.endpoint.OAuth2AuthorizationResponse
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/endpoint/OAuth2AuthorizationResponse.java
 */
public final class OAuth2AuthorizationResponse {

	private final String code;

	private final String state;

	private final String error;

	private final String errorDescription;

	private OAuth2AuthorizationResponse(String code, String state, String error, String errorDescription) {
		this.code = code;
		this.state = state;
		this.error = error;
		this.errorDescription = errorDescription;
	}

	public static OAuth2AuthorizationResponse success(String code, String state) {
		return new OAuth2AuthorizationResponse(code, state, null, null);
	}

	public static OAuth2AuthorizationResponse error(String error, String errorDescription, String state) {
		return new OAuth2AuthorizationResponse(null, state, error, errorDescription);
	}

	public String getCode() {
		return this.code;
	}

	public String getState() {
		return this.state;
	}

	public String getError() {
		return this.error;
	}

	public String getErrorDescription() {
		return this.errorDescription;
	}

	public boolean isSuccess() {
		return this.code != null && this.error == null;
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/web/OAuth2AuthorizationExchange.java`

```java
package simple.security.oauth2.client.web;

/**
 * Pairs the {@link OAuth2AuthorizationRequest} with the {@link OAuth2AuthorizationResponse}
 * to represent a complete OAuth2 authorization exchange.
 *
 * <p>This is passed to the authentication provider so it has both the original
 * request (with client details and state) and the response (with the authorization code).
 *
 * Mirrors: org.springframework.security.oauth2.core.endpoint.OAuth2AuthorizationExchange
 * Source:  oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/endpoint/OAuth2AuthorizationExchange.java
 */
public final class OAuth2AuthorizationExchange {

	private final OAuth2AuthorizationRequest authorizationRequest;

	private final OAuth2AuthorizationResponse authorizationResponse;

	public OAuth2AuthorizationExchange(OAuth2AuthorizationRequest authorizationRequest,
			OAuth2AuthorizationResponse authorizationResponse) {
		this.authorizationRequest = authorizationRequest;
		this.authorizationResponse = authorizationResponse;
	}

	public OAuth2AuthorizationRequest getAuthorizationRequest() {
		return this.authorizationRequest;
	}

	public OAuth2AuthorizationResponse getAuthorizationResponse() {
		return this.authorizationResponse;
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/web/SimpleOAuth2AuthorizationRequestRedirectFilter.java`

```java
package simple.security.oauth2.client.web;

import java.io.IOException;
import java.util.UUID;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.registration.ClientRegistrationRepository;
import simple.security.oauth2.core.AuthorizationGrantType;

/**
 * Filter that initiates the OAuth 2.0 Authorization Code flow by redirecting the
 * user's browser to the provider's authorization endpoint.
 *
 * <p>Matches requests to {@code /oauth2/authorization/{registrationId}}, looks up
 * the client registration, builds an authorization request with a random state
 * parameter, saves it in the HTTP session, and redirects to the authorization URI.
 *
 * <h2>How it works</h2>
 * <pre>
 * GET /oauth2/authorization/google
 *   1. Extracts "google" as the registration ID
 *   2. Looks up ClientRegistration for "google"
 *   3. Generates a random state parameter
 *   4. Resolves the redirect URI (replaces {baseUrl} and {registrationId} templates)
 *   5. Builds an OAuth2AuthorizationRequest
 *   6. Saves the request in the HTTP session
 *   7. Redirects browser to: https://accounts.google.com/o/oauth2/v2/auth?response_type=code&client_id=...
 * </pre>
 *
 * <p>Simplifications vs. real OAuth2AuthorizationRequestRedirectFilter:
 * <ul>
 *   <li>No separate OAuth2AuthorizationRequestResolver — resolution is inline</li>
 *   <li>No separate AuthorizationRequestRepository — stores directly in session</li>
 *   <li>No request cache (original URL before redirect)</li>
 *   <li>No ClientAuthorizationRequiredException handling</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/web/OAuth2AuthorizationRequestRedirectFilter.java
 */
public class SimpleOAuth2AuthorizationRequestRedirectFilter implements Filter {

	/** The base URI that this filter matches against. */
	public static final String DEFAULT_AUTHORIZATION_REQUEST_BASE_URI = "/oauth2/authorization";

	/** Session attribute key for storing the authorization request. */
	static final String AUTHORIZATION_REQUEST_ATTR =
			"simple.security.oauth2.client.web.OAuth2AuthorizationRequest";

	private final ClientRegistrationRepository clientRegistrationRepository;

	private String authorizationRequestBaseUri = DEFAULT_AUTHORIZATION_REQUEST_BASE_URI;

	public SimpleOAuth2AuthorizationRequestRedirectFilter(
			ClientRegistrationRepository clientRegistrationRepository) {
		this.clientRegistrationRepository = clientRegistrationRepository;
	}

	@Override
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		String registrationId = resolveRegistrationId(request);
		if (registrationId == null) {
			chain.doFilter(request, response);
			return;
		}

		ClientRegistration clientRegistration =
				this.clientRegistrationRepository.findByRegistrationId(registrationId);
		if (clientRegistration == null) {
			response.sendError(HttpServletResponse.SC_BAD_REQUEST,
					"Unknown client registration: " + registrationId);
			return;
		}

		if (!AuthorizationGrantType.AUTHORIZATION_CODE.equals(
				clientRegistration.getAuthorizationGrantType())) {
			response.sendError(HttpServletResponse.SC_BAD_REQUEST,
					"Only authorization_code grant type is supported for login");
			return;
		}

		// Generate state and resolve redirect URI
		String state = UUID.randomUUID().toString();
		String redirectUri = resolveRedirectUri(request, clientRegistration);

		// Build the authorization request
		OAuth2AuthorizationRequest authorizationRequest = OAuth2AuthorizationRequest.create(
				clientRegistration.getAuthorizationUri(),
				clientRegistration.getClientId(),
				redirectUri,
				clientRegistration.getScopes(),
				state,
				registrationId);

		// Save in session for the callback filter to retrieve
		request.getSession().setAttribute(AUTHORIZATION_REQUEST_ATTR, authorizationRequest);

		// Redirect to the authorization server
		response.sendRedirect(authorizationRequest.getAuthorizationRequestUri());
	}

	/**
	 * Extracts the registration ID from the request URI.
	 * Returns null if the request doesn't match the authorization base URI.
	 */
	private String resolveRegistrationId(HttpServletRequest request) {
		String path = getRequestPath(request);
		if (!path.startsWith(this.authorizationRequestBaseUri + "/")) {
			return null;
		}
		return path.substring(this.authorizationRequestBaseUri.length() + 1);
	}

	/**
	 * Resolves the redirect URI by replacing template variables.
	 */
	private String resolveRedirectUri(HttpServletRequest request,
			ClientRegistration clientRegistration) {
		String redirectUri = clientRegistration.getRedirectUri();
		if (redirectUri == null) {
			return null;
		}
		String baseUrl = getBaseUrl(request);
		redirectUri = redirectUri.replace("{baseUrl}", baseUrl);
		redirectUri = redirectUri.replace("{registrationId}",
				clientRegistration.getRegistrationId());
		return redirectUri;
	}

	private String getBaseUrl(HttpServletRequest request) {
		String scheme = request.getScheme();
		String serverName = request.getServerName();
		int serverPort = request.getServerPort();
		String contextPath = request.getContextPath();

		StringBuilder baseUrl = new StringBuilder();
		baseUrl.append(scheme).append("://").append(serverName);
		if (("http".equals(scheme) && serverPort != 80)
				|| ("https".equals(scheme) && serverPort != 443)) {
			baseUrl.append(":").append(serverPort);
		}
		baseUrl.append(contextPath);
		return baseUrl.toString();
	}

	private static String getRequestPath(HttpServletRequest request) {
		String uri = request.getRequestURI();
		String contextPath = request.getContextPath();
		if (contextPath != null && !contextPath.isEmpty()) {
			uri = uri.substring(contextPath.length());
		}
		return uri;
	}

	public void setAuthorizationRequestBaseUri(String authorizationRequestBaseUri) {
		this.authorizationRequestBaseUri = authorizationRequestBaseUri;
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/web/SimpleOAuth2LoginAuthenticationFilter.java`

```java
package simple.security.oauth2.client.web;

import java.io.IOException;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.oauth2.client.authentication.OAuth2AuthenticationToken;
import simple.security.oauth2.client.authentication.OAuth2LoginAuthenticationToken;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.registration.ClientRegistrationRepository;
import simple.security.oauth2.core.OAuth2AuthenticationException;
import simple.security.oauth2.core.OAuth2Error;
import simple.security.oauth2.core.user.OAuth2User;
import simple.security.web.authentication.SimpleAbstractAuthenticationProcessingFilter;

/**
 * Filter that handles the OAuth2 callback — the redirect back from the authorization
 * server after the user grants (or denies) access.
 *
 * <p>Matches requests to {@code /login/oauth2/code/*}, retrieves the saved authorization
 * request from the session, builds an {@link OAuth2LoginAuthenticationToken}, and
 * delegates to the {@link simple.security.authentication.AuthenticationManager}.
 *
 * <h2>How it works</h2>
 * <pre>
 * GET /login/oauth2/code/google?code=abc123&state=xyz
 *   1. Extracts "google" as the registration ID from the URL path
 *   2. Retrieves the saved OAuth2AuthorizationRequest from the session
 *   3. Validates the state parameter matches
 *   4. Builds an OAuth2AuthorizationResponse from the query parameters
 *   5. Builds an OAuth2AuthorizationExchange (request + response)
 *   6. Creates an unauthenticated OAuth2LoginAuthenticationToken
 *   7. Delegates to AuthenticationManager.authenticate()
 *      → OAuth2LoginAuthenticationProvider exchanges code for tokens, loads user
 *   8. Converts to OAuth2AuthenticationToken (the final principal)
 *   9. Returns it → parent class stores in SecurityContext
 * </pre>
 *
 * <p>Simplifications vs. real OAuth2LoginAuthenticationFilter:
 * <ul>
 *   <li>No AuthorizationRequestRepository interface — reads directly from session</li>
 *   <li>No OAuth2AuthorizedClient saving</li>
 *   <li>No custom authentication result converter</li>
 *   <li>Registration ID extracted from URL path instead of saved authorization request</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/web/OAuth2LoginAuthenticationFilter.java
 */
public class SimpleOAuth2LoginAuthenticationFilter
		extends SimpleAbstractAuthenticationProcessingFilter {

	/** Default URL pattern for the OAuth2 callback. */
	public static final String DEFAULT_FILTER_PROCESSES_URI = "/login/oauth2/code/";

	private final ClientRegistrationRepository clientRegistrationRepository;

	public SimpleOAuth2LoginAuthenticationFilter(
			ClientRegistrationRepository clientRegistrationRepository) {
		super(request -> getRequestPath(request).startsWith(DEFAULT_FILTER_PROCESSES_URI));
		this.clientRegistrationRepository = clientRegistrationRepository;
	}

	@Override
	public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException, IOException, ServletException {

		// 1. Extract registration ID from URL path
		String path = getRequestPath(request);
		String registrationId = path.substring(DEFAULT_FILTER_PROCESSES_URI.length());
		if (registrationId.isEmpty()) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("invalid_request", "Missing registration ID in callback URL", null));
		}

		// 2. Look up the client registration
		ClientRegistration clientRegistration =
				this.clientRegistrationRepository.findByRegistrationId(registrationId);
		if (clientRegistration == null) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("client_registration_not_found",
							"Unknown registration ID: " + registrationId, null));
		}

		// 3. Retrieve the saved authorization request from session
		OAuth2AuthorizationRequest authorizationRequest =
				(OAuth2AuthorizationRequest) request.getSession()
						.getAttribute(SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR);
		if (authorizationRequest == null) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("authorization_request_not_found",
							"No saved authorization request found in session", null));
		}
		// Remove from session (one-time use)
		request.getSession().removeAttribute(
				SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR);

		// 4. Build the authorization response from query parameters
		String code = request.getParameter("code");
		String state = request.getParameter("state");
		String error = request.getParameter("error");
		String errorDescription = request.getParameter("error_description");

		OAuth2AuthorizationResponse authorizationResponse;
		if (error != null) {
			authorizationResponse = OAuth2AuthorizationResponse.error(error, errorDescription, state);
		}
		else if (code != null) {
			authorizationResponse = OAuth2AuthorizationResponse.success(code, state);
		}
		else {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("invalid_response",
							"Callback must contain either 'code' or 'error' parameter", null));
		}

		// 5. Validate state parameter
		if (authorizationRequest.getState() != null
				&& !authorizationRequest.getState().equals(state)) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("invalid_state",
							"State parameter mismatch: expected '"
									+ authorizationRequest.getState() + "' but got '" + state + "'",
							null));
		}

		// 6. Create the authorization exchange and unauthenticated token
		OAuth2AuthorizationExchange exchange =
				new OAuth2AuthorizationExchange(authorizationRequest, authorizationResponse);
		OAuth2LoginAuthenticationToken authenticationRequest =
				new OAuth2LoginAuthenticationToken(clientRegistration, exchange);

		// 7. Delegate to AuthenticationManager (→ OAuth2LoginAuthenticationProvider)
		OAuth2LoginAuthenticationToken authResult =
				(OAuth2LoginAuthenticationToken) getAuthenticationManager().authenticate(authenticationRequest);

		// 8. Convert to OAuth2AuthenticationToken (the final principal stored in SecurityContext)
		OAuth2User oauth2User = (OAuth2User) authResult.getPrincipal();
		return new OAuth2AuthenticationToken(
				oauth2User,
				oauth2User.getAuthorities(),
				clientRegistration.getRegistrationId());
	}

	private static String getRequestPath(HttpServletRequest request) {
		String uri = request.getRequestURI();
		String contextPath = request.getContextPath();
		if (contextPath != null && !contextPath.isEmpty()) {
			uri = uri.substring(contextPath.length());
		}
		return uri;
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/authentication/OAuth2LoginAuthenticationToken.java`

```java
package simple.security.oauth2.client.authentication;

import simple.security.authentication.AbstractAuthenticationToken;
import simple.security.core.GrantedAuthority;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.web.OAuth2AuthorizationExchange;
import simple.security.oauth2.core.OAuth2AccessToken;
import simple.security.oauth2.core.user.OAuth2User;

import java.util.Collection;

/**
 * Authentication token used during the OAuth2 login flow.
 *
 * <p>Has two states, following the dual-constructor pattern from
 * {@link simple.security.authentication.UsernamePasswordAuthenticationToken}:
 *
 * <ol>
 *   <li><b>Unauthenticated</b> — created by the login filter with the authorization
 *       exchange (code + state). Principal is {@code null}.</li>
 *   <li><b>Authenticated</b> — created by the authentication provider after exchanging
 *       the code for tokens and loading user info. Principal is the {@link OAuth2User}.</li>
 * </ol>
 *
 * <p>Simplifications vs. real OAuth2LoginAuthenticationToken:
 * <ul>
 *   <li>No refresh token support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.authentication.OAuth2LoginAuthenticationToken
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/authentication/OAuth2LoginAuthenticationToken.java
 */
public class OAuth2LoginAuthenticationToken extends AbstractAuthenticationToken {

	private final ClientRegistration clientRegistration;

	private final OAuth2AuthorizationExchange authorizationExchange;

	private final OAuth2User principal;

	private final OAuth2AccessToken accessToken;

	/**
	 * Creates an <b>unauthenticated</b> token with the authorization exchange.
	 * Used by the login filter to pass to the AuthenticationManager.
	 */
	public OAuth2LoginAuthenticationToken(ClientRegistration clientRegistration,
			OAuth2AuthorizationExchange authorizationExchange) {
		super(null);
		this.clientRegistration = clientRegistration;
		this.authorizationExchange = authorizationExchange;
		this.principal = null;
		this.accessToken = null;
		setAuthenticated(false);
	}

	/**
	 * Creates an <b>authenticated</b> token with user info and access token.
	 * Used by the authentication provider to return the successful result.
	 */
	public OAuth2LoginAuthenticationToken(ClientRegistration clientRegistration,
			OAuth2AuthorizationExchange authorizationExchange,
			OAuth2User principal,
			Collection<? extends GrantedAuthority> authorities,
			OAuth2AccessToken accessToken) {
		super(authorities);
		this.clientRegistration = clientRegistration;
		this.authorizationExchange = authorizationExchange;
		this.principal = principal;
		this.accessToken = accessToken;
		setAuthenticated(true);
	}

	@Override
	public Object getPrincipal() {
		return this.principal;
	}

	@Override
	public Object getCredentials() {
		return "";
	}

	public ClientRegistration getClientRegistration() {
		return this.clientRegistration;
	}

	public OAuth2AuthorizationExchange getAuthorizationExchange() {
		return this.authorizationExchange;
	}

	public OAuth2AccessToken getAccessToken() {
		return this.accessToken;
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/authentication/OAuth2AuthenticationToken.java`

```java
package simple.security.oauth2.client.authentication;

import simple.security.authentication.AbstractAuthenticationToken;
import simple.security.core.GrantedAuthority;
import simple.security.oauth2.core.user.OAuth2User;

import java.util.Collection;

/**
 * The final authentication token stored in the {@link simple.security.core.context.SecurityContext}
 * after a successful OAuth2 login.
 *
 * <p>This is distinct from {@link OAuth2LoginAuthenticationToken} which is only used
 * during the authentication process. This token carries:
 * <ul>
 *   <li>The {@link OAuth2User} principal (user attributes from the provider)</li>
 *   <li>The authorities</li>
 *   <li>The registration ID (which OAuth2 client was used to authenticate)</li>
 * </ul>
 *
 * <p>Simplifications vs. real OAuth2AuthenticationToken:
 * <ul>
 *   <li>No serialization support</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationToken
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/authentication/OAuth2AuthenticationToken.java
 */
public class OAuth2AuthenticationToken extends AbstractAuthenticationToken {

	private final OAuth2User principal;

	private final String authorizedClientRegistrationId;

	public OAuth2AuthenticationToken(OAuth2User principal,
			Collection<? extends GrantedAuthority> authorities,
			String authorizedClientRegistrationId) {
		super(authorities);
		this.principal = principal;
		this.authorizedClientRegistrationId = authorizedClientRegistrationId;
		setAuthenticated(true);
	}

	@Override
	public Object getPrincipal() {
		return this.principal;
	}

	@Override
	public Object getCredentials() {
		return "";
	}

	public String getAuthorizedClientRegistrationId() {
		return this.authorizedClientRegistrationId;
	}

	@Override
	public String getName() {
		return this.principal.getName();
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/authentication/OAuth2LoginAuthenticationProvider.java`

```java
package simple.security.oauth2.client.authentication;

import simple.security.authentication.AuthenticationProvider;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.core.GrantedAuthority;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.userinfo.OAuth2UserRequest;
import simple.security.oauth2.client.userinfo.OAuth2UserService;
import simple.security.oauth2.client.web.OAuth2AuthorizationExchange;
import simple.security.oauth2.client.web.OAuth2AuthorizationResponse;
import simple.security.oauth2.core.OAuth2AccessToken;
import simple.security.oauth2.core.OAuth2AuthenticationException;
import simple.security.oauth2.core.OAuth2Error;
import simple.security.oauth2.core.user.OAuth2User;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Collection;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * Authentication provider that processes the OAuth2 Authorization Code flow.
 *
 * <p>Performs two steps:
 * <ol>
 *   <li><b>Token exchange</b> — sends the authorization code to the token endpoint
 *       and receives an access token</li>
 *   <li><b>User loading</b> — delegates to {@link OAuth2UserService} to load user
 *       attributes from the UserInfo endpoint using the access token</li>
 * </ol>
 *
 * <p>Simplifications vs. real OAuth2LoginAuthenticationProvider:
 * <ul>
 *   <li>Token exchange uses {@link HttpURLConnection} instead of a dedicated
 *       OAuth2AccessTokenResponseClient</li>
 *   <li>Uses simple JSON parsing instead of Jackson</li>
 *   <li>No OIDC handling (no "openid" scope detection)</li>
 *   <li>No refresh token support</li>
 *   <li>No authorities mapper</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.authentication.OAuth2LoginAuthenticationProvider
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/authentication/OAuth2LoginAuthenticationProvider.java
 */
public class OAuth2LoginAuthenticationProvider implements AuthenticationProvider {

	private final OAuth2UserService userService;

	public OAuth2LoginAuthenticationProvider(OAuth2UserService userService) {
		this.userService = userService;
	}

	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		OAuth2LoginAuthenticationToken authToken = (OAuth2LoginAuthenticationToken) authentication;
		ClientRegistration clientRegistration = authToken.getClientRegistration();
		OAuth2AuthorizationExchange authorizationExchange = authToken.getAuthorizationExchange();

		// Validate the authorization response
		OAuth2AuthorizationResponse authorizationResponse = authorizationExchange.getAuthorizationResponse();
		if (!authorizationResponse.isSuccess()) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error(authorizationResponse.getError(),
							authorizationResponse.getErrorDescription(), null));
		}

		// Step 1: Exchange the authorization code for an access token
		OAuth2AccessToken accessToken = exchangeCodeForToken(clientRegistration,
				authorizationResponse.getCode(),
				authorizationExchange.getAuthorizationRequest().getRedirectUri());

		// Step 2: Load user info using the access token
		OAuth2UserRequest userRequest = new OAuth2UserRequest(clientRegistration, accessToken);
		OAuth2User oauth2User = this.userService.loadUser(userRequest);

		// Build the authenticated token
		Collection<? extends GrantedAuthority> authorities = oauth2User.getAuthorities();
		return new OAuth2LoginAuthenticationToken(
				clientRegistration, authorizationExchange, oauth2User,
				new ArrayList<>(authorities), accessToken);
	}

	@Override
	public boolean supports(Class<?> authentication) {
		return OAuth2LoginAuthenticationToken.class.isAssignableFrom(authentication);
	}

	/**
	 * Exchanges the authorization code for an access token by POSTing to the token endpoint.
	 */
	private OAuth2AccessToken exchangeCodeForToken(ClientRegistration clientRegistration,
			String code, String redirectUri) {
		try {
			HttpURLConnection connection = (HttpURLConnection) URI.create(
					clientRegistration.getTokenUri()).toURL().openConnection();
			connection.setRequestMethod("POST");
			connection.setDoOutput(true);
			connection.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");
			connection.setRequestProperty("Accept", "application/json");

			// Client authentication via Basic auth
			if (clientRegistration.getClientSecret() != null
					&& !clientRegistration.getClientSecret().isBlank()) {
				String credentials = clientRegistration.getClientId() + ":"
						+ clientRegistration.getClientSecret();
				String encoded = Base64.getEncoder().encodeToString(
						credentials.getBytes(StandardCharsets.UTF_8));
				connection.setRequestProperty("Authorization", "Basic " + encoded);
			}

			// Build form body
			StringBuilder body = new StringBuilder();
			body.append("grant_type=authorization_code");
			body.append("&code=").append(java.net.URLEncoder.encode(code, StandardCharsets.UTF_8));
			if (redirectUri != null) {
				body.append("&redirect_uri=").append(
						java.net.URLEncoder.encode(redirectUri, StandardCharsets.UTF_8));
			}

			try (OutputStream os = connection.getOutputStream()) {
				os.write(body.toString().getBytes(StandardCharsets.UTF_8));
			}

			int status = connection.getResponseCode();
			if (status != 200) {
				throw new OAuth2AuthenticationException(
						new OAuth2Error("token_exchange_error",
								"Token endpoint returned HTTP " + status, null));
			}

			String responseBody = readBody(connection);
			return parseTokenResponse(responseBody);
		}
		catch (OAuth2AuthenticationException ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("token_exchange_error",
							"Error exchanging authorization code: " + ex.getMessage(), null),
					ex);
		}
	}

	private OAuth2AccessToken parseTokenResponse(String json) {
		Map<String, Object> map = simple.security.oauth2.client.userinfo.DefaultOAuth2UserService
				.parseSimpleJson(json);

		String tokenValue = (String) map.get("access_token");
		if (tokenValue == null || tokenValue.isBlank()) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("invalid_token_response",
							"Missing access_token in token response", null));
		}

		Set<String> scopes = new HashSet<>();
		Object scopeValue = map.get("scope");
		if (scopeValue instanceof String scopeStr) {
			for (String s : scopeStr.split(" ")) {
				if (!s.isBlank()) {
					scopes.add(s.trim());
				}
			}
		}

		return new OAuth2AccessToken(tokenValue, scopes);
	}

	private String readBody(HttpURLConnection connection) throws Exception {
		try (BufferedReader reader = new BufferedReader(
				new InputStreamReader(connection.getInputStream(), StandardCharsets.UTF_8))) {
			StringBuilder sb = new StringBuilder();
			String line;
			while ((line = reader.readLine()) != null) {
				sb.append(line);
			}
			return sb.toString();
		}
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/userinfo/OAuth2UserRequest.java`

```java
package simple.security.oauth2.client.userinfo;

import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.core.OAuth2AccessToken;

/**
 * Contains the information needed to load user attributes from the UserInfo endpoint:
 * the client registration (which has the userInfoUri) and the access token.
 *
 * Mirrors: org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/userinfo/OAuth2UserRequest.java
 */
public class OAuth2UserRequest {

	private final ClientRegistration clientRegistration;

	private final OAuth2AccessToken accessToken;

	public OAuth2UserRequest(ClientRegistration clientRegistration, OAuth2AccessToken accessToken) {
		this.clientRegistration = clientRegistration;
		this.accessToken = accessToken;
	}

	public ClientRegistration getClientRegistration() {
		return this.clientRegistration;
	}

	public OAuth2AccessToken getAccessToken() {
		return this.accessToken;
	}

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/userinfo/OAuth2UserService.java`

```java
package simple.security.oauth2.client.userinfo;

import simple.security.oauth2.core.OAuth2AuthenticationException;
import simple.security.oauth2.core.user.OAuth2User;

/**
 * Loads user attributes from the OAuth2 provider's UserInfo endpoint.
 *
 * <p>This is a functional interface — implementations take an {@link OAuth2UserRequest}
 * (containing the client registration and access token) and return an {@link OAuth2User}
 * populated with the user's attributes.
 *
 * <p>The generic types on the real interface ({@code <R extends OAuth2UserRequest, U extends OAuth2User>})
 * are simplified here to concrete types since we don't support OIDC.
 *
 * Mirrors: org.springframework.security.oauth2.client.userinfo.OAuth2UserService
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/userinfo/OAuth2UserService.java
 */
@FunctionalInterface
public interface OAuth2UserService {

	/**
	 * Loads the user from the provider's UserInfo endpoint.
	 *
	 * @param userRequest the user request containing client registration and access token
	 * @return the authenticated OAuth2 user
	 * @throws OAuth2AuthenticationException if an error occurs while loading the user
	 */
	OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException;

}
```

### [NEW] `simple-security-oauth2-client/src/main/java/simple/security/oauth2/client/userinfo/DefaultOAuth2UserService.java`

```java
package simple.security.oauth2.client.userinfo;

import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.oauth2.core.OAuth2AuthenticationException;
import simple.security.oauth2.core.OAuth2Error;
import simple.security.oauth2.core.user.DefaultOAuth2User;
import simple.security.oauth2.core.user.OAuth2User;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Default implementation of {@link OAuth2UserService} that loads user attributes
 * by making an HTTP GET request to the provider's UserInfo endpoint.
 *
 * <p>The access token is sent as a Bearer token in the Authorization header.
 * The response is expected to be JSON. Attributes are parsed into a {@code Map}
 * and wrapped in a {@link DefaultOAuth2User}.
 *
 * <h2>Authority mapping</h2>
 * <p>Authorities are derived from the access token's scopes, prefixed with
 * {@code SCOPE_} (e.g., scope "email" becomes authority "SCOPE_email"). An
 * additional {@code OAUTH2_USER} authority is always added.
 *
 * <p>Simplifications vs. real DefaultOAuth2UserService:
 * <ul>
 *   <li>Uses {@link HttpURLConnection} instead of RestOperations</li>
 *   <li>Uses simple JSON string parsing instead of Jackson (for primitive key-value JSON only)</li>
 *   <li>No custom requestEntityConverter</li>
 *   <li>No custom attributesConverter</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService
 * Source:  oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/userinfo/DefaultOAuth2UserService.java
 */
public class DefaultOAuth2UserService implements OAuth2UserService {

	@Override
	public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
		String userInfoUri = userRequest.getClientRegistration().getUserInfoUri();
		if (userInfoUri == null || userInfoUri.isBlank()) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("missing_user_info_uri",
							"UserInfo URI is required but not configured for registration '"
									+ userRequest.getClientRegistration().getRegistrationId() + "'",
							null));
		}

		String userNameAttributeName = userRequest.getClientRegistration().getUserNameAttributeName();
		if (userNameAttributeName == null || userNameAttributeName.isBlank()) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("missing_user_name_attribute",
							"userNameAttributeName is required but not configured for registration '"
									+ userRequest.getClientRegistration().getRegistrationId() + "'",
							null));
		}

		Map<String, Object> attributes = fetchUserAttributes(userInfoUri,
				userRequest.getAccessToken().getTokenValue());

		List<GrantedAuthority> authorities = new ArrayList<>();
		authorities.add(new SimpleGrantedAuthority("OAUTH2_USER"));
		for (String scope : userRequest.getAccessToken().getScopes()) {
			authorities.add(new SimpleGrantedAuthority("SCOPE_" + scope));
		}

		return new DefaultOAuth2User(authorities, attributes, userNameAttributeName);
	}

	/**
	 * Fetches user attributes from the UserInfo endpoint via HTTP GET.
	 */
	private Map<String, Object> fetchUserAttributes(String userInfoUri, String accessToken) {
		try {
			HttpURLConnection connection = (HttpURLConnection) URI.create(userInfoUri).toURL().openConnection();
			connection.setRequestMethod("GET");
			connection.setRequestProperty("Authorization", "Bearer " + accessToken);
			connection.setRequestProperty("Accept", "application/json");

			int status = connection.getResponseCode();
			if (status != 200) {
				throw new OAuth2AuthenticationException(
						new OAuth2Error("userinfo_error",
								"UserInfo endpoint returned HTTP " + status, null));
			}

			String body = readBody(connection);
			return parseSimpleJson(body);
		}
		catch (OAuth2AuthenticationException ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new OAuth2AuthenticationException(
					new OAuth2Error("userinfo_error",
							"Error fetching user attributes: " + ex.getMessage(), null),
					ex);
		}
	}

	private String readBody(HttpURLConnection connection) throws Exception {
		try (BufferedReader reader = new BufferedReader(
				new InputStreamReader(connection.getInputStream(), StandardCharsets.UTF_8))) {
			StringBuilder sb = new StringBuilder();
			String line;
			while ((line = reader.readLine()) != null) {
				sb.append(line);
			}
			return sb.toString();
		}
	}

	/**
	 * Parses a flat JSON object into a Map. Handles strings, numbers, booleans, and null.
	 * This is intentionally simple — no nested objects or arrays.
	 */
	public static Map<String, Object> parseSimpleJson(String json) {
		Map<String, Object> map = new LinkedHashMap<>();
		json = json.trim();
		if (!json.startsWith("{") || !json.endsWith("}")) {
			return map;
		}
		json = json.substring(1, json.length() - 1).trim();
		if (json.isEmpty()) {
			return map;
		}

		int i = 0;
		while (i < json.length()) {
			// Skip whitespace
			while (i < json.length() && Character.isWhitespace(json.charAt(i))) i++;
			if (i >= json.length()) break;

			// Parse key
			if (json.charAt(i) != '"') break;
			int keyStart = i + 1;
			int keyEnd = json.indexOf('"', keyStart);
			if (keyEnd < 0) break;
			String key = json.substring(keyStart, keyEnd);
			i = keyEnd + 1;

			// Skip colon
			while (i < json.length() && (json.charAt(i) == ':' || Character.isWhitespace(json.charAt(i)))) i++;

			// Parse value
			Object value;
			if (i < json.length() && json.charAt(i) == '"') {
				// String value
				int valStart = i + 1;
				int valEnd = findUnescapedQuote(json, valStart);
				value = json.substring(valStart, valEnd);
				i = valEnd + 1;
			}
			else if (i < json.length() && (json.charAt(i) == 't' || json.charAt(i) == 'f')) {
				// Boolean
				if (json.startsWith("true", i)) {
					value = true;
					i += 4;
				}
				else {
					value = false;
					i += 5;
				}
			}
			else if (i < json.length() && json.charAt(i) == 'n') {
				// Null
				value = null;
				i += 4;
			}
			else {
				// Number
				int numStart = i;
				while (i < json.length() && json.charAt(i) != ',' && json.charAt(i) != '}' && !Character.isWhitespace(json.charAt(i))) i++;
				String numStr = json.substring(numStart, i);
				if (numStr.contains(".")) {
					value = Double.parseDouble(numStr);
				}
				else {
					value = Long.parseLong(numStr);
				}
			}

			map.put(key, value);

			// Skip comma
			while (i < json.length() && (json.charAt(i) == ',' || Character.isWhitespace(json.charAt(i)))) i++;
		}

		return map;
	}

	private static int findUnescapedQuote(String s, int from) {
		for (int i = from; i < s.length(); i++) {
			if (s.charAt(i) == '"' && (i == 0 || s.charAt(i - 1) != '\\')) {
				return i;
			}
		}
		return s.length();
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/oauth2/client/SimpleOAuth2LoginConfigurer.java`

```java
package simple.security.config.annotation.web.configurers.oauth2.client;

import simple.security.authentication.AuthenticationManager;
import simple.security.authentication.AuthenticationProvider;
import simple.security.authentication.SimpleProviderManager;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.config.annotation.web.configurers.SimpleAbstractHttpConfigurer;
import simple.security.oauth2.client.authentication.OAuth2LoginAuthenticationProvider;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.registration.ClientRegistrationRepository;
import simple.security.oauth2.client.registration.SimpleInMemoryClientRegistrationRepository;
import simple.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import simple.security.oauth2.client.userinfo.OAuth2UserService;
import simple.security.oauth2.client.web.SimpleOAuth2AuthorizationRequestRedirectFilter;
import simple.security.oauth2.client.web.SimpleOAuth2LoginAuthenticationFilter;
import simple.security.web.AuthenticationEntryPoint;
import simple.security.web.authentication.AuthenticationFailureHandler;
import simple.security.web.authentication.AuthenticationSuccessHandler;
import simple.security.web.authentication.SimpleLoginUrlAuthenticationEntryPoint;
import simple.security.web.authentication.SimpleRedirectAuthenticationFailureHandler;
import simple.security.web.authentication.SimpleRedirectAuthenticationSuccessHandler;
import simple.security.web.context.SecurityContextRepository;

import java.util.List;

/**
 * Configures OAuth2 Login via the {@link SimpleHttpSecurity} DSL.
 *
 * <h2>Usage</h2>
 * <pre>
 * http.oauth2Login(oauth2 -> oauth2
 *     .clientRegistrationRepository(clientRegistrationRepository)
 *     .loginPage("/login")
 *     .defaultSuccessUrl("/home")
 *     .userInfoEndpoint(userInfo -> userInfo
 *         .userService(customOAuth2UserService)));
 * </pre>
 *
 * <h2>What it does</h2>
 * <ol>
 *   <li>{@code init()} — Registers the authentication entry point, creates and registers
 *       the {@link OAuth2LoginAuthenticationProvider}</li>
 *   <li>{@code configure()} — Creates and adds two filters to the chain:
 *       <ul>
 *         <li>{@link SimpleOAuth2AuthorizationRequestRedirectFilter} — redirects to auth server</li>
 *         <li>{@link SimpleOAuth2LoginAuthenticationFilter} — handles the callback</li>
 *       </ul>
 *   </li>
 * </ol>
 *
 * <p>Simplifications vs. real OAuth2LoginConfigurer:
 * <ul>
 *   <li>No auto-generated login page with OAuth2 login links</li>
 *   <li>No OIDC support</li>
 *   <li>No OAuth2AuthorizedClientRepository</li>
 *   <li>Doesn't extend AbstractAuthenticationFilterConfigurer</li>
 * </ul>
 *
 * Mirrors: org.springframework.security.config.annotation.web.configurers.oauth2.client.OAuth2LoginConfigurer
 * Source:  config/src/main/java/org/springframework/security/config/annotation/web/configurers/oauth2/client/OAuth2LoginConfigurer.java
 */
public class SimpleOAuth2LoginConfigurer extends SimpleAbstractHttpConfigurer<SimpleOAuth2LoginConfigurer> {

	private ClientRegistrationRepository clientRegistrationRepository;

	private String loginPage = "/login";

	private String defaultSuccessUrl = "/";

	private String failureUrl;

	private AuthenticationSuccessHandler successHandler;

	private AuthenticationFailureHandler failureHandler;

	private AuthenticationEntryPoint authenticationEntryPoint;

	private UserInfoEndpointConfig userInfoEndpointConfig;

	// === Lifecycle ===

	/**
	 * Resolves defaults and registers the authentication entry point and provider.
	 */
	@Override
	public void init(SimpleHttpSecurity builder) throws Exception {
		if (this.failureUrl == null) {
			this.failureUrl = this.loginPage + "?error";
		}
		if (this.authenticationEntryPoint == null) {
			this.authenticationEntryPoint = new SimpleLoginUrlAuthenticationEntryPoint(this.loginPage);
		}

		// Register entry point so ExceptionTranslationFilter can use it
		builder.setSharedObject(AuthenticationEntryPoint.class, this.authenticationEntryPoint);

		// Register the OAuth2 login authentication provider
		OAuth2UserService userService = getUserService();
		AuthenticationProvider provider = new OAuth2LoginAuthenticationProvider(userService);
		builder.authenticationProvider(provider);
	}

	/**
	 * Creates and adds the redirect filter and login authentication filter.
	 */
	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception {
		ClientRegistrationRepository repository = getClientRegistrationRepository(builder);

		// 1. Create and add the redirect filter (before the login filter)
		SimpleOAuth2AuthorizationRequestRedirectFilter redirectFilter =
				new SimpleOAuth2AuthorizationRequestRedirectFilter(repository);
		builder.addFilterBefore(redirectFilter,
				simple.security.web.authentication.SimpleUsernamePasswordAuthenticationFilter.class);

		// 2. Create and configure the login filter
		SimpleOAuth2LoginAuthenticationFilter loginFilter =
				new SimpleOAuth2LoginAuthenticationFilter(repository);

		// Build a dedicated AuthenticationManager with only the OAuth2 provider
		OAuth2UserService userService = getUserService();
		AuthenticationProvider provider = new OAuth2LoginAuthenticationProvider(userService);
		AuthenticationManager oauthAuthManager = new SimpleProviderManager(List.of(provider));
		loginFilter.setAuthenticationManager(oauthAuthManager);

		// Wire success handler
		if (this.successHandler != null) {
			loginFilter.setAuthenticationSuccessHandler(this.successHandler);
		}
		else {
			loginFilter.setAuthenticationSuccessHandler(
					new SimpleRedirectAuthenticationSuccessHandler(this.defaultSuccessUrl));
		}

		// Wire failure handler
		if (this.failureHandler != null) {
			loginFilter.setAuthenticationFailureHandler(this.failureHandler);
		}
		else {
			loginFilter.setAuthenticationFailureHandler(
					new SimpleRedirectAuthenticationFailureHandler(this.failureUrl));
		}

		// Wire SecurityContextRepository so the filter explicitly saves the context
		SecurityContextRepository contextRepository =
				builder.getSharedObject(SecurityContextRepository.class);
		if (contextRepository != null) {
			loginFilter.setSecurityContextRepository(contextRepository);
		}

		builder.addFilterAfter(loginFilter,
				SimpleOAuth2AuthorizationRequestRedirectFilter.class);
	}

	// === DSL methods ===

	/**
	 * Sets the {@link ClientRegistrationRepository} containing all OAuth2 client
	 * registrations.
	 */
	public SimpleOAuth2LoginConfigurer clientRegistrationRepository(
			ClientRegistrationRepository clientRegistrationRepository) {
		this.clientRegistrationRepository = clientRegistrationRepository;
		return this;
	}

	/**
	 * Sets the login page URL. When authentication is required and no user is logged in,
	 * the entry point redirects here.
	 */
	public SimpleOAuth2LoginConfigurer loginPage(String loginPage) {
		this.loginPage = loginPage;
		return this;
	}

	/**
	 * Sets the URL to redirect to after successful OAuth2 login.
	 */
	public SimpleOAuth2LoginConfigurer defaultSuccessUrl(String defaultSuccessUrl) {
		this.defaultSuccessUrl = defaultSuccessUrl;
		return this;
	}

	/**
	 * Sets the URL to redirect to after OAuth2 login failure.
	 */
	public SimpleOAuth2LoginConfigurer failureUrl(String failureUrl) {
		this.failureUrl = failureUrl;
		return this;
	}

	/**
	 * Sets a custom success handler.
	 */
	public SimpleOAuth2LoginConfigurer successHandler(AuthenticationSuccessHandler successHandler) {
		this.successHandler = successHandler;
		return this;
	}

	/**
	 * Sets a custom failure handler.
	 */
	public SimpleOAuth2LoginConfigurer failureHandler(AuthenticationFailureHandler failureHandler) {
		this.failureHandler = failureHandler;
		return this;
	}

	/**
	 * Sets a custom authentication entry point.
	 */
	public SimpleOAuth2LoginConfigurer authenticationEntryPoint(AuthenticationEntryPoint entryPoint) {
		this.authenticationEntryPoint = entryPoint;
		return this;
	}

	/**
	 * Configures the UserInfo endpoint — specifically, the {@link OAuth2UserService}
	 * used to load user attributes after the token exchange.
	 */
	public SimpleOAuth2LoginConfigurer userInfoEndpoint(
			simple.security.config.Customizer<UserInfoEndpointConfig> customizer) {
		if (this.userInfoEndpointConfig == null) {
			this.userInfoEndpointConfig = new UserInfoEndpointConfig();
		}
		customizer.customize(this.userInfoEndpointConfig);
		return this;
	}

	// === Internal ===

	private ClientRegistrationRepository getClientRegistrationRepository(SimpleHttpSecurity builder) {
		if (this.clientRegistrationRepository != null) {
			return this.clientRegistrationRepository;
		}
		// Try to get from shared objects (e.g., set by auto-configuration)
		ClientRegistrationRepository shared =
				builder.getSharedObject(ClientRegistrationRepository.class);
		if (shared != null) {
			return shared;
		}
		throw new IllegalStateException(
				"A ClientRegistrationRepository must be configured. "
						+ "Call .clientRegistrationRepository(repository) on the OAuth2 login configurer, "
						+ "or set it as a shared object.");
	}

	private OAuth2UserService getUserService() {
		if (this.userInfoEndpointConfig != null && this.userInfoEndpointConfig.userService != null) {
			return this.userInfoEndpointConfig.userService;
		}
		return new DefaultOAuth2UserService();
	}

	// === UserInfo sub-configurer ===

	/**
	 * Configuration for the UserInfo endpoint within OAuth2 Login.
	 *
	 * <pre>
	 * http.oauth2Login(oauth2 -> oauth2
	 *     .userInfoEndpoint(userInfo -> userInfo
	 *         .userService(customOAuth2UserService)));
	 * </pre>
	 */
	public static class UserInfoEndpointConfig {

		private OAuth2UserService userService;

		/**
		 * Sets a custom {@link OAuth2UserService} for loading user attributes
		 * from the provider's UserInfo endpoint.
		 */
		public UserInfoEndpointConfig userService(OAuth2UserService userService) {
			this.userService = userService;
			return this;
		}

	}

}
```

### [MODIFIED] `simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

Added `SimpleOAuth2AuthorizationRequestRedirectFilter` to filter order map and `oauth2Login()` DSL method:

```java
package simple.security.config.annotation.web.builders;

// ... (existing imports) ...
import simple.security.config.annotation.web.configurers.oauth2.client.SimpleOAuth2LoginConfigurer;

public final class SimpleHttpSecurity implements SecurityBuilder<SecurityFilterChain> {

    private static final Map<String, Integer> FILTER_ORDER_MAP = new LinkedHashMap<>();

    static {
        int order = 100;
        FILTER_ORDER_MAP.put("SimpleSecurityContextHolderFilter", order);       // 100
        order += 100;
        FILTER_ORDER_MAP.put("SimpleCsrfFilter", order);                        // 200
        order += 50;
        FILTER_ORDER_MAP.put("SimpleLogoutFilter", order);                      // 250
        order += 20;
        FILTER_ORDER_MAP.put("SimpleOAuth2AuthorizationRequestRedirectFilter", order); // 270  <-- NEW
        order += 30;
        FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", order);     // 300
        order += 100;
        FILTER_ORDER_MAP.put("SimpleBasicAuthenticationFilter", order);         // 400
        order += 50;
        FILTER_ORDER_MAP.put("SimpleBearerTokenAuthenticationFilter", order);   // 450
        order += 50;
        FILTER_ORDER_MAP.put("SimpleExceptionTranslationFilter", order);        // 500
        order += 100;
        FILTER_ORDER_MAP.put("SimpleAuthorizationFilter", order);               // 600
    }

    // ... (existing fields and methods unchanged) ...

    /**
     * Configures OAuth2 Login for "Login with Google/GitHub/etc." functionality.
     */
    public SimpleHttpSecurity oauth2Login(Customizer<SimpleOAuth2LoginConfigurer> oauth2LoginCustomizer) {
        oauth2LoginCustomizer.customize(getOrApply(new SimpleOAuth2LoginConfigurer()));
        return this;
    }

    // ... (rest of existing methods unchanged) ...
}
```

### [MODIFIED] `simple-security-config/build.gradle`

Added OAuth2 client module dependency:

```groovy
// simple-security-config — @EnableWebSecurity, HttpSecurity DSL, configurers
// Mirrors: spring-security-config

dependencies {
    api project(':simple-security-core')
    api project(':simple-security-web')
    implementation project(':simple-security-oauth2-jose')
    implementation project(':simple-security-oauth2-resource-server')
    implementation project(':simple-security-oauth2-client')                    // <-- NEW
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

### [MODIFIED] `simple-security-oauth2-client/build.gradle`

```groovy
// simple-security-oauth2-client — OAuth2 Login / Client support
// Mirrors: spring-security-oauth2-client

dependencies {
    api project(':simple-security-core')
    api project(':simple-security-oauth2-core')
    api project(':simple-security-web')

    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    testImplementation 'jakarta.servlet:jakarta.servlet-api:6.1.0'
    testImplementation 'org.springframework:spring-test:6.2.3'
    testImplementation 'org.springframework:spring-web:6.2.3'
}
```

### [NEW] `simple-security-oauth2-core/src/test/java/simple/security/oauth2/core/AuthorizationGrantTypeTest.java`

```java
package simple.security.oauth2.core;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests for {@link AuthorizationGrantType}.
 */
class AuthorizationGrantTypeTest {

	@Test
	void shouldHaveAuthorizationCodeConstant() {
		assertEquals("authorization_code", AuthorizationGrantType.AUTHORIZATION_CODE.getValue());
	}

	@Test
	void shouldHaveClientCredentialsConstant() {
		assertEquals("client_credentials", AuthorizationGrantType.CLIENT_CREDENTIALS.getValue());
	}

	@Test
	void shouldBeEqualWhenSameValue() {
		AuthorizationGrantType a = new AuthorizationGrantType("custom");
		AuthorizationGrantType b = new AuthorizationGrantType("custom");
		assertEquals(a, b);
		assertEquals(a.hashCode(), b.hashCode());
	}

	@Test
	void shouldNotBeEqualWhenDifferentValue() {
		assertNotEquals(AuthorizationGrantType.AUTHORIZATION_CODE,
				AuthorizationGrantType.CLIENT_CREDENTIALS);
	}

	@Test
	void shouldRejectBlankValue() {
		assertThrows(IllegalArgumentException.class,
				() -> new AuthorizationGrantType(""));
	}

	@Test
	void shouldRejectNullValue() {
		assertThrows(IllegalArgumentException.class,
				() -> new AuthorizationGrantType(null));
	}

}
```

### [NEW] `simple-security-oauth2-core/src/test/java/simple/security/oauth2/core/user/DefaultOAuth2UserTest.java`

```java
package simple.security.oauth2.core.user;

import org.junit.jupiter.api.Test;

import simple.security.core.GrantedAuthority;
import simple.security.core.authority.SimpleGrantedAuthority;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests for {@link DefaultOAuth2User}.
 */
class DefaultOAuth2UserTest {

	@Test
	void shouldReturnNameFromAttributes() {
		DefaultOAuth2User user = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				Map.of("sub", "user123", "name", "John"),
				"sub");

		assertEquals("user123", user.getName());
	}

	@Test
	void shouldReturnUnmodifiableAttributes() {
		DefaultOAuth2User user = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				Map.of("sub", "user123"),
				"sub");

		assertThrows(UnsupportedOperationException.class,
				() -> user.getAttributes().put("extra", "value"));
	}

	@Test
	void shouldReturnUnmodifiableAuthorities() {
		List<GrantedAuthority> authorities = List.of(
				new SimpleGrantedAuthority("OAUTH2_USER"),
				new SimpleGrantedAuthority("SCOPE_email"));
		DefaultOAuth2User user = new DefaultOAuth2User(
				authorities,
				Map.of("sub", "user123"),
				"sub");

		assertEquals(2, user.getAuthorities().size());
	}

	@Test
	void shouldRejectEmptyAttributes() {
		assertThrows(IllegalArgumentException.class,
				() -> new DefaultOAuth2User(List.of(), Map.of(), "sub"));
	}

	@Test
	void shouldRejectMissingNameAttribute() {
		assertThrows(IllegalArgumentException.class,
				() -> new DefaultOAuth2User(List.of(),
						Map.of("email", "john@example.com"), "sub"));
	}

	@Test
	void shouldRejectBlankNameAttributeKey() {
		assertThrows(IllegalArgumentException.class,
				() -> new DefaultOAuth2User(List.of(),
						Map.of("sub", "user123"), ""));
	}

	@Test
	void shouldBeEqualWhenSameData() {
		DefaultOAuth2User user1 = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				Map.of("sub", "user123"), "sub");
		DefaultOAuth2User user2 = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				Map.of("sub", "user123"), "sub");

		assertEquals(user1, user2);
		assertEquals(user1.hashCode(), user2.hashCode());
	}

}
```

### [NEW] `simple-security-oauth2-client/src/test/java/simple/security/oauth2/client/registration/ClientRegistrationTest.java`

```java
package simple.security.oauth2.client.registration;

import org.junit.jupiter.api.Test;

import simple.security.oauth2.core.AuthorizationGrantType;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests for {@link ClientRegistration} and its {@link ClientRegistration.Builder}.
 */
class ClientRegistrationTest {

	@Test
	void shouldBuildValidClientRegistration() {
		ClientRegistration registration = ClientRegistration.withRegistrationId("google")
				.clientId("google-client-id")
				.clientSecret("google-client-secret")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
				.scope("openid", "profile", "email")
				.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
				.tokenUri("https://oauth2.googleapis.com/token")
				.userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
				.userNameAttributeName("sub")
				.clientName("Google")
				.build();

		assertEquals("google", registration.getRegistrationId());
		assertEquals("google-client-id", registration.getClientId());
		assertEquals("google-client-secret", registration.getClientSecret());
		assertEquals(AuthorizationGrantType.AUTHORIZATION_CODE, registration.getAuthorizationGrantType());
		assertEquals("{baseUrl}/login/oauth2/code/{registrationId}", registration.getRedirectUri());
		assertEquals(3, registration.getScopes().size());
		assertTrue(registration.getScopes().contains("openid"));
		assertEquals("https://accounts.google.com/o/oauth2/v2/auth", registration.getAuthorizationUri());
		assertEquals("https://oauth2.googleapis.com/token", registration.getTokenUri());
		assertEquals("Google", registration.getClientName());
	}

	@Test
	void shouldDefaultClientNameToRegistrationId() {
		ClientRegistration registration = ClientRegistration.withRegistrationId("github")
				.clientId("github-client-id")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("http://localhost/callback")
				.authorizationUri("https://github.com/login/oauth/authorize")
				.tokenUri("https://github.com/login/oauth/access_token")
				.build();

		assertEquals("github", registration.getClientName());
	}

	@Test
	void shouldRejectMissingClientId() {
		assertThrows(IllegalArgumentException.class, () ->
				ClientRegistration.withRegistrationId("test")
						.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
						.redirectUri("http://localhost/callback")
						.authorizationUri("https://example.com/auth")
						.tokenUri("https://example.com/token")
						.build());
	}

	@Test
	void shouldRejectMissingRedirectUriForAuthCode() {
		assertThrows(IllegalArgumentException.class, () ->
				ClientRegistration.withRegistrationId("test")
						.clientId("client-id")
						.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
						.authorizationUri("https://example.com/auth")
						.tokenUri("https://example.com/token")
						.build());
	}

	@Test
	void shouldRejectMissingAuthorizationUri() {
		assertThrows(IllegalArgumentException.class, () ->
				ClientRegistration.withRegistrationId("test")
						.clientId("client-id")
						.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
						.redirectUri("http://localhost/callback")
						.tokenUri("https://example.com/token")
						.build());
	}

	@Test
	void shouldRejectMissingTokenUri() {
		assertThrows(IllegalArgumentException.class, () ->
				ClientRegistration.withRegistrationId("test")
						.clientId("client-id")
						.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
						.redirectUri("http://localhost/callback")
						.authorizationUri("https://example.com/auth")
						.build());
	}

	@Test
	void shouldReturnUnmodifiableScopes() {
		ClientRegistration registration = ClientRegistration.withRegistrationId("test")
				.clientId("client-id")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("http://localhost/callback")
				.authorizationUri("https://example.com/auth")
				.tokenUri("https://example.com/token")
				.scope("read", "write")
				.build();

		assertThrows(UnsupportedOperationException.class,
				() -> registration.getScopes().add("admin"));
	}

}
```

### [NEW] `simple-security-oauth2-client/src/test/java/simple/security/oauth2/client/registration/SimpleInMemoryClientRegistrationRepositoryTest.java`

```java
package simple.security.oauth2.client.registration;

import org.junit.jupiter.api.Test;

import simple.security.oauth2.core.AuthorizationGrantType;

import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests for {@link SimpleInMemoryClientRegistrationRepository}.
 */
class SimpleInMemoryClientRegistrationRepositoryTest {

	@Test
	void shouldFindByRegistrationId() {
		ClientRegistration google = createRegistration("google");
		ClientRegistration github = createRegistration("github");

		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(google, github);

		assertSame(google, repository.findByRegistrationId("google"));
		assertSame(github, repository.findByRegistrationId("github"));
	}

	@Test
	void shouldReturnNullForUnknownRegistrationId() {
		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(createRegistration("google"));

		assertNull(repository.findByRegistrationId("unknown"));
	}

	@Test
	void shouldBeIterable() {
		ClientRegistration google = createRegistration("google");
		ClientRegistration github = createRegistration("github");

		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(google, github);

		List<ClientRegistration> all = new ArrayList<>();
		repository.forEach(all::add);

		assertEquals(2, all.size());
		assertTrue(all.contains(google));
		assertTrue(all.contains(github));
	}

	@Test
	void shouldRejectDuplicateRegistrationIds() {
		ClientRegistration reg1 = createRegistration("google");
		ClientRegistration reg2 = createRegistration("google");

		assertThrows(IllegalStateException.class,
				() -> new SimpleInMemoryClientRegistrationRepository(reg1, reg2));
	}

	private ClientRegistration createRegistration(String registrationId) {
		return ClientRegistration.withRegistrationId(registrationId)
				.clientId(registrationId + "-client-id")
				.clientSecret(registrationId + "-secret")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
				.authorizationUri("https://example.com/" + registrationId + "/auth")
				.tokenUri("https://example.com/" + registrationId + "/token")
				.userInfoUri("https://example.com/" + registrationId + "/userinfo")
				.userNameAttributeName("sub")
				.build();
	}

}
```

### [NEW] `simple-security-oauth2-client/src/test/java/simple/security/oauth2/client/web/SimpleOAuth2AuthorizationRequestRedirectFilterTest.java`

```java
package simple.security.oauth2.client.web;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import jakarta.servlet.FilterChain;

import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.registration.SimpleInMemoryClientRegistrationRepository;
import simple.security.oauth2.core.AuthorizationGrantType;

import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Tests for {@link SimpleOAuth2AuthorizationRequestRedirectFilter}.
 */
class SimpleOAuth2AuthorizationRequestRedirectFilterTest {

	private SimpleOAuth2AuthorizationRequestRedirectFilter filter;

	private ClientRegistration googleRegistration;

	@BeforeEach
	void setUp() {
		this.googleRegistration = ClientRegistration.withRegistrationId("google")
				.clientId("google-client-id")
				.clientSecret("google-secret")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
				.scope("openid", "profile")
				.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
				.tokenUri("https://oauth2.googleapis.com/token")
				.userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
				.userNameAttributeName("sub")
				.build();

		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(this.googleRegistration);
		this.filter = new SimpleOAuth2AuthorizationRequestRedirectFilter(repository);
	}

	@Test
	void shouldRedirectToAuthorizationEndpoint_WhenRequestMatchesBaseUri() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/oauth2/authorization/google");
		request.setServerName("localhost");
		request.setServerPort(8080);
		request.setScheme("http");
		MockHttpServletResponse response = new MockHttpServletResponse();
		FilterChain chain = mock(FilterChain.class);

		this.filter.doFilter(request, response, chain);

		// Should redirect, not continue chain
		assertNotNull(response.getRedirectedUrl());
		assertTrue(response.getRedirectedUrl().startsWith(
				"https://accounts.google.com/o/oauth2/v2/auth?"));
		assertTrue(response.getRedirectedUrl().contains("response_type=code"));
		assertTrue(response.getRedirectedUrl().contains("client_id=google-client-id"));
		assertTrue(response.getRedirectedUrl().contains("state="));
		verify(chain, never()).doFilter(request, response);
	}

	@Test
	void shouldSaveAuthorizationRequestInSession_WhenRedirecting() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/oauth2/authorization/google");
		request.setServerName("localhost");
		request.setServerPort(8080);
		request.setScheme("http");
		MockHttpServletResponse response = new MockHttpServletResponse();
		FilterChain chain = mock(FilterChain.class);

		this.filter.doFilter(request, response, chain);

		Object saved = request.getSession().getAttribute(
				SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR);
		assertNotNull(saved);
		assertInstanceOf(OAuth2AuthorizationRequest.class, saved);

		OAuth2AuthorizationRequest savedRequest = (OAuth2AuthorizationRequest) saved;
		assertEquals("google", savedRequest.getRegistrationId());
		assertEquals("google-client-id", savedRequest.getClientId());
		assertNotNull(savedRequest.getState());
	}

	@Test
	void shouldResolveRedirectUriTemplateVariables() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/oauth2/authorization/google");
		request.setServerName("myapp.com");
		request.setServerPort(443);
		request.setScheme("https");
		MockHttpServletResponse response = new MockHttpServletResponse();
		FilterChain chain = mock(FilterChain.class);

		this.filter.doFilter(request, response, chain);

		OAuth2AuthorizationRequest savedRequest = (OAuth2AuthorizationRequest) request.getSession()
				.getAttribute(SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR);
		assertEquals("https://myapp.com/login/oauth2/code/google",
				savedRequest.getRedirectUri());
	}

	@Test
	void shouldPassThroughForNonMatchingUrls() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		MockHttpServletResponse response = new MockHttpServletResponse();
		FilterChain chain = mock(FilterChain.class);

		this.filter.doFilter(request, response, chain);

		verify(chain).doFilter(request, response);
		assertNull(response.getRedirectedUrl());
	}

	@Test
	void shouldReturn400ForUnknownRegistrationId() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/oauth2/authorization/unknown");
		MockHttpServletResponse response = new MockHttpServletResponse();
		FilterChain chain = mock(FilterChain.class);

		this.filter.doFilter(request, response, chain);

		assertEquals(400, response.getStatus());
		verify(chain, never()).doFilter(request, response);
	}

}
```

### [NEW] `simple-security-oauth2-client/src/test/java/simple/security/oauth2/client/web/SimpleOAuth2LoginAuthenticationFilterTest.java`

```java
package simple.security.oauth2.client.web;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import jakarta.servlet.FilterChain;

import simple.security.authentication.AuthenticationManager;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.oauth2.client.authentication.OAuth2AuthenticationToken;
import simple.security.oauth2.client.authentication.OAuth2LoginAuthenticationToken;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.registration.SimpleInMemoryClientRegistrationRepository;
import simple.security.oauth2.core.AuthorizationGrantType;
import simple.security.oauth2.core.OAuth2AccessToken;
import simple.security.oauth2.core.user.DefaultOAuth2User;

import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

/**
 * Tests for {@link SimpleOAuth2LoginAuthenticationFilter}.
 */
class SimpleOAuth2LoginAuthenticationFilterTest {

	private SimpleOAuth2LoginAuthenticationFilter filter;

	private AuthenticationManager authenticationManager;

	private ClientRegistration googleRegistration;

	@BeforeEach
	void setUp() {
		this.googleRegistration = ClientRegistration.withRegistrationId("google")
				.clientId("google-client-id")
				.clientSecret("google-secret")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
				.scope("openid", "profile")
				.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
				.tokenUri("https://oauth2.googleapis.com/token")
				.userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
				.userNameAttributeName("sub")
				.build();

		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(this.googleRegistration);
		this.filter = new SimpleOAuth2LoginAuthenticationFilter(repository);

		this.authenticationManager = mock(AuthenticationManager.class);
		this.filter.setAuthenticationManager(this.authenticationManager);
	}

	@Test
	void shouldAuthenticate_WhenCallbackContainsCodeAndState() throws Exception {
		// Simulate a saved authorization request (as if redirect filter already ran)
		String savedState = "test-state-123";
		OAuth2AuthorizationRequest savedRequest = OAuth2AuthorizationRequest.create(
				"https://accounts.google.com/o/oauth2/v2/auth",
				"google-client-id",
				"http://localhost/login/oauth2/code/google",
				Set.of("openid", "profile"),
				savedState,
				"google");

		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/login/oauth2/code/google");
		request.setParameter("code", "auth-code-xyz");
		request.setParameter("state", savedState);
		request.getSession().setAttribute(
				SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR,
				savedRequest);

		MockHttpServletResponse response = new MockHttpServletResponse();

		// Mock the authentication manager to return an authenticated token
		DefaultOAuth2User oauth2User = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER"),
						new SimpleGrantedAuthority("SCOPE_openid")),
				Map.of("sub", "user123", "name", "John Doe"),
				"sub");

		when(this.authenticationManager.authenticate(any())).thenAnswer(invocation -> {
			OAuth2LoginAuthenticationToken inputToken = invocation.getArgument(0);
			return new OAuth2LoginAuthenticationToken(
					inputToken.getClientRegistration(),
					inputToken.getAuthorizationExchange(),
					oauth2User,
					List.of(new SimpleGrantedAuthority("OAUTH2_USER"),
							new SimpleGrantedAuthority("SCOPE_openid")),
					new OAuth2AccessToken("access-token-value", Set.of("openid", "profile")));
		});

		// Execute the filter — the abstract base class handles SecurityContext + success handler
		Authentication result = this.filter.attemptAuthentication(request, response);

		// Verify: returns an OAuth2AuthenticationToken
		assertNotNull(result);
		assertInstanceOf(OAuth2AuthenticationToken.class, result);
		OAuth2AuthenticationToken authToken = (OAuth2AuthenticationToken) result;
		assertEquals("user123", authToken.getName());
		assertEquals("google", authToken.getAuthorizedClientRegistrationId());
		assertTrue(authToken.isAuthenticated());

		// Verify: authorization request removed from session
		assertNull(request.getSession().getAttribute(
				SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR));
	}

	@Test
	void shouldRejectCallback_WhenStateMismatch() throws Exception {
		OAuth2AuthorizationRequest savedRequest = OAuth2AuthorizationRequest.create(
				"https://accounts.google.com/o/oauth2/v2/auth",
				"google-client-id",
				"http://localhost/login/oauth2/code/google",
				Set.of("openid"),
				"expected-state",
				"google");

		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/login/oauth2/code/google");
		request.setParameter("code", "auth-code-xyz");
		request.setParameter("state", "wrong-state");
		request.getSession().setAttribute(
				SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR,
				savedRequest);

		MockHttpServletResponse response = new MockHttpServletResponse();

		assertThrows(simple.security.oauth2.core.OAuth2AuthenticationException.class,
				() -> this.filter.attemptAuthentication(request, response));
	}

	@Test
	void shouldRejectCallback_WhenNoSavedAuthorizationRequest() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/login/oauth2/code/google");
		request.setParameter("code", "auth-code-xyz");
		request.setParameter("state", "some-state");
		MockHttpServletResponse response = new MockHttpServletResponse();

		assertThrows(simple.security.oauth2.core.OAuth2AuthenticationException.class,
				() -> this.filter.attemptAuthentication(request, response));
	}

	@Test
	void shouldHandleErrorResponseFromProvider() throws Exception {
		OAuth2AuthorizationRequest savedRequest = OAuth2AuthorizationRequest.create(
				"https://accounts.google.com/o/oauth2/v2/auth",
				"google-client-id",
				"http://localhost/login/oauth2/code/google",
				Set.of("openid"),
				"test-state",
				"google");

		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/login/oauth2/code/google");
		request.setParameter("error", "access_denied");
		request.setParameter("error_description", "User denied access");
		request.setParameter("state", "test-state");
		request.getSession().setAttribute(
				SimpleOAuth2AuthorizationRequestRedirectFilter.AUTHORIZATION_REQUEST_ATTR,
				savedRequest);

		MockHttpServletResponse response = new MockHttpServletResponse();

		// The error response should be passed to the authentication manager, which
		// will throw because the authorization response has no code
		when(this.authenticationManager.authenticate(any())).thenAnswer(invocation -> {
			OAuth2LoginAuthenticationToken token = invocation.getArgument(0);
			OAuth2AuthorizationResponse authResponse =
					token.getAuthorizationExchange().getAuthorizationResponse();
			assertFalse(authResponse.isSuccess());
			assertEquals("access_denied", authResponse.getError());
			throw new simple.security.oauth2.core.OAuth2AuthenticationException(
					new simple.security.oauth2.core.OAuth2Error("access_denied",
							"User denied access", null));
		});

		assertThrows(simple.security.oauth2.core.OAuth2AuthenticationException.class,
				() -> this.filter.attemptAuthentication(request, response));
	}

	@Test
	void shouldPassThroughForNonMatchingUrls() throws Exception {
		MockHttpServletRequest request = new MockHttpServletRequest("GET", "/api/data");
		MockHttpServletResponse response = new MockHttpServletResponse();
		FilterChain chain = mock(FilterChain.class);

		this.filter.doFilter(request, response, chain);

		verify(chain).doFilter(request, response);
	}

}
```

### [NEW] `simple-security-oauth2-client/src/test/java/simple/security/oauth2/client/authentication/OAuth2LoginAuthenticationTokenTest.java`

```java
package simple.security.oauth2.client.authentication;

import org.junit.jupiter.api.Test;

import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.web.OAuth2AuthorizationExchange;
import simple.security.oauth2.client.web.OAuth2AuthorizationRequest;
import simple.security.oauth2.client.web.OAuth2AuthorizationResponse;
import simple.security.oauth2.core.AuthorizationGrantType;
import simple.security.oauth2.core.OAuth2AccessToken;
import simple.security.oauth2.core.user.DefaultOAuth2User;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests for {@link OAuth2LoginAuthenticationToken} and {@link OAuth2AuthenticationToken}.
 */
class OAuth2LoginAuthenticationTokenTest {

	@Test
	void shouldCreateUnauthenticatedToken() {
		ClientRegistration registration = createRegistration();
		OAuth2AuthorizationExchange exchange = createExchange();

		OAuth2LoginAuthenticationToken token =
				new OAuth2LoginAuthenticationToken(registration, exchange);

		assertFalse(token.isAuthenticated());
		assertNull(token.getPrincipal());
		assertNull(token.getAccessToken());
		assertSame(registration, token.getClientRegistration());
		assertSame(exchange, token.getAuthorizationExchange());
	}

	@Test
	void shouldCreateAuthenticatedToken() {
		ClientRegistration registration = createRegistration();
		OAuth2AuthorizationExchange exchange = createExchange();
		DefaultOAuth2User user = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				Map.of("sub", "user123"),
				"sub");
		OAuth2AccessToken accessToken = new OAuth2AccessToken("token", Set.of("openid"));

		OAuth2LoginAuthenticationToken token = new OAuth2LoginAuthenticationToken(
				registration, exchange, user,
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				accessToken);

		assertTrue(token.isAuthenticated());
		assertSame(user, token.getPrincipal());
		assertSame(accessToken, token.getAccessToken());
		assertEquals(1, token.getAuthorities().size());
	}

	@Test
	void shouldCreateFinalAuthenticationToken() {
		DefaultOAuth2User user = new DefaultOAuth2User(
				List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
				Map.of("sub", "user123"),
				"sub");

		OAuth2AuthenticationToken token = new OAuth2AuthenticationToken(
				user,
				user.getAuthorities(),
				"google");

		assertTrue(token.isAuthenticated());
		assertEquals("user123", token.getName());
		assertEquals("google", token.getAuthorizedClientRegistrationId());
		assertSame(user, token.getPrincipal());
	}

	private ClientRegistration createRegistration() {
		return ClientRegistration.withRegistrationId("google")
				.clientId("client-id")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("http://localhost/callback")
				.authorizationUri("https://example.com/auth")
				.tokenUri("https://example.com/token")
				.build();
	}

	private OAuth2AuthorizationExchange createExchange() {
		OAuth2AuthorizationRequest request = OAuth2AuthorizationRequest.create(
				"https://example.com/auth", "client-id",
				"http://localhost/callback", Set.of("openid"),
				"state123", "google");
		OAuth2AuthorizationResponse response =
				OAuth2AuthorizationResponse.success("code123", "state123");
		return new OAuth2AuthorizationExchange(request, response);
	}

}
```

### [NEW] `simple-security-oauth2-client/src/test/java/simple/security/oauth2/client/userinfo/DefaultOAuth2UserServiceTest.java`

```java
package simple.security.oauth2.client.userinfo;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests for {@link DefaultOAuth2UserService} — specifically the JSON parsing utility.
 */
class DefaultOAuth2UserServiceTest {

	@Test
	void shouldParseStringValues() {
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson(
				"{\"name\": \"John Doe\", \"email\": \"john@example.com\"}");

		assertEquals("John Doe", result.get("name"));
		assertEquals("john@example.com", result.get("email"));
	}

	@Test
	void shouldParseNumericValues() {
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson(
				"{\"id\": 12345, \"score\": 99.5}");

		assertEquals(12345L, result.get("id"));
		assertEquals(99.5, result.get("score"));
	}

	@Test
	void shouldParseBooleanValues() {
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson(
				"{\"verified\": true, \"suspended\": false}");

		assertEquals(true, result.get("verified"));
		assertEquals(false, result.get("suspended"));
	}

	@Test
	void shouldParseNullValues() {
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson(
				"{\"bio\": null}");

		assertTrue(result.containsKey("bio"));
		assertNull(result.get("bio"));
	}

	@Test
	void shouldParseMixedValues() {
		String json = "{\"sub\": \"user123\", \"name\": \"John\", \"email_verified\": true, \"age\": 30}";
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson(json);

		assertEquals("user123", result.get("sub"));
		assertEquals("John", result.get("name"));
		assertEquals(true, result.get("email_verified"));
		assertEquals(30L, result.get("age"));
	}

	@Test
	void shouldReturnEmptyMapForEmptyJson() {
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson("{}");
		assertTrue(result.isEmpty());
	}

	@Test
	void shouldReturnEmptyMapForInvalidJson() {
		Map<String, Object> result = DefaultOAuth2UserService.parseSimpleJson("not json");
		assertTrue(result.isEmpty());
	}

}
```

### [NEW] `simple-security-config/src/test/java/simple/security/config/annotation/web/configurers/oauth2/client/SimpleOAuth2LoginConfigurerTest.java`

```java
package simple.security.config.annotation.web.configurers.oauth2.client;

import org.junit.jupiter.api.Test;

import jakarta.servlet.Filter;

import simple.security.config.Customizer;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.oauth2.client.registration.ClientRegistration;
import simple.security.oauth2.client.registration.SimpleInMemoryClientRegistrationRepository;
import simple.security.oauth2.client.userinfo.OAuth2UserService;
import simple.security.oauth2.core.AuthorizationGrantType;
import simple.security.oauth2.core.user.DefaultOAuth2User;
import simple.security.web.SecurityFilterChain;

import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Integration tests for {@link SimpleOAuth2LoginConfigurer}.
 *
 * <p>Tests the full DSL flow: configuring OAuth2 login, building the filter chain,
 * and verifying the correct filters are added in the right order.
 */
class SimpleOAuth2LoginConfigurerTest {

	@Test
	void shouldBuildFilterChainWithOAuth2Filters() throws Exception {
		ClientRegistration google = createGoogleRegistration();
		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(google);

		// Custom user service to avoid real HTTP calls
		OAuth2UserService mockUserService = userRequest ->
				new DefaultOAuth2User(
						List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
						Map.of("sub", "user123", "name", "John"),
						"sub");

		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2Login(oauth2 -> oauth2
				.clientRegistrationRepository(repository)
				.defaultSuccessUrl("/home")
				.userInfoEndpoint(userInfo -> userInfo
						.userService(mockUserService)));

		SecurityFilterChain chain = http.build();

		assertNotNull(chain);
		List<Filter> filters = chain.getFilters();
		assertFalse(filters.isEmpty());

		// Verify that both OAuth2 filters are present
		boolean hasRedirectFilter = filters.stream()
				.anyMatch(f -> f.getClass().getSimpleName().contains("OAuth2AuthorizationRequestRedirect"));
		boolean hasLoginFilter = filters.stream()
				.anyMatch(f -> f.getClass().getSimpleName().contains("OAuth2LoginAuthentication"));
		assertTrue(hasRedirectFilter, "Should contain OAuth2 redirect filter");
		assertTrue(hasLoginFilter, "Should contain OAuth2 login filter");
	}

	@Test
	void shouldRedirectToAuthorizationEndpoint_WhenAccessingOAuth2AuthorizationUrl()
			throws Exception {
		ClientRegistration google = createGoogleRegistration();
		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(google);

		OAuth2UserService mockUserService = userRequest ->
				new DefaultOAuth2User(
						List.of(new SimpleGrantedAuthority("OAUTH2_USER")),
						Map.of("sub", "user123"),
						"sub");

		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2Login(oauth2 -> oauth2
				.clientRegistrationRepository(repository)
				.userInfoEndpoint(userInfo -> userInfo
						.userService(mockUserService)));

		SecurityFilterChain chain = http.build();

		// Simulate a request to the authorization URL
		MockHttpServletRequest request = new MockHttpServletRequest("GET",
				"/oauth2/authorization/google");
		request.setServerName("localhost");
		request.setServerPort(8080);
		request.setScheme("http");
		MockHttpServletResponse response = new MockHttpServletResponse();

		// Run through the filter chain
		for (Filter filter : chain.getFilters()) {
			if (response.isCommitted()) break;
			filter.doFilter(request, response, (req, res) -> {});
		}

		assertNotNull(response.getRedirectedUrl());
		assertTrue(response.getRedirectedUrl().startsWith(
				"https://accounts.google.com/o/oauth2/v2/auth?"));
	}

	@Test
	void shouldFailWithoutClientRegistrationRepository() {
		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2Login(Customizer.withDefaults());

		assertThrows(Exception.class, http::build);
	}

	@Test
	void shouldSupportCustomLoginPage() throws Exception {
		ClientRegistration google = createGoogleRegistration();
		SimpleInMemoryClientRegistrationRepository repository =
				new SimpleInMemoryClientRegistrationRepository(google);

		SimpleHttpSecurity http = new SimpleHttpSecurity();
		http.oauth2Login(oauth2 -> oauth2
				.clientRegistrationRepository(repository)
				.loginPage("/custom-login")
				.failureUrl("/custom-login?error"));

		// Should build without error — the login page and failure URL are just configuration
		SecurityFilterChain chain = http.build();
		assertNotNull(chain);
	}

	private ClientRegistration createGoogleRegistration() {
		return ClientRegistration.withRegistrationId("google")
				.clientId("google-client-id")
				.clientSecret("google-secret")
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
				.scope("openid", "profile", "email")
				.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
				.tokenUri("https://oauth2.googleapis.com/token")
				.userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
				.userNameAttributeName("sub")
				.clientName("Google")
				.build();
	}

}
```
