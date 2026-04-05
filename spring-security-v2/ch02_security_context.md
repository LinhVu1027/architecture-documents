# Chapter 2: Security Context & Authentication Model

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// A request comes in. Somewhere deep in the call stack, code asks: "Who is this?"
SecurityContext ctx = SecurityContextHolder.getContext();
Authentication auth = ctx.getAuthentication();
String username = auth.getName();                                    // "alice"
Collection<?> roles = auth.getAuthorities();                        // [ROLE_USER]

// Before authentication, the filter creates this:
var request = UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret");
// isAuthenticated() → false, authorities → empty

// After the auth manager verifies credentials, it creates this:
var result = UsernamePasswordAuthenticationToken.authenticated(
    userDetails, null, userDetails.getAuthorities());
// isAuthenticated() → true, authorities → user's roles

// Build a user with a fluent builder:
UserDetails user = User.withUsername("admin")
    .password("{bcrypt}$2a$10$...")
    .roles("ADMIN", "USER")
    .build();
```

Questions to guide your thinking:
- Where is the security context stored so any code on the same thread can find it?
- Why are there *two* factory methods for `UsernamePasswordAuthenticationToken`?
- Why does `roles("ADMIN")` produce `"ROLE_ADMIN"` in the authority set?

---

## The API Contract

Feature 2 introduces the **identity model** — the data structures that answer "who is the current
user and what can they do?" There are four groups of types:

### Group 1: Where is the identity stored? (`SecurityContext`)

```java
// simple/security/core/context/SecurityContextHolder.java
public final class SecurityContextHolder {
    static SecurityContext getContext()
    static void setContext(SecurityContext context)
    static void clearContext()
    static SecurityContext createEmptyContext()
}

// simple/security/core/context/SecurityContext.java
public interface SecurityContext {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

**`SecurityContextHolder`** is the single static entry point. Any code running on the same
thread — a controller, a service, an AOP interceptor — calls `getContext()` to find the
current user. The context lives in a `ThreadLocal`.

### Group 2: What is the identity? (`Authentication`)

```java
// simple/security/core/Authentication.java
public interface Authentication extends Principal {
    Collection<? extends GrantedAuthority> getAuthorities()
    Object getCredentials()    // usually password — often erased after auth
    Object getDetails()        // IP, session ID, etc.
    Object getPrincipal()      // username String before auth, UserDetails after
    boolean isAuthenticated()
    void setAuthenticated(boolean) throws IllegalArgumentException
}
```

`Authentication` is a dual-purpose interface:
- **Before authentication**: carries the credential to verify (`principal = "alice"`, `credentials = "secret"`)
- **After authentication**: carries the verified identity (`principal = UserDetails`, `credentials = null`)

### Group 3: What roles does the identity have? (`GrantedAuthority`)

```java
// simple/security/core/GrantedAuthority.java
public interface GrantedAuthority {
    String getAuthority();     // e.g., "ROLE_USER" or "ROLE_ADMIN"
}
```

### Group 4: Who is the user? (`UserDetails`)

```java
// simple/security/core/userdetails/UserDetails.java
public interface UserDetails {
    Collection<? extends GrantedAuthority> getAuthorities()
    String getPassword()
    String getUsername()
    default boolean isEnabled()               // true
    default boolean isAccountNonExpired()     // true
    default boolean isAccountNonLocked()      // true
    default boolean isCredentialsNonExpired() // true
}
```

---

## Client Usage & Tests

Here is the complete API as a client would use it:

```java
// ---- SecurityContextHolder ------------------------------------------------
SecurityContext ctx = SecurityContextHolder.getContext();   // never null
Authentication auth = ctx.getAuthentication();             // null if unauthenticated
if (auth != null && auth.isAuthenticated()) {
    String username = auth.getName();
}

// ---- Authentication tokens ------------------------------------------------
// Unauthenticated request token — carries credentials to verify:
var token = UsernamePasswordAuthenticationToken.unauthenticated("alice", "rawPass");
assert !token.isAuthenticated();
assert token.getAuthorities().isEmpty();

// Authenticated result token — returned by AuthenticationManager after verification:
UserDetails user = User.withUsername("alice").password(null).roles("USER").build();
var result = UsernamePasswordAuthenticationToken.authenticated(
    user, null, user.getAuthorities());
assert result.isAuthenticated();
assert result.getName().equals("alice"); // delegates to UserDetails.getUsername()

// ---- User builder --------------------------------------------------------
UserDetails admin = User.withUsername("admin")
    .password("{bcrypt}$2a$10$...")
    .roles("ADMIN", "USER")           // auto-prefixes with "ROLE_"
    .build();

admin.getAuthorities()  // [ROLE_ADMIN, ROLE_USER] (sorted)
admin.isEnabled()       // true (default)
```

The client-perspective tests (`SecurityContextApiTest`) verify every API promise:

```java
@Test
void shouldReturnEmptyContext_WhenNoneHasBeenSet() {
    SecurityContext context = SecurityContextHolder.getContext();
    assertThat(context).isNotNull();
    assertThat(context.getAuthentication()).isNull();
}

@Test
void shouldThrow_WhenSetAuthenticatedTrueCalledDirectly() {
    var token = UsernamePasswordAuthenticationToken.unauthenticated("alice", "secret");
    assertThatIllegalArgumentException()
            .isThrownBy(() -> token.setAuthenticated(true));
}

@Test
void shouldAutoSortAuthorities_WhenBuilding() {
    UserDetails user = User.withUsername("u").password("p")
            .roles("USER", "ADMIN").build();
    var roles = user.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority).toList();
    assertThat(roles).containsExactly("ROLE_ADMIN", "ROLE_USER"); // sorted
}
```

---

## Implementing the Call Chain (Top-Down)

### Layer 1: API Layer — `SecurityContextHolder`

The ThreadLocal is the key infrastructure:

```java
public final class SecurityContextHolder {
    private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();

    public static SecurityContext getContext() {
        SecurityContext ctx = contextHolder.get();
        if (ctx == null) {
            ctx = createEmptyContext();   // auto-vivification: never return null
            contextHolder.set(ctx);
        }
        return ctx;
    }

    public static void setContext(SecurityContext context) {
        if (context == null) throw new IllegalArgumentException(...);
        contextHolder.set(context);
    }

    public static void clearContext() {
        contextHolder.remove();   // critical: prevents memory leaks in thread pools
    }
}
```

**Why ThreadLocal?** Each incoming HTTP request is handled by a different thread from the
server's thread pool. `ThreadLocal` gives each thread its own isolated storage slot — thread
1's authenticated user is invisible to thread 2. The filter that authenticates a request stores
the result in `SecurityContextHolder.setContext(...)`, and the filter clears it in a `finally`
block after the response is sent.

### Layer 2: Model Layer — `SecurityContext` and `SecurityContextImpl`

`SecurityContext` is a thin container:

```java
public class SecurityContextImpl implements SecurityContext {
    private Authentication authentication;

    @Override
    public Authentication getAuthentication() { return authentication; }

    @Override
    public void setAuthentication(Authentication authentication) {
        this.authentication = authentication;
    }

    @Override
    public boolean equals(Object o) {   // equality by authentication content
        if (!(o instanceof SecurityContextImpl that)) return false;
        return Objects.equals(this.authentication, that.authentication);
    }
}
```

### Layer 3: Authentication Layer — `AbstractAuthenticationToken` and `UsernamePasswordAuthenticationToken`

`AbstractAuthenticationToken` handles the fields shared by all token types:

```java
public abstract class AbstractAuthenticationToken implements Authentication {
    private final List<GrantedAuthority> authorities;  // immutable after construction
    private Object details;
    private boolean authenticated = false;

    @Override
    public String getName() {
        Object principal = getPrincipal();
        if (principal instanceof UserDetails ud) return ud.getUsername();
        return principal.toString();
    }
}
```

`UsernamePasswordAuthenticationToken` enforces the security invariant via private constructors
and public factory methods:

```java
public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {

    private UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
        super(null);               // no authorities → clearly unauthenticated
        super.setAuthenticated(false);
    }

    private UsernamePasswordAuthenticationToken(Object principal, Object credentials,
            Collection<? extends GrantedAuthority> authorities) {
        super(authorities);        // authorities from the provider → clearly authenticated
        super.setAuthenticated(true);
    }

    // Only the factory methods are public:
    public static UsernamePasswordAuthenticationToken unauthenticated(...) { ... }
    public static UsernamePasswordAuthenticationToken authenticated(...) { ... }

    @Override
    public void setAuthenticated(boolean authenticated) {
        if (authenticated) throw new IllegalArgumentException(...);  // block escalation
        super.setAuthenticated(false);
    }
}
```

**Why block `setAuthenticated(true)`?** Without this guard, any code could create an
unauthenticated token and then call `token.setAuthenticated(true)` to impersonate anyone. The
factory method pattern is the only safe path to a trusted token.

### Layer 4: User Model — `User` with `UserBuilder`

`User` introduces two key design choices:

**1. Sorted authorities** — stored in a `TreeSet` keyed by `getAuthority()`:
```java
private static SortedSet<GrantedAuthority> sortAuthorities(Collection<? extends GrantedAuthority> authorities) {
    SortedSet<GrantedAuthority> sorted = new TreeSet<>(Comparator.comparing(GrantedAuthority::getAuthority));
    sorted.addAll(authorities);
    return sorted;
}
```
This ensures consistent ordering regardless of the order they were added.

**2. Equality by username only**:
```java
@Override
public boolean equals(Object o) {
    if (!(o instanceof UserDetails other)) return false;
    return username.equals(other.getUsername());
}
```
Two `User` objects with the same username but different passwords are considered equal.
This allows the in-memory user store to look up users by username without worrying about
transient state changes.

**The `roles()` vs `authorities()` distinction in the builder**:
```java
public UserBuilder roles(String... roles) {
    for (String role : roles) {
        if (role.startsWith("ROLE_")) throw new IllegalArgumentException(...);
        authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
    }
    return this;
}
```
`roles("ADMIN")` produces `"ROLE_ADMIN"`. Spring Security's access control expressions
(like `hasRole("ADMIN")`) automatically prepend `ROLE_` when checking — so if you stored
`"ROLE_ADMIN"` without the prefix, nothing would match. The guard against `"ROLE_"` in the
argument prevents double-prefixing.

---

## Try It Yourself

1. Store an authenticated token in `SecurityContextHolder` and read it back from a separate
   method (simulate what a service layer does):
   ```java
   void simulateAuthenticatedRequest() {
       var user = User.withUsername("alice").password(null).roles("USER").build();
       var token = UsernamePasswordAuthenticationToken.authenticated(user, null, user.getAuthorities());
       SecurityContextHolder.setContext(new SecurityContextImpl(token));
       // ... call your service
       ServiceLayer.doSomething();   // reads SecurityContextHolder.getContext()
       SecurityContextHolder.clearContext();
   }
   ```

2. Try calling `token.setAuthenticated(true)` on an unauthenticated token — see that it throws.

3. Build two `User` objects with the same username but different passwords and verify they're equal.

---

## Why This Works

| Design Decision | Reason |
|---|---|
| `ThreadLocal` for storage | Each request thread is isolated — no cross-request leakage |
| Auto-vivification in `getContext()` | Callers never need to null-check — the context is always safe to use |
| `clearContext()` needed in `finally` | Thread pools reuse threads — stale auth would leak to the next request |
| Factory methods + blocked `setAuthenticated(true)` | Prevents accidental or malicious privilege escalation |
| `getName()` delegates to `UserDetails.getUsername()` | After auth, principal is a `UserDetails`, not a string |
| Authorities sorted in `TreeSet` | Consistent behavior regardless of insertion order |
| User equality by username | Stable lookups in `UserDetailsService` caches |

---

## What We Enhanced

| What changed | Previous behavior | New behavior |
|---|---|---|
| Module `simple-security-core` added | Not compiled | Compiles, produces `.jar` |
| `SecurityContextHolder` available | Did not exist | Static ThreadLocal accessor |
| `Authentication` interface | Did not exist | Dual-purpose token interface |
| `User` builder | Did not exist | `User.withUsername(...).roles(...).build()` |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | File:Line | Simplification |
|---|---|---|---|
| `SecurityContextHolder` | `o.s.s.core.context.SecurityContextHolder` | `SecurityContextHolder.java:1` | Only ThreadLocal mode |
| `SecurityContextImpl` | `o.s.s.core.context.SecurityContextImpl` | `SecurityContextImpl.java:1` | Faithful |
| `Authentication` | `o.s.s.core.Authentication` | `Authentication.java:1` | No `toBuilder()` |
| `AbstractAuthenticationToken` | `o.s.s.authentication.AbstractAuthenticationToken` | `AbstractAuthenticationToken.java:1` | No serialization |
| `UsernamePasswordAuthenticationToken` | `o.s.s.authentication.UsernamePasswordAuthenticationToken` | `UsernamePasswordAuthenticationToken.java:1` | Faithful |
| `SimpleGrantedAuthority` | `o.s.s.core.authority.SimpleGrantedAuthority` | `SimpleGrantedAuthority.java:1` | Faithful |
| `User` + `UserBuilder` | `o.s.s.core.userdetails.User` | `User.java:1` | No `CredentialsContainer` interface |
| `UserDetails` | `o.s.s.core.userdetails.UserDetails` | `UserDetails.java:1` | Faithful |
| `AuthenticationException` | `o.s.s.core.AuthenticationException` | `AuthenticationException.java:1` | No `authenticationRequest` field |
| `AccessDeniedException` | `o.s.s.access.AccessDeniedException` | `AccessDeniedException.java:1` | Faithful |

---

## Complete Code

### [NEW] `simple-security-core/src/main/java/simple/security/core/GrantedAuthority.java`

```java
package simple.security.core;

public interface GrantedAuthority {
    String getAuthority();
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/Authentication.java`

```java
package simple.security.core;

import java.security.Principal;
import java.util.Collection;

public interface Authentication extends Principal {
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();
    Object getDetails();
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/AuthenticationException.java`

```java
package simple.security.core;

public abstract class AuthenticationException extends RuntimeException {
    public AuthenticationException(String msg) { super(msg); }
    public AuthenticationException(String msg, Throwable cause) { super(msg, cause); }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/context/SecurityContext.java`

```java
package simple.security.core.context;

import simple.security.core.Authentication;

public interface SecurityContext {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/context/SecurityContextImpl.java`

```java
package simple.security.core.context;

import simple.security.core.Authentication;
import java.util.Objects;

public class SecurityContextImpl implements SecurityContext {
    private Authentication authentication;

    public SecurityContextImpl() {}
    public SecurityContextImpl(Authentication authentication) { this.authentication = authentication; }

    @Override public Authentication getAuthentication() { return authentication; }
    @Override public void setAuthentication(Authentication a) { this.authentication = a; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof SecurityContextImpl that)) return false;
        return Objects.equals(authentication, that.authentication);
    }
    @Override public int hashCode() { return Objects.hashCode(authentication); }
    @Override public String toString() {
        return authentication == null ? "SecurityContextImpl [Null authentication]"
            : "SecurityContextImpl [Authentication=" + authentication + "]";
    }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/context/SecurityContextHolder.java`

```java
package simple.security.core.context;

public final class SecurityContextHolder {
    private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();

    public static SecurityContext getContext() {
        SecurityContext ctx = contextHolder.get();
        if (ctx == null) { ctx = createEmptyContext(); contextHolder.set(ctx); }
        return ctx;
    }
    public static void setContext(SecurityContext context) {
        if (context == null) throw new IllegalArgumentException("SecurityContext cannot be null");
        contextHolder.set(context);
    }
    public static void clearContext() { contextHolder.remove(); }
    public static SecurityContext createEmptyContext() { return new SecurityContextImpl(); }
    private SecurityContextHolder() {}
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/authority/SimpleGrantedAuthority.java`

```java
package simple.security.core.authority;

import simple.security.core.GrantedAuthority;
import java.util.Objects;

public final class SimpleGrantedAuthority implements GrantedAuthority {
    private final String role;
    public SimpleGrantedAuthority(String authority) {
        if (authority == null || authority.isBlank()) throw new IllegalArgumentException("authority cannot be blank");
        this.role = authority;
    }
    @Override public String getAuthority() { return role; }
    @Override public boolean equals(Object o) {
        if (!(o instanceof SimpleGrantedAuthority that)) return false;
        return Objects.equals(role, that.role);
    }
    @Override public int hashCode() { return Objects.hashCode(role); }
    @Override public String toString() { return role; }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/userdetails/UserDetails.java`

```java
package simple.security.core.userdetails;

import simple.security.core.GrantedAuthority;
import java.util.Collection;

public interface UserDetails {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    default boolean isAccountNonExpired() { return true; }
    default boolean isAccountNonLocked() { return true; }
    default boolean isCredentialsNonExpired() { return true; }
    default boolean isEnabled() { return true; }
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/core/userdetails/User.java`

See full source: `simple-security-core/src/main/java/simple/security/core/userdetails/User.java`

Key excerpt — the builder's `roles()` method:
```java
public UserBuilder roles(String... roles) {
    for (String role : roles) {
        if (role.startsWith("ROLE_")) throw new IllegalArgumentException(...);
        authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
    }
    return this;
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/AbstractAuthenticationToken.java`

See full source: `simple-security-core/src/main/java/simple/security/authentication/AbstractAuthenticationToken.java`

### [NEW] `simple-security-core/src/main/java/simple/security/authentication/UsernamePasswordAuthenticationToken.java`

See full source: `simple-security-core/src/main/java/simple/security/authentication/UsernamePasswordAuthenticationToken.java`

Key excerpt — the security invariant:
```java
@Override
public void setAuthenticated(boolean authenticated) {
    if (authenticated) throw new IllegalArgumentException(
        "Cannot set this token to trusted. Use authenticated() factory method.");
    super.setAuthenticated(false);
}
```

### [NEW] `simple-security-core/src/main/java/simple/security/access/AccessDeniedException.java`

```java
package simple.security.access;

public class AccessDeniedException extends RuntimeException {
    public AccessDeniedException(String msg) { super(msg); }
    public AccessDeniedException(String msg, Throwable cause) { super(msg, cause); }
}
```

### [NEW] `simple-security-core/src/test/java/simple/security/core/SecurityContextApiTest.java`

See full source: `simple-security-core/src/test/java/simple/security/core/SecurityContextApiTest.java`
