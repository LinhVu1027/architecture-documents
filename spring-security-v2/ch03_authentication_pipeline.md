# Chapter 3: Authentication Pipeline

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Set up an in-memory user store:
UserDetailsService users = new SimpleInMemoryUserDetailsManager(
    User.withUsername("alice").password("{noop}secret").roles("USER").build()
);

// Wire up the pipeline:
PasswordEncoder encoder = SimplePasswordEncoderFactories.createDelegatingPasswordEncoder();
AuthenticationManager manager = new SimpleProviderManager(
    new SimpleDaoAuthenticationProvider(users, encoder));

// Submit a credential to verify:
Authentication result = manager.authenticate(
    UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret"));
// result.isAuthenticated() → true
// result.getPrincipal()    → UserDetails for "alice"
// result.getCredentials()  → null  (erased immediately after auth)
```

Questions to guide your thinking:
- Why does `SimpleProviderManager` iterate a *list* of providers instead of calling one directly?
- Why is `UsernameNotFoundException` converted to `BadCredentialsException` before propagating?
- Why is the principal a `UserDetails` object after authentication rather than just a username String?
- What does `credentials = null` in the result token mean for security?

---

## The API Contract

Feature 3 introduces the **authentication pipeline** — the mechanism that answers
"are these credentials valid, and if so, who is this user?"

There are three layers in the public API:

### Layer 1: The entry point (`AuthenticationManager`)

```java
// simple/security/authentication/AuthenticationManager.java
@FunctionalInterface
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

`AuthenticationManager` is the single interface callers use. Submit an unauthenticated
token, get back an authenticated one (or an exception). Callers never need to know which
provider handled the request.

### Layer 2: The strategy per token type (`AuthenticationProvider`)

```java
// simple/security/authentication/AuthenticationProvider.java
public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
    boolean supports(Class<?> authentication);
}
```

Each `AuthenticationProvider` knows how to handle one *type* of authentication token.
The `supports()` method lets `SimpleProviderManager` route tokens to the right provider
without trying every provider for every token type.

### Layer 3: The user store (`UserDetailsService`)

```java
// simple/security/core/userdetails/UserDetailsService.java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

`UserDetailsService` is the data access layer. It answers "given this username, give me
everything about this user." The concrete implementation can load from any source.

### Client usage example

```java
// Build the user store
UserDetailsService svc = new SimpleInMemoryUserDetailsManager(
    User.withUsername("alice")
        .password(encoder.encode("secret"))
        .roles("USER")
        .build());

// Wire the pipeline
AuthenticationManager manager = new SimpleProviderManager(
    new SimpleDaoAuthenticationProvider(svc, encoder));

// Authenticate
try {
    Authentication auth = manager.authenticate(
        UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret"));
    System.out.println(auth.getName());           // alice
    System.out.println(auth.isAuthenticated());   // true
    System.out.println(auth.getCredentials());    // null
} catch (BadCredentialsException e) {
    System.out.println("Authentication failed");
}
```

---

## Client Usage & Tests

The tests in `AuthenticationPipelineApiTest` define the full behavioral contract.
Key scenarios:

```java
@Test
void shouldReturnAuthenticatedToken_WhenCredentialsAreValid() {
    var token = UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret");
    Authentication result = authManager.authenticate(token);
    assertThat(result.isAuthenticated()).isTrue();
    assertThat(result.getName()).isEqualTo("alice");
}

@Test
void shouldEraseCredentials_AfterSuccessfulAuthentication() {
    Authentication result = authManager.authenticate(
        UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret"));
    assertThat(result.getCredentials()).isNull(); // raw password never stored
}

@Test
void shouldThrowBadCredentialsException_WhenUserNotFound() {
    // User enumeration protection: indistinguishable from wrong password
    assertThatExceptionOfType(BadCredentialsException.class)
        .isThrownBy(() -> authManager.authenticate(
            UsernamePasswordAuthenticationToken.unauthenticated("no-such-user", "any")));
}

@Test
void shouldThrowProviderNotFoundException_WhenNoProviderSupportsTokenType() {
    // A token type with no registered provider → configuration error
    assertThatExceptionOfType(ProviderNotFoundException.class)
        .isThrownBy(() -> manager.authenticate(customUnknownToken));
}
```

---

## Implementing the Call Chain

### Call chain recap

```
manager.authenticate(UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret"))
  │
  ├─► [Dispatch] SimpleProviderManager.authenticate()
  │     Iterates providers, calls supports(), tries first match
  │
  ├─► [Provider] SimpleDaoAuthenticationProvider.authenticate()
  │     Calls UserDetailsService, checks account flags, verifies password
  │
  ├─► [UserStore] SimpleInMemoryUserDetailsManager.loadUserByUsername("alice")
  │     HashMap lookup → returns UserDetails
  │
  └─► [Crypto] PasswordEncoder.matches("secret", "{bcrypt}$2a$10$...")
        Verifies password → returns true/false
```

### 3a. API Layer: `AuthenticationManager`

This is the interface clients import. It is a `@FunctionalInterface` — in tests or
simple setups, you can replace the whole pipeline with a lambda:

```java
AuthenticationManager stubManager = auth -> {
    // Return authenticated token immediately (useful in tests)
    return UsernamePasswordAuthenticationToken.authenticated(auth.getPrincipal(), null, List.of());
};
```

The real `ProviderManager` implements this interface, and so does our `SimpleProviderManager`.

### 3b. Dispatch Layer: `SimpleProviderManager`

The key insight is the **two-stage dispatch**:

```java
for (AuthenticationProvider provider : providers) {
    if (!provider.supports(authentication.getClass())) {
        continue;                         // Stage 1: skip incompatible providers
    }
    Authentication result = provider.authenticate(authentication);
    if (result != null) {
        return result;                    // Stage 2: first success wins
    }
    // null → provider declined; try next
}
```

`supports()` avoids calling every provider for every token type. `null` return from
`authenticate()` is the provider's way of saying "I support this class, but I can't
authenticate this specific request — let someone else try."

The real `ProviderManager` also handles:
- A **parent** `AuthenticationManager` as a fallback when no child provider matches
- **Event publishing** after success or failure
- Copying `details` from the request token to the result token

### 3c. Provider Layer: `SimpleDaoAuthenticationProvider`

This provider knows how to authenticate `UsernamePasswordAuthenticationToken` tokens
using a `UserDetailsService` and `PasswordEncoder`.

The authentication algorithm, step by step:

```java
// Step 1: load the user record
UserDetails user;
try {
    user = userDetailsService.loadUserByUsername(username);
} catch (UsernameNotFoundException ex) {
    throw new BadCredentialsException("Bad credentials", ex); // hide the real cause
}

// Step 2: check account status
if (!user.isEnabled())              throw new BadCredentialsException("User account is disabled");
if (!user.isAccountNonLocked())     throw new BadCredentialsException("User account is locked");
if (!user.isAccountNonExpired())    throw new BadCredentialsException("User account has expired");
if (!user.isCredentialsNonExpired()) throw new BadCredentialsException("User credentials have expired");

// Step 3: verify the password
if (!passwordEncoder.matches(presentedPassword, user.getPassword())) {
    throw new BadCredentialsException("Bad credentials");
}

// Step 4: return an authenticated token
return UsernamePasswordAuthenticationToken.authenticated(user, null, user.getAuthorities());
//                                                             ^^^^
//                                                     null credentials: raw password discarded
```

**Why null credentials?** The raw password is never needed after authentication. Storing
it in the result token would be a security risk — any code that holds a reference to
the `Authentication` object could read the plaintext password.

**Why hide `UsernameNotFoundException`?** If callers could distinguish "user not found"
from "wrong password" by catching different exceptions, they could enumerate valid
usernames by probing the system. By converting both to `BadCredentialsException`,
the pipeline reveals nothing about which users exist.

The real `DaoAuthenticationProvider` also does timing-attack mitigation: when the user
is not found, it still runs `passwordEncoder.matches()` on a pre-encoded dummy password.
This ensures the response time is the same whether or not the user exists, preventing
timing-based user enumeration.

### 3d. UserStore Layer: `SimpleInMemoryUserDetailsManager`

The simplest possible `UserDetailsService` — a `HashMap`:

```java
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    UserDetails user = users.get(username.toLowerCase(Locale.ROOT));
    if (user == null) {
        throw new UsernameNotFoundException("User '" + username + "' not found");
    }
    return user;
}
```

Username normalization (lowercase) means "Alice" and "alice" are the same user.
The real `InMemoryUserDetailsManager` uses the same approach.

---

## Try It Yourself

**Challenge 1**: Add a second user `"admin"` with role `"ADMIN"` and verify the pipeline
returns their roles correctly after authentication.

**Challenge 2**: Configure two `AuthenticationProvider`s in `SimpleProviderManager`.
Make the first one always throw `BadCredentialsException`. Verify that the second provider
still gets a chance to authenticate the request.

**Challenge 3**: Write a custom `UserDetailsService` that loads users from a `Map<String, String>`
(username → encoded password) passed to its constructor. Wire it into the pipeline.

**Challenge 4**: Implement a `NoOpPasswordEncoder` that skips hashing (for testing only)
and verify authentication works with plaintext passwords.

---

## Why This Works

**The Chain of Responsibility pattern**: `SimpleProviderManager` implements the classic
Chain of Responsibility. Instead of `if/else` logic to decide who handles a request,
it passes the request down a chain and the first handler that can process it does so.
This makes it trivial to add new authentication types (JWT, OAuth2, SAML) by adding new
providers — the manager doesn't change.

**Dependency inversion at every layer**: `SimpleProviderManager` depends on
`AuthenticationProvider` (interface), not `SimpleDaoAuthenticationProvider` (class).
`SimpleDaoAuthenticationProvider` depends on `UserDetailsService` (interface), not
`SimpleInMemoryUserDetailsManager` (class). Tests can swap any layer independently.

**The principal transformation**: Before authentication, `getPrincipal()` returns a
username String. After authentication, `getPrincipal()` returns a `UserDetails` object.
This upgrade is intentional — downstream code (authorization filters, controllers) needs
more than just the name: they need authorities, account status, and any custom user data.

---

## What We Enhanced

| Layer | Feature 1 | Feature 2 | Feature 3 (This Chapter) |
|-------|-----------|-----------|--------------------------|
| Crypto | `PasswordEncoder` / BCrypt | — | Used by `SimpleDaoAuthenticationProvider` |
| Identity model | — | `Authentication`, `UserDetails`, `User` | Used as inputs and outputs of the pipeline |
| Pipeline | — | — | `AuthenticationManager`, `SimpleProviderManager`, `SimpleDaoAuthenticationProvider` |
| User store | — | — | `UserDetailsService`, `SimpleInMemoryUserDetailsManager` |
| Exceptions | — | `AuthenticationException`, `AccessDeniedException` | `BadCredentialsException`, `UsernameNotFoundException`, `ProviderNotFoundException` |

The crypto module (Feature 1) and the identity model (Feature 2) are both now wired
together for the first time — Feature 3 is the layer that makes them cooperate.

---

## Connection to Real Framework

| Our Class | Real Framework Class | Key Difference |
|-----------|---------------------|----------------|
| `AuthenticationManager` | `o.s.s.authentication.AuthenticationManager` | Identical interface |
| `AuthenticationProvider` | `o.s.s.authentication.AuthenticationProvider` | Identical interface |
| `SimpleProviderManager` | `o.s.s.authentication.ProviderManager` | No parent manager, no event publishing, no credential erasure post-processing |
| `SimpleDaoAuthenticationProvider` | `o.s.s.authentication.dao.DaoAuthenticationProvider` | Single flat class (real splits across `AbstractUserDetailsAuthenticationProvider` + `DaoAuthenticationProvider`); no caching, no timing-attack mitigation, no password upgrade |
| `UserDetailsService` | `o.s.s.core.userdetails.UserDetailsService` | Identical interface |
| `SimpleInMemoryUserDetailsManager` | `o.s.s.provisioning.InMemoryUserDetailsManager` | Read-only (no createUser/updateUser/deleteUser); no changePassword |
| `BadCredentialsException` | `o.s.s.authentication.BadCredentialsException` | Identical structure |
| `UsernameNotFoundException` | `o.s.s.core.userdetails.UsernameNotFoundException` | Identical structure |
| `ProviderNotFoundException` | `o.s.s.authentication.ProviderNotFoundException` | Identical structure |

### Real framework: the template method split

The real `DaoAuthenticationProvider` extends `AbstractUserDetailsAuthenticationProvider`,
which defines the authentication algorithm as a template method:

```
AbstractUserDetailsAuthenticationProvider.authenticate()  ← orchestrates the flow
  ├── retrieveUser()           ← abstract, implemented by DaoAuthenticationProvider
  ├── preAuthenticationChecks  ← account status checks (locked/disabled/expired)
  ├── additionalAuthenticationChecks()  ← abstract, DaoAuthenticationProvider checks password
  ├── postAuthenticationChecks ← credentials expiry check
  └── createSuccessAuthentication()     ← builds the result token
```

Our `SimpleDaoAuthenticationProvider` collapses all of this into one `authenticate()`
method. The behavior is equivalent but the structure is flatter. A future enhancement
could introduce the template method split if you want to support other credential types
(e.g., token-based authentication) by subclassing.

---

## Complete Code

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/AuthenticationManager.java`

```java
package simple.security.authentication;

import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;

@FunctionalInterface
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/AuthenticationProvider.java`

```java
package simple.security.authentication;

import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;

public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
    boolean supports(Class<?> authentication);
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/SimpleProviderManager.java`

```java
package simple.security.authentication;

import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;

import java.util.Arrays;
import java.util.List;

public class SimpleProviderManager implements AuthenticationManager {

    private final List<AuthenticationProvider> providers;

    public SimpleProviderManager(AuthenticationProvider... providers) {
        this(Arrays.asList(providers));
    }

    public SimpleProviderManager(List<AuthenticationProvider> providers) {
        if (providers == null || providers.isEmpty()) {
            throw new IllegalArgumentException("At least one AuthenticationProvider is required");
        }
        this.providers = List.copyOf(providers);
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        AuthenticationException lastException = null;
        boolean providerFound = false;

        for (AuthenticationProvider provider : providers) {
            if (!provider.supports(authentication.getClass())) {
                continue;
            }
            providerFound = true;

            try {
                Authentication result = provider.authenticate(authentication);
                if (result != null) {
                    return result;
                }
            } catch (AuthenticationException ex) {
                lastException = ex;
            }
        }

        if (!providerFound) {
            throw new ProviderNotFoundException(
                    "No AuthenticationProvider found for " + authentication.getClass().getName());
        }
        if (lastException != null) {
            throw lastException;
        }
        throw new ProviderNotFoundException(
                "No AuthenticationProvider was able to authenticate " + authentication.getClass().getName());
    }

    public List<AuthenticationProvider> getProviders() {
        return providers;
    }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/BadCredentialsException.java`

```java
package simple.security.authentication;

import simple.security.core.AuthenticationException;

public class BadCredentialsException extends AuthenticationException {
    public BadCredentialsException(String msg) { super(msg); }
    public BadCredentialsException(String msg, Throwable cause) { super(msg, cause); }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/ProviderNotFoundException.java`

```java
package simple.security.authentication;

import simple.security.core.AuthenticationException;

public class ProviderNotFoundException extends AuthenticationException {
    public ProviderNotFoundException(String msg) { super(msg); }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/dao/SimpleDaoAuthenticationProvider.java`

```java
package simple.security.authentication.dao;

import simple.security.authentication.AuthenticationProvider;
import simple.security.authentication.BadCredentialsException;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.AuthenticationException;
import simple.security.core.userdetails.UserDetails;
import simple.security.core.userdetails.UserDetailsService;
import simple.security.core.userdetails.UsernameNotFoundException;
import simple.security.crypto.password.PasswordEncoder;

public class SimpleDaoAuthenticationProvider implements AuthenticationProvider {

    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public SimpleDaoAuthenticationProvider(UserDetailsService userDetailsService,
                                           PasswordEncoder passwordEncoder) {
        if (userDetailsService == null) throw new IllegalArgumentException("userDetailsService cannot be null");
        if (passwordEncoder == null) throw new IllegalArgumentException("passwordEncoder cannot be null");
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) authentication;
        String username = token.getName();

        UserDetails user;
        try {
            user = userDetailsService.loadUserByUsername(username);
        } catch (UsernameNotFoundException ex) {
            throw new BadCredentialsException("Bad credentials", ex);
        }

        checkAccountStatus(user);

        Object credentials = token.getCredentials();
        if (credentials == null) {
            throw new BadCredentialsException("Bad credentials");
        }
        String presentedPassword = credentials.toString();
        if (!passwordEncoder.matches(presentedPassword, user.getPassword())) {
            throw new BadCredentialsException("Bad credentials");
        }

        return UsernamePasswordAuthenticationToken.authenticated(user, null, user.getAuthorities());
    }

    private void checkAccountStatus(UserDetails user) {
        if (!user.isEnabled())               throw new BadCredentialsException("User account is disabled");
        if (!user.isAccountNonLocked())      throw new BadCredentialsException("User account is locked");
        if (!user.isAccountNonExpired())     throw new BadCredentialsException("User account has expired");
        if (!user.isCredentialsNonExpired()) throw new BadCredentialsException("User credentials have expired");
    }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/userdetails/UserDetailsService.java`

```java
package simple.security.core.userdetails;

public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/userdetails/UsernameNotFoundException.java`

```java
package simple.security.core.userdetails;

import simple.security.core.AuthenticationException;

public class UsernameNotFoundException extends AuthenticationException {
    public UsernameNotFoundException(String msg) { super(msg); }
    public UsernameNotFoundException(String msg, Throwable cause) { super(msg, cause); }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/provisioning/SimpleInMemoryUserDetailsManager.java`

```java
package simple.security.provisioning;

import simple.security.core.userdetails.UserDetails;
import simple.security.core.userdetails.UserDetailsService;
import simple.security.core.userdetails.UsernameNotFoundException;

import java.util.HashMap;
import java.util.Locale;
import java.util.Map;

public class SimpleInMemoryUserDetailsManager implements UserDetailsService {

    private final Map<String, UserDetails> users = new HashMap<>();

    public SimpleInMemoryUserDetailsManager(UserDetails... users) {
        for (UserDetails user : users) {
            this.users.put(user.getUsername().toLowerCase(Locale.ROOT), user);
        }
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserDetails user = users.get(username.toLowerCase(Locale.ROOT));
        if (user == null) {
            throw new UsernameNotFoundException("User '" + username + "' not found");
        }
        return user;
    }

    public boolean userExists(String username) {
        return users.containsKey(username.toLowerCase(Locale.ROOT));
    }
}
```

### [NEW] `simple-security-core/src/test/java/simple/security/authentication/AuthenticationPipelineApiTest.java`

> See `src/test/java/simple/security/authentication/AuthenticationPipelineApiTest.java` in the repository.
> Full test file with 13 test cases covering: successful authentication, principal as UserDetails,
> credential erasure, authority propagation, bad password, user not found (hidden), null credentials,
> disabled account, locked account, no provider match, multiple providers, and case-insensitive lookup.
