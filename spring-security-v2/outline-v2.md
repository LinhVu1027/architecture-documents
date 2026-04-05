# Simple Spring Security — API-First Learning Outline

> Learn Spring Security by building a simplified version from the **outside in** — start with
> the APIs that clients actually call, then implement the internal machinery behind each one.

## Quick Start

```bash
# After implementing Feature 1, a client can already do:
PasswordEncoder encoder = new SimpleBCryptPasswordEncoder();
String encoded = encoder.encode("myPassword");
boolean matches = encoder.matches("myPassword", encoded); // true
```

## Architecture Overview

Spring Security protects your web application by **intercepting every HTTP request** through a
chain of servlet filters, **authenticating** the user (who are you?), and **authorizing** their
access (are you allowed?). As a client, you configure security rules through a fluent DSL
(`HttpSecurity`), provide a way to load users (`UserDetailsService`), and Spring Security handles
the rest — from password hashing to CSRF protection to OAuth2 token validation.

**Real-world analogy**: Using Spring Security is like hiring a security team for a building.
You tell them the rules ("employees can enter floors 1-10, executives can enter all floors,
visitors need a badge for floor 1 only"), provide them an employee directory (`UserDetailsService`),
and they station guards at every door (`SecurityFilterChain`) who check IDs (`AuthenticationManager`)
and enforce your rules (`AuthorizationManager`).

## Module Structure

This simplified version mirrors Spring Security's multi-module architecture:

```
simple-v2-spring-security/                    ← spring-security (root)
├── simple-security-crypto/                   ← spring-security-crypto
│   └── Password encoding (PasswordEncoder, BCrypt)
├── simple-security-core/                     ← spring-security-core
│   └── Authentication model, SecurityContext, UserDetails, AuthorizationManager
├── simple-security-web/                      ← spring-security-web
│   └── Servlet filter chain, CSRF, request matching, entry points
├── simple-security-config/                   ← spring-security-config
│   └── @EnableWebSecurity, HttpSecurity DSL, configurers
├── simple-security-oauth2-core/              ← spring-security-oauth2-core
│   └── Shared OAuth2/OIDC types
├── simple-security-oauth2-jose/              ← spring-security-oauth2-jose
│   └── JWT encoding/decoding
├── simple-security-oauth2-resource-server/   ← spring-security-oauth2-resource-server
│   └── Bearer token authentication for APIs
├── simple-security-oauth2-client/            ← spring-security-oauth2-client
│   └── OAuth2 Login / Client support
└── simple-security-test/                     ← spring-security-test
    └── @WithMockUser, MockMvc security support
```

### Module Dependency Graph

```
simple-security-crypto                          (no dependencies — leaf module)
         │
simple-security-core ──────────────────────► simple-security-crypto
         │
simple-security-web ───────────────────────► simple-security-core
         │                                        │
simple-security-config ────────────────────► simple-security-core
         │                                   simple-security-web
         │
simple-security-oauth2-core ───────────────► simple-security-core
         │
simple-security-oauth2-jose ───────────────► simple-security-oauth2-core
         │
simple-security-oauth2-resource-server ────► simple-security-core
         │                                   simple-security-oauth2-core
         │                                   simple-security-web
         │
simple-security-oauth2-client ─────────────► simple-security-core
         │                                   simple-security-oauth2-core
         │                                   simple-security-web
         │
simple-security-test ──────────────────────► simple-security-core
                                             simple-security-web
```

## API Surface Map

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT CODE                                     │
│  @EnableWebSecurity  @PreAuthorize  UserDetailsService  PasswordEncoder      │
│  HttpSecurity DSL    @WithMockUser  JwtDecoder          oauth2Login()        │
└────┬──────────┬──────────┬──────────┬──────────┬──────────┬─────────────────┘
     │          │          │          │          │          │
┌────▼───┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────┐ ┌──▼──────────┐
│Feature │ │Feature │ │Feature │ │Feature │ │Feature │ │Feature      │
│ 1-2    │ │ 3-4    │ │ 5-7    │ │ 8-12   │ │13-14   │ │  15         │
│crypto  │ │core    │ │config  │ │web+cfg │ │oauth2  │ │  test       │
│core    │ │auth    │ │DSL     │ │protect │ │modules │ │  module     │
└────┬───┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └──┬──────────┘
     │         │          │          │          │          │
┌────▼─────────▼──────────▼──────────▼──────────▼──────────▼──────────────────┐
│                          SHARED INTERNALS                                    │
│  SecurityContext · FilterChainProxy · AuthorizationManager · RequestMatcher  │
│  ExceptionTranslationFilter · CsrfFilter · SessionManagement               │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Call Chain Diagrams

### Password Encoding Call Chain
```
Client calls: passwordEncoder.encode("rawPassword")
  │
  ├─► [API Layer] PasswordEncoder.encode(CharSequence)
  │     Interface contract — delegates to concrete encoder
  │
  └─► [Crypto Layer] BCryptPasswordEncoder.encode()
        Generates salt, applies BCrypt hash, returns "{bcrypt}$2a$10$..."

Client calls: passwordEncoder.matches("raw", encodedPassword)
  │
  ├─► [API Layer] DelegatingPasswordEncoder.matches()
  │     Extracts {id} prefix, selects matching encoder
  │
  └─► [Crypto Layer] BCryptPasswordEncoder.matches()
        Extracts salt from encoded, re-hashes raw, compares
```

### SecurityContext Access Call Chain
```
Client calls: SecurityContextHolder.getContext().getAuthentication()
  │
  ├─► [API Layer] SecurityContextHolder.getContext()
  │     Static accessor — delegates to strategy
  │
  ├─► [Strategy Layer] ThreadLocalSecurityContextHolderStrategy.getContext()
  │     Returns SecurityContext from ThreadLocal<SecurityContext>
  │
  └─► [Model Layer] SecurityContextImpl.getAuthentication()
        Returns the current Authentication object (principal + authorities)
```

### Authentication Pipeline Call Chain
```
Client calls: authenticationManager.authenticate(usernamePasswordToken)
  │
  ├─► [API Layer] ProviderManager.authenticate(Authentication)
  │     Iterates through List<AuthenticationProvider>
  │
  ├─► [Provider Layer] DaoAuthenticationProvider.authenticate()
  │     Calls UserDetailsService, checks password via PasswordEncoder
  │
  ├─► [UserDetails Layer] UserDetailsService.loadUserByUsername(String)
  │     Returns UserDetails (username, encoded password, authorities)
  │
  └─► [Crypto Layer] PasswordEncoder.matches(raw, encoded)
        Verifies credentials — returns authenticated token or throws
```

### HTTP Request Through SecurityFilterChain
```
HTTP Request: POST /api/orders
  │
  ├─► [Servlet Bridge] DelegatingFilterProxy.doFilter()
  │     Looks up "springSecurityFilterChain" bean from ApplicationContext
  │
  ├─► [Dispatch] FilterChainProxy.doFilter()
  │     Iterates SecurityFilterChain list, selects first match
  │
  ├─► [Filter Chain] Ordered security filters execute sequentially:
  │     │
  │     ├─► SecurityContextHolderFilter — loads SecurityContext
  │     ├─► CsrfFilter — validates CSRF token
  │     ├─► UsernamePasswordAuthenticationFilter — processes login form
  │     ├─► BasicAuthenticationFilter — processes HTTP Basic
  │     ├─► AuthorizationFilter — checks access rules
  │     └─► ExceptionTranslationFilter — catches auth exceptions
  │
  └─► [Dispatch to Servlet] request continues to DispatcherServlet
```

### HttpSecurity DSL Configuration Call Chain
```
Client calls: http.authorizeHttpRequests(auth -> auth.requestMatchers("/api/**").authenticated())
  │
  ├─► [API Layer] HttpSecurity.authorizeHttpRequests(Customizer)
  │     Retrieves or creates AuthorizeHttpRequestsConfigurer
  │
  ├─► [Configurer Layer] AuthorizeHttpRequestsConfigurer
  │     Customizer callback populates RequestMatcher → rule mappings
  │
  ├─► [Registry Layer] AuthorizationManagerRequestMatcherRegistry
  │     Collects .requestMatchers("/api/**").authenticated() as
  │     PathPatternRequestMatcher → AuthenticatedAuthorizationManager
  │
  └─► [Build Phase] HttpSecurity.build() → DefaultSecurityFilterChain
        Calls configurer.configure(http) → creates AuthorizationFilter
        with RequestMatcherDelegatingAuthorizationManager
```

## Features

---

### Feature 1: Password Encoding {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
// Client writes this:
PasswordEncoder encoder = new SimpleBCryptPasswordEncoder();
String encoded = encoder.encode("myPassword");
boolean valid = encoder.matches("myPassword", encoded);

// Or use the delegating encoder:
PasswordEncoder delegating = SimplePasswordEncoderFactories.createDelegatingPasswordEncoder();
String hashed = delegating.encode("secret");  // "{bcrypt}$2a$10$..."
```

**What it does**: Provides a clean API for hashing passwords and verifying them, with support for multiple encoding algorithms via a prefix-based delegating strategy.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `PasswordEncoder` | `o.s.s.crypto.password.PasswordEncoder` | Interface clients program against |
| Crypto | `SimpleBCryptPasswordEncoder` | `o.s.s.crypto.bcrypt.BCryptPasswordEncoder` | BCrypt hashing implementation |
| Delegation | `SimpleDelegatingPasswordEncoder` | `o.s.s.crypto.password.DelegatingPasswordEncoder` | Routes by `{id}` prefix |
| Factory | `SimplePasswordEncoderFactories` | `o.s.s.crypto.factory.PasswordEncoderFactories` | Creates pre-configured delegating encoder |

**Real source files**:
- `crypto/src/main/java/org/springframework/security/crypto/password/PasswordEncoder.java`
- `crypto/src/main/java/org/springframework/security/crypto/bcrypt/BCryptPasswordEncoder.java`
- `crypto/src/main/java/org/springframework/security/crypto/bcrypt/BCrypt.java`
- `crypto/src/main/java/org/springframework/security/crypto/password/DelegatingPasswordEncoder.java`
- `crypto/src/main/java/org/springframework/security/crypto/factory/PasswordEncoderFactories.java`

**Module**: `simple-security-crypto`
**Depends on**: None (leaf module)
**Complexity**: Low · 5 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-crypto/.../crypto/password/PasswordEncoder.java` — core encoding interface
- [ ] `simple-security-crypto/.../crypto/bcrypt/SimpleBCryptPasswordEncoder.java` — BCrypt implementation
- [ ] `simple-security-crypto/.../crypto/password/SimpleDelegatingPasswordEncoder.java` — prefix-based delegation
- [ ] `simple-security-crypto/.../crypto/factory/SimplePasswordEncoderFactories.java` — factory with defaults
- [ ] Tests for all above
- [ ] `docs-v2/ch01_password_encoding.md` — tutorial chapter

---

### Feature 2: Security Context & Authentication Model {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
// Read current authentication:
SecurityContext context = SecurityContextHolder.getContext();
Authentication auth = context.getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();

// Create authentication tokens:
var token = UsernamePasswordAuthenticationToken.unauthenticated("user", "pass");
var authenticated = UsernamePasswordAuthenticationToken.authenticated(
    userDetails, null, userDetails.getAuthorities());

// Build user details:
UserDetails user = User.withUsername("admin")
    .password("{bcrypt}$2a$10$...")
    .roles("ADMIN", "USER")
    .build();
```

**What it does**: Provides the identity model — who is the current user (`Authentication`), what can they do (`GrantedAuthority`), and where is this stored during a request (`SecurityContext` in a `ThreadLocal`).

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `SecurityContextHolder` | `o.s.s.core.context.SecurityContextHolder` | Static accessor for current SecurityContext |
| Strategy | `ThreadLocalSecurityContextHolderStrategy` | `o.s.s.core.context.ThreadLocalSecurityContextHolderStrategy` | ThreadLocal storage strategy |
| Model | `SecurityContext` / `SecurityContextImpl` | `o.s.s.core.context.SecurityContext/Impl` | Holds Authentication |
| Model | `Authentication` | `o.s.s.core.Authentication` | Principal + credentials + authorities |
| Model | `UsernamePasswordAuthenticationToken` | `o.s.s.authentication.UsernamePasswordAuthenticationToken` | Username/password auth token |
| Model | `GrantedAuthority` / `SimpleGrantedAuthority` | `o.s.s.core.GrantedAuthority` | Authority representation |
| Model | `UserDetails` / `User` | `o.s.s.core.userdetails.UserDetails/User` | User info with builder |
| Exception | `AuthenticationException` / `AccessDeniedException` | `o.s.s.core.AuthenticationException` | Security exception hierarchy |

**Real source files**:
- `core/src/main/java/org/springframework/security/core/context/SecurityContextHolder.java`
- `core/src/main/java/org/springframework/security/core/context/SecurityContext.java`
- `core/src/main/java/org/springframework/security/core/Authentication.java`
- `core/src/main/java/org/springframework/security/authentication/UsernamePasswordAuthenticationToken.java`
- `core/src/main/java/org/springframework/security/core/GrantedAuthority.java`
- `core/src/main/java/org/springframework/security/core/userdetails/UserDetails.java`
- `core/src/main/java/org/springframework/security/core/userdetails/User.java`

**Module**: `simple-security-core`
**Depends on**: Feature 1 (PasswordEncoder for User.builder password encoding)
**Complexity**: Medium · 10 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-core/.../core/context/SecurityContextHolder.java`
- [ ] `simple-security-core/.../core/context/SecurityContext.java`
- [ ] `simple-security-core/.../core/Authentication.java`
- [ ] `simple-security-core/.../authentication/UsernamePasswordAuthenticationToken.java`
- [ ] `simple-security-core/.../core/GrantedAuthority.java`
- [ ] `simple-security-core/.../core/authority/SimpleGrantedAuthority.java`
- [ ] `simple-security-core/.../core/userdetails/UserDetails.java`
- [ ] `simple-security-core/.../core/userdetails/User.java`
- [ ] `simple-security-core/.../core/AuthenticationException.java`
- [ ] `simple-security-core/.../access/AccessDeniedException.java`
- [ ] Tests for all above
- [ ] `docs-v2/ch02_security_context.md` — tutorial chapter

---

### Feature 3: Authentication Pipeline {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
// Configure a UserDetailsService:
@Bean
UserDetailsService userDetailsService() {
    UserDetails user = User.withUsername("user")
        .password("{bcrypt}$2a$10$...")
        .roles("USER")
        .build();
    return new SimpleInMemoryUserDetailsManager(user);
}

// Programmatic authentication:
AuthenticationManager manager = new SimpleProviderManager(
    new SimpleDaoAuthenticationProvider(userDetailsService, passwordEncoder));
Authentication result = manager.authenticate(
    UsernamePasswordAuthenticationToken.unauthenticated("user", "password"));
```

**What it does**: Provides the authentication pipeline — a manager delegates to providers, each provider knows how to verify a specific credential type. The most common provider loads users from a `UserDetailsService` and checks passwords.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `AuthenticationManager` | `o.s.s.authentication.AuthenticationManager` | Central authentication interface |
| Dispatch | `SimpleProviderManager` | `o.s.s.authentication.ProviderManager` | Iterates providers |
| Provider | `AuthenticationProvider` | `o.s.s.authentication.AuthenticationProvider` | Strategy per auth type |
| Provider | `SimpleDaoAuthenticationProvider` | `o.s.s.authentication.dao.DaoAuthenticationProvider` | UserDetails + PasswordEncoder |
| UserStore | `UserDetailsService` | `o.s.s.core.userdetails.UserDetailsService` | Load user by username |
| UserStore | `SimpleInMemoryUserDetailsManager` | `o.s.s.provisioning.InMemoryUserDetailsManager` | HashMap-backed user store |

**Real source files**:
- `core/src/main/java/org/springframework/security/authentication/AuthenticationManager.java`
- `core/src/main/java/org/springframework/security/authentication/ProviderManager.java`
- `core/src/main/java/org/springframework/security/authentication/AuthenticationProvider.java`
- `core/src/main/java/org/springframework/security/authentication/dao/DaoAuthenticationProvider.java`
- `core/src/main/java/org/springframework/security/core/userdetails/UserDetailsService.java`
- `core/src/main/java/org/springframework/security/provisioning/InMemoryUserDetailsManager.java`

**Module**: `simple-security-core`
**Depends on**: Feature 1 (PasswordEncoder), Feature 2 (Authentication model)
**Complexity**: Medium · 6 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-core/.../authentication/AuthenticationManager.java`
- [ ] `simple-security-core/.../authentication/SimpleProviderManager.java`
- [ ] `simple-security-core/.../authentication/AuthenticationProvider.java`
- [ ] `simple-security-core/.../authentication/dao/SimpleDaoAuthenticationProvider.java`
- [ ] `simple-security-core/.../core/userdetails/UserDetailsService.java`
- [ ] `simple-security-core/.../provisioning/SimpleInMemoryUserDetailsManager.java`
- [ ] Tests for full authentication flow
- [ ] `docs-v2/ch03_authentication_pipeline.md` — tutorial chapter

---

### Feature 4: Security Filter Chain {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
// The SecurityFilterChain interface — what gets exposed as a @Bean:
SecurityFilterChain chain = new SimpleDefaultSecurityFilterChain(
    new SimplePathPatternRequestMatcher("/api/**"),
    List.of(authFilter, authorizationFilter, exceptionFilter));

// FilterChainProxy dispatches requests to the matching chain:
FilterChainProxy proxy = new SimpleFilterChainProxy(List.of(chain));

// Request matching:
RequestMatcher matcher = new SimplePathPatternRequestMatcher("/admin/**");
boolean matches = matcher.matches(request); // true for /admin/users
```

**What it does**: Bridges the servlet container to Spring Security by intercepting every HTTP request through a `FilterChainProxy`, which selects the correct `SecurityFilterChain` and executes its ordered list of security filters.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `SecurityFilterChain` | `o.s.s.web.SecurityFilterChain` | Interface: matches + getFilters |
| Implementation | `SimpleDefaultSecurityFilterChain` | `o.s.s.web.DefaultSecurityFilterChain` | Concrete chain with matcher + filters |
| Dispatch | `SimpleFilterChainProxy` | `o.s.s.web.FilterChainProxy` | Selects and runs matching chain |
| Bridge | `SimpleDelegatingFilterProxy` | `o.s.web.filter.DelegatingFilterProxy` | Servlet → Spring bean bridge |
| Matching | `RequestMatcher` | `o.s.s.web.util.matcher.RequestMatcher` | Functional interface for request matching |
| Matching | `SimplePathPatternRequestMatcher` | `o.s.s.web.servlet.util.matcher.PathPatternRequestMatcher` | Pattern-based URL matcher |
| Matching | `SimpleAnyRequestMatcher` | `o.s.s.web.util.matcher.AnyRequestMatcher` | Matches all requests |

**Real source files**:
- `web/src/main/java/org/springframework/security/web/SecurityFilterChain.java`
- `web/src/main/java/org/springframework/security/web/DefaultSecurityFilterChain.java`
- `web/src/main/java/org/springframework/security/web/FilterChainProxy.java`
- `web/src/main/java/org/springframework/security/web/util/matcher/RequestMatcher.java`
- `web/src/main/java/org/springframework/security/web/servlet/util/matcher/PathPatternRequestMatcher.java`

**Module**: `simple-security-web`
**Depends on**: Feature 2 (SecurityContext for filter context propagation)
**Complexity**: Medium · 7 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-web/.../web/SecurityFilterChain.java`
- [ ] `simple-security-web/.../web/SimpleDefaultSecurityFilterChain.java`
- [ ] `simple-security-web/.../web/SimpleFilterChainProxy.java`
- [ ] `simple-security-web/.../web/SimpleDelegatingFilterProxy.java`
- [ ] `simple-security-web/.../web/util/matcher/RequestMatcher.java`
- [ ] `simple-security-web/.../web/util/matcher/SimplePathPatternRequestMatcher.java`
- [ ] `simple-security-web/.../web/util/matcher/SimpleAnyRequestMatcher.java`
- [ ] Tests with mock servlet requests
- [ ] `docs-v2/ch04_security_filter_chain.md` — tutorial chapter

---

### Feature 5: HTTP Security DSL & Auto-Configuration {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
@EnableSimpleWebSecurity
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(SimpleHttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

**What it does**: Provides the fluent DSL builder (`HttpSecurity`) that clients use to declare security rules, and the `@EnableWebSecurity` annotation that bootstraps the entire security infrastructure.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `@EnableSimpleWebSecurity` | `@EnableWebSecurity` | Activation annotation |
| API | `SimpleHttpSecurity` | `HttpSecurity` | Fluent builder for filter chain |
| API | `Customizer<T>` | `Customizer<T>` | Lambda callback for sub-configurers |
| Config | `SimpleWebSecurityConfiguration` | `WebSecurityConfiguration` | Creates FilterChainProxy bean |
| Config | `SimpleHttpSecurityConfiguration` | `HttpSecurityConfiguration` | Creates HttpSecurity prototype bean |
| Configurer | `SimpleAbstractHttpConfigurer` | `AbstractHttpConfigurer` | Base for DSL extensions |

**Real source files**:
- `config/src/main/java/org/springframework/security/config/annotation/web/configuration/EnableWebSecurity.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/builders/HttpSecurity.java`
- `config/src/main/java/org/springframework/security/config/Customizer.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configuration/WebSecurityConfiguration.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configuration/HttpSecurityConfiguration.java`

**Module**: `simple-security-config`
**Depends on**: Feature 2 (core model), Feature 4 (SecurityFilterChain, FilterChainProxy)
**Complexity**: High · 8 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-config/.../config/annotation/web/configuration/EnableSimpleWebSecurity.java`
- [ ] `simple-security-config/.../config/annotation/web/builders/SimpleHttpSecurity.java`
- [ ] `simple-security-config/.../config/Customizer.java`
- [ ] `simple-security-config/.../config/annotation/web/configuration/SimpleWebSecurityConfiguration.java`
- [ ] `simple-security-config/.../config/annotation/web/configuration/SimpleHttpSecurityConfiguration.java`
- [ ] `simple-security-config/.../config/annotation/web/configurers/SimpleAbstractHttpConfigurer.java`
- [ ] Integration test: full DSL → working filter chain
- [ ] `docs-v2/ch05_http_security_dsl.md` — tutorial chapter

---

### Feature 6: Form Login & HTTP Basic Authentication {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
// Form login with defaults (generates login page):
http.formLogin(Customizer.withDefaults());

// Custom form login:
http.formLogin(form -> form
    .loginPage("/login")
    .usernameParameter("email")
    .defaultSuccessUrl("/dashboard")
    .failureUrl("/login?error"));

// HTTP Basic:
http.httpBasic(Customizer.withDefaults());

// Custom Basic with realm:
http.httpBasic(basic -> basic
    .authenticationEntryPoint(customEntryPoint));
```

**What it does**: Provides two standard web authentication mechanisms — form-based login (POST username/password) and HTTP Basic (Authorization header) — as configurable filters that extract credentials and feed them to the `AuthenticationManager`.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Filter | `SimpleUsernamePasswordAuthenticationFilter` | `UsernamePasswordAuthenticationFilter` | Processes login form POST |
| Filter | `SimpleBasicAuthenticationFilter` | `BasicAuthenticationFilter` | Decodes Basic auth header |
| Base | `SimpleAbstractAuthenticationProcessingFilter` | `AbstractAuthenticationProcessingFilter` | Template for auth filters |
| Configurer | `SimpleFormLoginConfigurer` | `FormLoginConfigurer` | DSL for form login |
| Configurer | `SimpleHttpBasicConfigurer` | `HttpBasicConfigurer` | DSL for HTTP Basic |
| Handler | `AuthenticationSuccessHandler` | `AuthenticationSuccessHandler` | Post-login redirect |
| Handler | `AuthenticationFailureHandler` | `AuthenticationFailureHandler` | Login failure handling |
| EntryPoint | `AuthenticationEntryPoint` | `AuthenticationEntryPoint` | Initiates authentication |

**Real source files**:
- `web/src/main/java/org/springframework/security/web/authentication/UsernamePasswordAuthenticationFilter.java`
- `web/src/main/java/org/springframework/security/web/authentication/www/BasicAuthenticationFilter.java`
- `web/src/main/java/org/springframework/security/web/authentication/AbstractAuthenticationProcessingFilter.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/FormLoginConfigurer.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/HttpBasicConfigurer.java`
- `web/src/main/java/org/springframework/security/web/AuthenticationEntryPoint.java`

**Module**: `simple-security-web` (filters) + `simple-security-config` (configurers)
**Depends on**: Feature 3 (AuthenticationManager), Feature 4 (filter chain), Feature 5 (DSL)
**Complexity**: High · 10 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-web/.../web/authentication/SimpleUsernamePasswordAuthenticationFilter.java`
- [ ] `simple-security-web/.../web/authentication/SimpleBasicAuthenticationFilter.java`
- [ ] `simple-security-web/.../web/authentication/SimpleAbstractAuthenticationProcessingFilter.java`
- [ ] `simple-security-web/.../web/AuthenticationEntryPoint.java`
- [ ] `simple-security-web/.../web/authentication/AuthenticationSuccessHandler.java`
- [ ] `simple-security-web/.../web/authentication/AuthenticationFailureHandler.java`
- [ ] `simple-security-config/.../configurers/SimpleFormLoginConfigurer.java`
- [ ] `simple-security-config/.../configurers/SimpleHttpBasicConfigurer.java`
- [ ] Integration test: form login + basic auth end-to-end
- [ ] `docs-v2/ch06_form_login_http_basic.md` — tutorial chapter

---

### Feature 7: URL Authorization {{Tier: 1}}
✅ Complete

**API Contract** (what clients call):
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/**").hasAnyAuthority("SCOPE_read", "ROLE_USER")
    .anyRequest().authenticated());

// Custom AuthorizationManager:
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/special/**").access(customAuthorizationManager));
```

**What it does**: Protects URL patterns with authorization rules — after authentication, the `AuthorizationFilter` checks if the current user's authorities satisfy the rules declared for the matched URL.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| API | `AuthorizationManager<T>` | `o.s.s.authorization.AuthorizationManager` | Core authorization interface |
| Decision | `AuthorizationDecision` | `o.s.s.authorization.AuthorizationDecision` | Grant/deny result |
| Filter | `SimpleAuthorizationFilter` | `o.s.s.web.access.intercept.AuthorizationFilter` | Invokes AuthorizationManager per request |
| Composite | `SimpleRequestMatcherDelegatingAuthorizationManager` | `RequestMatcherDelegatingAuthorizationManager` | Maps matchers → managers |
| Managers | `SimpleAuthorityAuthorizationManager` | `AuthorityAuthorizationManager` | hasRole/hasAuthority checks |
| Managers | `SimpleAuthenticatedAuthorizationManager` | `AuthenticatedAuthorizationManager` | authenticated() check |
| Configurer | `SimpleAuthorizeHttpRequestsConfigurer` | `AuthorizeHttpRequestsConfigurer` | DSL for URL rules |

**Real source files**:
- `core/src/main/java/org/springframework/security/authorization/AuthorizationManager.java`
- `core/src/main/java/org/springframework/security/authorization/AuthorizationDecision.java`
- `core/src/main/java/org/springframework/security/authorization/AuthorityAuthorizationManager.java`
- `web/src/main/java/org/springframework/security/web/access/intercept/AuthorizationFilter.java`
- `web/src/main/java/org/springframework/security/web/access/intercept/RequestMatcherDelegatingAuthorizationManager.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/AuthorizeHttpRequestsConfigurer.java`

**Module**: `simple-security-core` (AuthorizationManager) + `simple-security-web` (filter) + `simple-security-config` (configurer)
**Depends on**: Feature 2 (Authentication), Feature 4 (filter chain, RequestMatcher), Feature 5 (DSL)
**Complexity**: High · 8 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-core/.../authorization/AuthorizationManager.java`
- [ ] `simple-security-core/.../authorization/AuthorizationDecision.java`
- [ ] `simple-security-core/.../authorization/SimpleAuthorityAuthorizationManager.java`
- [ ] `simple-security-core/.../authorization/SimpleAuthenticatedAuthorizationManager.java`
- [ ] `simple-security-web/.../web/access/intercept/SimpleAuthorizationFilter.java`
- [ ] `simple-security-web/.../web/access/intercept/SimpleRequestMatcherDelegatingAuthorizationManager.java`
- [ ] `simple-security-config/.../configurers/SimpleAuthorizeHttpRequestsConfigurer.java`
- [ ] Tests: permit/deny rules with mock requests
- [ ] `docs-v2/ch07_url_authorization.md` — tutorial chapter

---

### Feature 8: Exception Handling {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((request, response, authException) ->
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Please log in"))
    .accessDeniedHandler((request, response, accessDeniedException) ->
        response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied")));
```

**What it does**: Translates `AuthenticationException` and `AccessDeniedException` thrown by downstream filters into appropriate HTTP responses (401/redirect to login or 403).

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Filter | `SimpleExceptionTranslationFilter` | `ExceptionTranslationFilter` | Catches and translates exceptions |
| Handler | `AuthenticationEntryPoint` | `AuthenticationEntryPoint` | Handles 401 (unauthenticated) |
| Handler | `AccessDeniedHandler` | `AccessDeniedHandler` | Handles 403 (unauthorized) |
| Configurer | `SimpleExceptionHandlingConfigurer` | `ExceptionHandlingConfigurer` | DSL for exception config |

**Real source files**:
- `web/src/main/java/org/springframework/security/web/access/ExceptionTranslationFilter.java`
- `web/src/main/java/org/springframework/security/web/AuthenticationEntryPoint.java`
- `web/src/main/java/org/springframework/security/web/access/AccessDeniedHandler.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/ExceptionHandlingConfigurer.java`

**Module**: `simple-security-web` (filter, handlers) + `simple-security-config` (configurer)
**Depends on**: Feature 4 (filter chain), Feature 5 (DSL)
**Complexity**: Medium · 5 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-web/.../web/access/SimpleExceptionTranslationFilter.java`
- [ ] `simple-security-web/.../web/access/AccessDeniedHandler.java` (interface)
- [ ] `simple-security-config/.../configurers/SimpleExceptionHandlingConfigurer.java`
- [ ] Tests: exception → HTTP response mapping
- [ ] `docs-v2/ch08_exception_handling.md` — tutorial chapter

---

### Feature 9: CSRF Protection {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
// Enable CSRF with defaults (HttpSession token store):
http.csrf(Customizer.withDefaults());

// Cookie-based CSRF for SPAs:
http.csrf(csrf -> csrf
    .csrfTokenRepository(new SimpleCookieCsrfTokenRepository()));

// Disable CSRF for stateless APIs:
http.csrf(csrf -> csrf.disable());

// Ignore specific paths:
http.csrf(csrf -> csrf
    .ignoringRequestMatchers("/api/webhooks/**"));
```

**What it does**: Protects against Cross-Site Request Forgery by requiring a secret token on state-changing requests (POST/PUT/DELETE) that matches a token stored in the session or cookie.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Filter | `SimpleCsrfFilter` | `CsrfFilter` | Validates CSRF token on requests |
| Token | `CsrfToken` | `CsrfToken` | Token value + header/parameter names |
| Repository | `CsrfTokenRepository` | `CsrfTokenRepository` | Generate/store/load tokens |
| Repository | `SimpleHttpSessionCsrfTokenRepository` | `HttpSessionCsrfTokenRepository` | Session-based storage |
| Repository | `SimpleCookieCsrfTokenRepository` | `CookieCsrfTokenRepository` | Cookie-based storage (SPA-friendly) |
| Configurer | `SimpleCsrfConfigurer` | `CsrfConfigurer` | DSL for CSRF configuration |

**Real source files**:
- `web/src/main/java/org/springframework/security/web/csrf/CsrfFilter.java`
- `web/src/main/java/org/springframework/security/web/csrf/CsrfToken.java`
- `web/src/main/java/org/springframework/security/web/csrf/CsrfTokenRepository.java`
- `web/src/main/java/org/springframework/security/web/csrf/HttpSessionCsrfTokenRepository.java`
- `web/src/main/java/org/springframework/security/web/csrf/CookieCsrfTokenRepository.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/CsrfConfigurer.java`

**Module**: `simple-security-web` (filter, repository) + `simple-security-config` (configurer)
**Depends on**: Feature 4 (filter chain, RequestMatcher), Feature 5 (DSL)
**Complexity**: Medium · 7 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-web/.../web/csrf/SimpleCsrfFilter.java`
- [ ] `simple-security-web/.../web/csrf/CsrfToken.java`
- [ ] `simple-security-web/.../web/csrf/CsrfTokenRepository.java`
- [ ] `simple-security-web/.../web/csrf/SimpleHttpSessionCsrfTokenRepository.java`
- [ ] `simple-security-web/.../web/csrf/SimpleCookieCsrfTokenRepository.java`
- [ ] `simple-security-config/.../configurers/SimpleCsrfConfigurer.java`
- [ ] Tests: CSRF validation, cookie and session modes
- [ ] `docs-v2/ch09_csrf_protection.md` — tutorial chapter

---

### Feature 10: Session Management {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
// Stateless (for APIs — no HttpSession created):
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS));

// Session fixation protection (default):
http.sessionManagement(session -> session
    .sessionFixation(fix -> fix.changeSessionId()));

// Concurrent session control:
http.sessionManagement(session -> session
    .maximumSessions(1));

// SecurityContext persistence via session:
http.securityContext(ctx -> ctx
    .securityContextRepository(new SimpleHttpSessionSecurityContextRepository()));
```

**What it does**: Controls how HTTP sessions interact with security — whether sessions are created, how the `SecurityContext` is persisted between requests, and protection against session fixation attacks.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Filter | `SimpleSecurityContextHolderFilter` | `SecurityContextHolderFilter` | Loads/saves SecurityContext per request |
| Repository | `SecurityContextRepository` | `SecurityContextRepository` | Persist SecurityContext strategy |
| Repository | `SimpleHttpSessionSecurityContextRepository` | `HttpSessionSecurityContextRepository` | Session-based persistence |
| Policy | `SessionCreationPolicy` | `SessionCreationPolicy` | ALWAYS / IF_REQUIRED / NEVER / STATELESS |
| Configurer | `SimpleSessionManagementConfigurer` | `SessionManagementConfigurer` | DSL for session management |

**Real source files**:
- `web/src/main/java/org/springframework/security/web/context/SecurityContextHolderFilter.java`
- `web/src/main/java/org/springframework/security/web/context/SecurityContextRepository.java`
- `web/src/main/java/org/springframework/security/web/context/HttpSessionSecurityContextRepository.java`
- `config/src/main/java/org/springframework/security/config/http/SessionCreationPolicy.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/SessionManagementConfigurer.java`

**Module**: `simple-security-web` (filters, repository) + `simple-security-config` (configurer, policy)
**Depends on**: Feature 2 (SecurityContext), Feature 4 (filter chain), Feature 5 (DSL)
**Complexity**: Medium · 6 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-web/.../web/context/SimpleSecurityContextHolderFilter.java`
- [ ] `simple-security-web/.../web/context/SecurityContextRepository.java`
- [ ] `simple-security-web/.../web/context/SimpleHttpSessionSecurityContextRepository.java`
- [ ] `simple-security-config/.../config/http/SessionCreationPolicy.java`
- [ ] `simple-security-config/.../configurers/SimpleSessionManagementConfigurer.java`
- [ ] Tests: stateless vs session-based context persistence
- [ ] `docs-v2/ch10_session_management.md` — tutorial chapter

---

### Feature 11: Method Security {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
@EnableSimpleMethodSecurity
@Configuration
public class MethodSecurityConfig { }

@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) { /* ... */ }

    @PreAuthorize("hasAuthority('ORDER_READ')")
    public Order getOrder(Long id) { /* ... */ }

    @PostAuthorize("returnObject.owner == authentication.name")
    public Order findOrder(Long id) { /* ... */ }
}
```

**What it does**: Secures individual method invocations using annotations with SpEL expressions. An AOP interceptor evaluates the expression against the current `Authentication` before/after method execution.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Annotation | `@PreAuthorize` | `@PreAuthorize` | Pre-invocation SpEL check |
| Annotation | `@PostAuthorize` | `@PostAuthorize` | Post-invocation SpEL check |
| Config | `@EnableSimpleMethodSecurity` | `@EnableMethodSecurity` | Activates method security |
| Interceptor | `SimplePreAuthorizeAuthorizationManager` | `PreAuthorizeAuthorizationManager` | Evaluates @PreAuthorize |
| Interceptor | `SimpleAuthorizationManagerBeforeMethodInterceptor` | `AuthorizationManagerBeforeMethodInterceptor` | AOP advice wrapping methods |
| Expression | `SimpleMethodSecurityExpressionHandler` | `DefaultMethodSecurityExpressionHandler` | SpEL evaluation with security roots |

**Real source files**:
- `core/src/main/java/org/springframework/security/access/prepost/PreAuthorize.java`
- `core/src/main/java/org/springframework/security/access/prepost/PostAuthorize.java`
- `config/src/main/java/org/springframework/security/config/annotation/method/configuration/EnableMethodSecurity.java`
- `core/src/main/java/org/springframework/security/authorization/method/PreAuthorizeAuthorizationManager.java`
- `core/src/main/java/org/springframework/security/authorization/method/AuthorizationManagerBeforeMethodInterceptor.java`

**Module**: `simple-security-core` (annotations, managers) + `simple-security-config` (enable annotation, AOP config)
**Depends on**: Feature 2 (Authentication, SecurityContext), Feature 7 (AuthorizationManager)
**Complexity**: High · 8 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-core/.../access/prepost/PreAuthorize.java`
- [ ] `simple-security-core/.../access/prepost/PostAuthorize.java`
- [ ] `simple-security-core/.../authorization/method/SimplePreAuthorizeAuthorizationManager.java`
- [ ] `simple-security-core/.../authorization/method/SimpleAuthorizationManagerBeforeMethodInterceptor.java`
- [ ] `simple-security-config/.../method/configuration/EnableSimpleMethodSecurity.java`
- [ ] Tests: method-level access control with SpEL
- [ ] `docs-v2/ch11_method_security.md` — tutorial chapter

---

### Feature 12: Logout {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
http.logout(logout -> logout
    .logoutUrl("/logout")
    .logoutSuccessUrl("/login?logout")
    .invalidateHttpSession(true)
    .deleteCookies("JSESSIONID")
    .addLogoutHandler(customHandler)
    .logoutSuccessHandler(customSuccessHandler));
```

**What it does**: Provides a logout endpoint that invalidates the session, clears the `SecurityContext`, removes cookies, and redirects to a configurable URL.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Filter | `SimpleLogoutFilter` | `LogoutFilter` | Intercepts logout URL, delegates to handlers |
| Handler | `LogoutHandler` | `LogoutHandler` | Functional: perform logout action |
| Handler | `LogoutSuccessHandler` | `LogoutSuccessHandler` | Post-logout redirect/response |
| Handler | `SimpleSecurityContextLogoutHandler` | `SecurityContextLogoutHandler` | Clears context + invalidates session |
| Configurer | `SimpleLogoutConfigurer` | `LogoutConfigurer` | DSL for logout configuration |

**Real source files**:
- `web/src/main/java/org/springframework/security/web/authentication/logout/LogoutFilter.java`
- `web/src/main/java/org/springframework/security/web/authentication/logout/LogoutHandler.java`
- `web/src/main/java/org/springframework/security/web/authentication/logout/LogoutSuccessHandler.java`
- `web/src/main/java/org/springframework/security/web/authentication/logout/SecurityContextLogoutHandler.java`
- `config/src/main/java/org/springframework/security/config/annotation/web/configurers/LogoutConfigurer.java`

**Module**: `simple-security-web` (filter, handlers) + `simple-security-config` (configurer)
**Depends on**: Feature 4 (filter chain), Feature 5 (DSL), Feature 10 (session)
**Complexity**: Low · 6 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-web/.../web/authentication/logout/SimpleLogoutFilter.java`
- [ ] `simple-security-web/.../web/authentication/logout/LogoutHandler.java`
- [ ] `simple-security-web/.../web/authentication/logout/LogoutSuccessHandler.java`
- [ ] `simple-security-web/.../web/authentication/logout/SimpleSecurityContextLogoutHandler.java`
- [ ] `simple-security-config/.../configurers/SimpleLogoutConfigurer.java`
- [ ] Tests: logout flow
- [ ] `docs-v2/ch12_logout.md` — tutorial chapter

---

### Feature 13: OAuth2 Resource Server (JWT) {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
// Configure JWT resource server:
http.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt
        .decoder(SimpleNimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build())
        .jwtAuthenticationConverter(customConverter)));

// Or with a custom JwtDecoder bean:
@Bean
JwtDecoder jwtDecoder() {
    return SimpleNimbusJwtDecoder.withPublicKey(rsaPublicKey).build();
}
```

**What it does**: Protects API endpoints by extracting a Bearer token from the `Authorization` header, decoding/validating it as a JWT, and converting it into an `Authentication` with granted authorities.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Filter | `SimpleBearerTokenAuthenticationFilter` | `BearerTokenAuthenticationFilter` | Extracts Bearer token from header |
| Resolver | `SimpleBearerTokenResolver` | `DefaultBearerTokenResolver` | Resolves token from request |
| Decoder | `JwtDecoder` | `JwtDecoder` | Functional: decode JWT string → Jwt |
| Decoder | `SimpleNimbusJwtDecoder` | `NimbusJwtDecoder` | Nimbus-based JWT decoding |
| Model | `Jwt` | `Jwt` | Decoded JWT (headers + claims) |
| Converter | `SimpleJwtAuthenticationConverter` | `JwtAuthenticationConverter` | Jwt → Authentication |
| Configurer | `SimpleOAuth2ResourceServerConfigurer` | `OAuth2ResourceServerConfigurer` | DSL for resource server |
| Model | `OAuth2AuthenticationToken` | `JwtAuthenticationToken` | Authentication for JWT principal |

**Real source files**:
- `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/authentication/BearerTokenAuthenticationFilter.java`
- `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/JwtDecoder.java`
- `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/NimbusJwtDecoder.java`
- `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/Jwt.java`
- `oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/authentication/JwtAuthenticationConverter.java`

**Module**: `simple-security-oauth2-core` + `simple-security-oauth2-jose` + `simple-security-oauth2-resource-server` + `simple-security-config`
**Depends on**: Feature 2 (Authentication), Feature 4 (filter chain), Feature 5 (DSL), Feature 7 (AuthorizationManager)
**Complexity**: High · 10 new classes across 4 modules
**Java version**: 17

**Concrete deliverables**:
- [x] `simple-security-oauth2-core/.../oauth2/core/OAuth2AuthenticationException.java`
- [x] `simple-security-oauth2-core/.../oauth2/core/OAuth2Error.java`
- [x] `simple-security-oauth2-jose/.../oauth2/jwt/Jwt.java`
- [x] `simple-security-oauth2-jose/.../oauth2/jwt/JwtDecoder.java`
- [x] `simple-security-oauth2-jose/.../oauth2/jwt/SimpleNimbusJwtDecoder.java`
- [x] `simple-security-oauth2-resource-server/.../resource/authentication/SimpleJwtAuthenticationConverter.java`
- [x] `simple-security-oauth2-resource-server/.../resource/web/SimpleBearerTokenAuthenticationFilter.java`
- [x] `simple-security-config/.../configurers/oauth2/SimpleOAuth2ResourceServerConfigurer.java`
- [x] Tests: JWT decoding, bearer token authentication flow
- [x] `docs-v2/ch13_oauth2_resource_server.md` — tutorial chapter

---

### Feature 14: OAuth2 Client & Login {{Tier: 3}}
✅ Complete

**API Contract** (what clients call):
```java
// Configure OAuth2 login:
http.oauth2Login(oauth2 -> oauth2
    .loginPage("/login")
    .userInfoEndpoint(userInfo -> userInfo
        .userService(customOAuth2UserService)));

// Register OAuth2 clients:
@Bean
ClientRegistrationRepository clientRegistrationRepository() {
    ClientRegistration google = ClientRegistration.withRegistrationId("google")
        .clientId("client-id")
        .clientSecret("client-secret")
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
        .scope("openid", "profile", "email")
        .authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
        .tokenUri("https://oauth2.googleapis.com/token")
        .userInfoUri("https://openidconnect.googleapis.com/v1/userinfo")
        .build();
    return new SimpleInMemoryClientRegistrationRepository(google);
}
```

**What it does**: Implements the OAuth 2.0 Authorization Code flow for "Login with Google/GitHub/etc." — redirects user to the identity provider, handles the callback with the authorization code, exchanges it for tokens, and loads user info.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Model | `ClientRegistration` | `ClientRegistration` | Registered OAuth2 client details |
| Model | `AuthorizationGrantType` | `AuthorizationGrantType` | Grant type enum (AUTH_CODE, CLIENT_CREDS) |
| Repository | `ClientRegistrationRepository` | `ClientRegistrationRepository` | Stores client registrations |
| Repository | `SimpleInMemoryClientRegistrationRepository` | `InMemoryClientRegistrationRepository` | In-memory storage |
| Filter | `SimpleOAuth2LoginAuthenticationFilter` | `OAuth2LoginAuthenticationFilter` | Handles OAuth2 callback |
| Filter | `SimpleOAuth2AuthorizationRequestRedirectFilter` | `OAuth2AuthorizationRequestRedirectFilter` | Redirects to auth server |
| Service | `OAuth2UserService` | `OAuth2UserService` | Loads user from UserInfo endpoint |
| Model | `OAuth2User` | `OAuth2User` | OAuth2 user principal |
| Configurer | `SimpleOAuth2LoginConfigurer` | `OAuth2LoginConfigurer` | DSL for OAuth2 login |

**Real source files**:
- `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/ClientRegistration.java`
- `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/registration/ClientRegistrationRepository.java`
- `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/web/OAuth2LoginAuthenticationFilter.java`
- `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/web/OAuth2AuthorizationRequestRedirectFilter.java`
- `oauth2/oauth2-core/src/main/java/org/springframework/security/oauth2/core/user/OAuth2User.java`

**Module**: `simple-security-oauth2-core` + `simple-security-oauth2-client` + `simple-security-config`
**Depends on**: Feature 2 (Authentication), Feature 4 (filter chain), Feature 5 (DSL), Feature 13 (OAuth2 core types)
**Complexity**: High · 12 new classes across 3 modules
**Java version**: 17

**Concrete deliverables**:
- [x] `simple-security-oauth2-client/.../client/registration/ClientRegistration.java`
- [x] `simple-security-oauth2-client/.../client/registration/ClientRegistrationRepository.java`
- [x] `simple-security-oauth2-client/.../client/registration/SimpleInMemoryClientRegistrationRepository.java`
- [x] `simple-security-oauth2-client/.../client/web/SimpleOAuth2LoginAuthenticationFilter.java`
- [x] `simple-security-oauth2-client/.../client/web/SimpleOAuth2AuthorizationRequestRedirectFilter.java`
- [x] `simple-security-oauth2-client/.../client/userinfo/OAuth2UserService.java`
- [x] `simple-security-oauth2-core/.../oauth2/core/user/OAuth2User.java`
- [x] `simple-security-oauth2-core/.../oauth2/core/AuthorizationGrantType.java`
- [x] `simple-security-config/.../configurers/oauth2/client/SimpleOAuth2LoginConfigurer.java`
- [x] Tests: client registration, authorization code flow (mocked)
- [x] `docs-v2/ch14_oauth2_client_login.md` — tutorial chapter

---

### Feature 15: Testing Support {{Tier: 2}}
✅ Complete

**API Contract** (what clients call):
```java
// Annotation-based mock authentication:
@Test
@WithMockUser(username = "admin", roles = "ADMIN")
void adminCanAccessDashboard() throws Exception {
    mockMvc.perform(get("/admin/dashboard"))
           .andExpect(status().isOk());
}

// MockMvc request post-processors:
@Test
void testWithCsrf() throws Exception {
    mockMvc.perform(post("/api/orders")
            .with(csrf())
            .with(user("john").roles("USER")))
           .andExpect(status().isCreated());
}

// Mock JWT:
@Test
void testJwtEndpoint() throws Exception {
    mockMvc.perform(get("/api/data")
            .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_read"))))
           .andExpect(status().isOk());
}
```

**What it does**: Provides test annotations and MockMvc post-processors that let you simulate authenticated users, CSRF tokens, and OAuth2 tokens without a running authentication infrastructure.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class | Purpose |
|-------|------------------|----------------------|---------|
| Annotation | `@WithMockUser` | `@WithMockUser` | Creates mock Authentication |
| Annotation | `@WithSecurityContext` | `@WithSecurityContext` | Meta-annotation for custom contexts |
| Factory | `SimpleWithMockUserSecurityContextFactory` | `WithMockUserSecurityContextFactory` | Builds SecurityContext from annotation |
| MockMvc | `SimpleSecurityMockMvcRequestPostProcessors` | `SecurityMockMvcRequestPostProcessors` | csrf(), user(), jwt() etc. |
| Setup | `SimpleSecurityMockMvcConfigurer` | `SecurityMockMvcConfigurers` | Applies security to MockMvc |

**Real source files**:
- `test/src/main/java/org/springframework/security/test/context/support/WithMockUser.java`
- `test/src/main/java/org/springframework/security/test/context/support/WithSecurityContext.java`
- `test/src/main/java/org/springframework/security/test/context/support/WithMockUserSecurityContextFactory.java`
- `test/src/main/java/org/springframework/security/test/web/servlet/request/SecurityMockMvcRequestPostProcessors.java`

**Module**: `simple-security-test`
**Depends on**: Feature 2 (SecurityContext, Authentication), Feature 4 (filter chain), Feature 9 (CSRF token)
**Complexity**: Medium · 6 new classes
**Java version**: 17

**Concrete deliverables**:
- [ ] `simple-security-test/.../test/context/support/WithMockUser.java`
- [ ] `simple-security-test/.../test/context/support/WithSecurityContext.java`
- [ ] `simple-security-test/.../test/context/support/SimpleWithMockUserSecurityContextFactory.java`
- [ ] `simple-security-test/.../test/web/servlet/request/SimpleSecurityMockMvcRequestPostProcessors.java`
- [ ] `simple-security-test/.../test/web/servlet/setup/SimpleSecurityMockMvcConfigurer.java`
- [ ] Tests demonstrating test utilities
- [ ] `docs-v2/ch15_testing_support.md` — tutorial chapter

---

## Implementation Notes

### Simplification Strategy

| Real Framework Concept | Simplified Version | Why Simplified |
|------------------------|--------------------|----------------|
| Reactive security (`Mono<SecurityContext>`, `ReactiveSecurityContextHolder`) | Omitted | Servlet-only focus reduces scope by ~40% while teaching the same patterns |
| XML namespace configuration (`<http>`, `<authentication-manager>`) | Omitted | Java config is the modern approach; XML adds complexity without new concepts |
| `DeferredSecurityContext` / lazy loading | Eager loading | Deferred evaluation is an optimization, not a concept |
| `ObservationAuthenticationManager` / Micrometer integration | Omitted | Observability is orthogonal to security |
| `SecurityFilterChain` ordering with `@Order` | Simplified single-chain for most features | Multi-chain is covered conceptually but not over-engineered |
| Jackson serialization of tokens | Omitted | Serialization is infrastructure, not security |
| `CompromisedPasswordChecker` | Omitted | Nice feature but not core to understanding security |
| AspectJ weaving for method security | Spring AOP only | One AOP mechanism is sufficient for learning |
| SAML 2.0 (OpenSAML) | Omitted (entire module) | Heavy external dependency; OAuth2 covers federated identity patterns |
| WebAuthn / Passkeys | Omitted (entire module) | Specialized; doesn't teach core security patterns |
| ACL (domain object security) | Omitted (entire module) | Niche use case; URL + method security covers authorization patterns |
| Kerberos / SPNEGO | Omitted (entire module) | Enterprise-specific authentication |
| CAS integration | Omitted (entire module) | Legacy SSO protocol |
| RSocket security | Omitted (entire module) | Specialized transport |
| Messaging / WebSocket security | Omitted (entire module) | Specialized transport |
| JSP taglibs (`<sec:authorize>`) | Omitted (entire module) | Template-engine specific |
| Spring Data integration | Omitted (entire module) | Thin integration layer |
| `password4j` / `Argon2` / `SCrypt` encoders | BCrypt only + NoOp | BCrypt is the default and sufficient for learning |
| `RequestCache` / saved request restoration | Simplified | Not essential to understanding the filter chain |
| Remember-Me authentication | Omitted | Nice feature but not core to the security model |

### In Scope
- Password encoding with BCrypt and delegating encoder
- Security context with ThreadLocal storage
- Full authentication pipeline (Manager → Provider → UserDetailsService)
- Servlet filter chain (FilterChainProxy → SecurityFilterChain → ordered filters)
- HttpSecurity DSL with Customizer pattern
- Form login and HTTP Basic authentication
- URL-based authorization with AuthorizationManager
- Exception translation (401/403)
- CSRF protection (session and cookie modes)
- Session management and SecurityContext persistence
- Method-level security with @PreAuthorize/@PostAuthorize
- Logout handling
- OAuth2 Resource Server with JWT
- OAuth2 Client with Authorization Code flow
- Testing utilities (@WithMockUser, MockMvc support)

### Out of Scope
- Reactive / WebFlux security stack
- XML namespace configuration
- SAML 2.0, WebAuthn, ACL, Kerberos, CAS, RSocket, Messaging
- JSP tag libraries, Spring Data integration
- Micrometer observation / metrics
- Jackson serialization support
- Multi-tenancy features
- Password migration / compromise checking
- Authorization server (issuing tokens)

### Java Version
- Base: Java 17 (matching Spring Security's compile target)
- All features use Java 17

## Module-to-Feature Mapping

| Module | Features |
|--------|----------|
| `simple-security-crypto` | Feature 1 |
| `simple-security-core` | Features 2, 3, 7 (partial), 11 (partial) |
| `simple-security-web` | Features 4, 6 (partial), 7 (partial), 8 (partial), 9 (partial), 10 (partial), 12 (partial) |
| `simple-security-config` | Features 5, 6 (partial), 7 (partial), 8 (partial), 9 (partial), 10 (partial), 11 (partial), 12 (partial), 13 (partial), 14 (partial) |
| `simple-security-oauth2-core` | Features 13 (partial), 14 (partial) |
| `simple-security-oauth2-jose` | Feature 13 (partial) |
| `simple-security-oauth2-resource-server` | Feature 13 (partial) |
| `simple-security-oauth2-client` | Feature 14 (partial) |
| `simple-security-test` | Feature 15 |

## Status Markers
- ⬜ = not started
- ✅ = implemented, tests passing, tutorial written
