# Chapter 1: Password Encoding

## Build Challenge

Before reading any Spring Security source, try to implement this yourself:

```java
// A client wants to store user passwords safely. They call:
PasswordEncoder encoder = new SimpleBCryptPasswordEncoder();
String stored = encoder.encode("myPassword");     // store this in DB

// Later, at login:
boolean valid = encoder.matches("myPassword", stored); // must be true

// They also want a "universal" encoder that can read old plaintext passwords:
PasswordEncoder universal = SimplePasswordEncoderFactories.createDelegatingPasswordEncoder();
String newHash = universal.encode("secret");    // produces "{bcrypt}$2a$10$..."
boolean ok = universal.matches("old", "{noop}old"); // legacy plaintext still works
```

Questions to guide your thinking:
- What does "encoding" mean here? Why not encryption?
- How can `matches()` work if every `encode()` call produces a different hash?
- How does the delegating encoder know which algorithm decoded a stored password?

---

## The API Contract

The entire password encoding feature is built around one interface:

```java
// simple/security/crypto/password/PasswordEncoder.java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
    default boolean upgradeEncoding(String encodedPassword) { return false; }
}
```

This is the only thing clients import. The three methods capture everything:
- **encode**: Hash a raw password → produce a stored string
- **matches**: Verify a raw password against a stored hash
- **upgradeEncoding**: (optional) Should this hash be rehashed with stronger settings?

Client code programs against `PasswordEncoder`, never against `SimpleBCryptPasswordEncoder`
directly. This is the interface segregation that lets Spring Security swap algorithms
without breaking application code.

---

## Client Usage & Tests

Here is how the API is used, from a client's perspective:

```java
// Direct BCrypt usage
PasswordEncoder encoder = new SimpleBCryptPasswordEncoder();
String stored = encoder.encode("hunter2");
assert encoder.matches("hunter2", stored);      // true
assert !encoder.matches("wrong", stored);        // false

// Factory-produced delegating encoder (recommended)
PasswordEncoder encoder = SimplePasswordEncoderFactories.createDelegatingPasswordEncoder();
String stored = encoder.encode("hunter2");       // "{bcrypt}$2a$10$..."
assert encoder.matches("hunter2", stored);       // true
assert encoder.matches("old", "{noop}old");      // legacy plaintext still works
```

The client-perspective tests (`PasswordEncodingApiTest`) verify all of this without
knowing anything about BCrypt internals or the delegation mechanism:

```java
@Test
void shouldProduceDifferentHashes_ForSameRawPassword() {
    // BCrypt generates a new random salt on every encode() call
    PasswordEncoder encoder = new SimpleBCryptPasswordEncoder();
    String encoded1 = encoder.encode("password");
    String encoded2 = encoder.encode("password");

    assertThat(encoded1).isNotEqualTo(encoded2);
    assertThat(encoder.matches("password", encoded1)).isTrue();
    assertThat(encoder.matches("password", encoded2)).isTrue();
}
```

---

## Implementing the Call Chain

### Layer 1: API Surface — `PasswordEncoder`

The interface lives in `simple.security.crypto.password`. It has no dependencies.

The key insight: `encode()` is **one-way**. It produces a hash you can never reverse.
`matches()` works by re-running the same hash function with the salt embedded in the
stored hash — it does not decrypt.

### Layer 2: Crypto Layer — `SimpleBCryptPasswordEncoder`

BCrypt is an adaptive hash function. "Adaptive" means you can dial up the work factor
(the `strength` parameter) as hardware gets faster, keeping brute-force attacks hard.

```java
public class SimpleBCryptPasswordEncoder implements PasswordEncoder {

    private static final Pattern BCRYPT_PATTERN =
            Pattern.compile("\\A\\$2(a|y|b)?\\$(\\d\\d)\\$[./0-9A-Za-z]{53}");

    private final int strength; // default 10 — about 100ms on modern hardware

    @Override
    public String encode(CharSequence rawPassword) {
        if (rawPassword == null) return null;
        String salt = BCrypt.gensalt(this.strength);  // fresh random salt each time
        return BCrypt.hashpw(rawPassword.toString(), salt);
    }

    @Override
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        if (rawPassword == null || encodedPassword == null || encodedPassword.isEmpty())
            return false;
        if (!BCRYPT_PATTERN.matcher(encodedPassword).matches())
            return false;
        return BCrypt.checkpw(rawPassword.toString(), encodedPassword);
        //    ↑ BCrypt.checkpw extracts the salt from encodedPassword,
        //      re-hashes rawPassword with it, and compares
    }
}
```

**Why pattern-check first?** If `encodedPassword` is a plaintext password that slipped
through (a common misconfiguration), `BCrypt.checkpw` would throw or behave unexpectedly.
The pattern guard makes `matches()` return `false` safely instead.

**Why `org.mindrot:jbcrypt`?** The real Spring Security ships its own `BCrypt.java` (~900
lines of raw bit manipulation). We use the well-tested `jbcrypt` library instead — the
BCrypt algorithm itself is not the learning goal here; the API design pattern is.

### Layer 3: Delegation Layer — `SimpleDelegatingPasswordEncoder`

The delegating encoder solves the "migration problem": when you upgrade your hashing
algorithm, you have millions of existing passwords hashed with the old algorithm. You
can't force all users to change their passwords at once.

The solution: tag every stored password with the algorithm that produced it.

```
{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe...   ← BCrypt-hashed
{noop}myOldPlaintextPassword               ← plaintext (legacy, test-only)
```

The implementation is a routing table:

```java
public class SimpleDelegatingPasswordEncoder implements PasswordEncoder {

    private final String idForEncode;           // "bcrypt"
    private final PasswordEncoder encoderForEncode;  // BCryptPasswordEncoder
    private final Map<String, PasswordEncoder> idToEncoder;

    @Override
    public String encode(CharSequence rawPassword) {
        if (rawPassword == null) return null;
        return "{" + idForEncode + "}" + encoderForEncode.encode(rawPassword);
        //          ↑ prepend tag               ↑ delegate actual hashing
    }

    @Override
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        if (rawPassword == null || encodedPassword == null || encodedPassword.isEmpty())
            return false;
        String id = extractId(encodedPassword);       // pull "bcrypt" from "{bcrypt}..."
        PasswordEncoder delegate = idToEncoder.get(id);
        if (delegate == null) throw new IllegalArgumentException("No encoder for id: " + id);
        String stripped = extractEncodedPassword(encodedPassword); // remove "{bcrypt}"
        return delegate.matches(rawPassword, stripped);
    }
}
```

The `extractId` logic is simple string parsing:
```java
// "{bcrypt}$2a$..." → "bcrypt"
private String extractId(String prefixEncodedPassword) {
    int start = prefixEncodedPassword.indexOf('{');
    if (start != 0) return null;
    int end = prefixEncodedPassword.indexOf('}', start);
    return prefixEncodedPassword.substring(1, end);
}
```

### Layer 4: Factory Layer — `SimplePasswordEncoderFactories`

The factory wires everything together into one object:

```java
public static PasswordEncoder createDelegatingPasswordEncoder() {
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put("bcrypt", new SimpleBCryptPasswordEncoder());  // for new passwords
    encoders.put("noop", NoOpPasswordEncoder.INSTANCE);         // for legacy plaintext
    return new SimpleDelegatingPasswordEncoder("bcrypt", encoders);
}
```

This is the recommended entry point for application code. Call this once, inject the
result as a singleton, and use it everywhere.

---

## Try It Yourself

1. What happens if you call `matches("pass", "{bcrypt}notavalidhash")`? Trace the
   call through `SimpleDelegatingPasswordEncoder` → `SimpleBCryptPasswordEncoder`.

2. Add a `"sha256"` entry to `SimplePasswordEncoderFactories` that just stores
   a SHA-256 hex digest. What happens to `upgradeEncoding()` for old sha256 passwords?

3. Intentionally break the BCrypt pattern regex — what happens to `matches()`
   for valid BCrypt hashes?

---

## Why This Works

**BCrypt embeds the salt in the hash string.** A BCrypt hash like
`$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy`
contains: version (`$2a`), strength (`$10`), 22-char salt, 31-char hash.
`BCrypt.checkpw` extracts the salt, rehashes the raw password, and compares.
No separate salt column needed in the database.

**The `{id}` prefix is a future-proof contract.** When BCrypt gets too fast on GPUs
in 5 years, Spring Security can add argon2 support and set `"argon2"` as the new
default. Existing `{bcrypt}` passwords keep working. New passwords are stored with
`{argon2}`. The encoder transparently handles both.

**Null safety at every layer.** Both `encode(null)` → `null` and `matches(null, ...)` → `false`
are defined behaviors, not bugs. Spring Security's real code uses `AbstractValidatingPasswordEncoder`
to centralize this null-handling; our version inlines it in each class.

---

## What We Enhanced

| What | This Chapter (Simplified) | Future Enhancement |
|------|--------------------------|-------------------|
| BCrypt implementation | `org.mindrot:jbcrypt` library | — (algorithm detail, not the goal) |
| Null handling | Inline in each class | AbstractValidatingPasswordEncoder base class (Ch 2) |
| BCrypt versions | Default $2a only | $2a/$2b/$2y version enum |
| upgradeEncoding() | Default `false` | Strength comparison (re-encode if strength < current) |
| Factory encoders | bcrypt + noop only | + pbkdf2, scrypt, argon2, sha256, MD5, ldap... |
| ID prefix/suffix | Hard-coded `{` and `}` | Configurable via constructor |
| Unmapped ID | throws IllegalArgumentException | Configurable fallback encoder |

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | Source File (commit e43275d) |
|-----------------|---------------------|---------------------------|
| `PasswordEncoder` | `o.s.s.crypto.password.PasswordEncoder` | `crypto/.../password/PasswordEncoder.java:31` |
| `SimpleBCryptPasswordEncoder` | `o.s.s.crypto.bcrypt.BCryptPasswordEncoder` | `crypto/.../bcrypt/BCryptPasswordEncoder.java:37` |
| `SimpleDelegatingPasswordEncoder` | `o.s.s.crypto.password.DelegatingPasswordEncoder` | `crypto/.../password/DelegatingPasswordEncoder.java:128` |
| `NoOpPasswordEncoder` | `o.s.s.crypto.password.NoOpPasswordEncoder` | `crypto/.../password/NoOpPasswordEncoder.java` |
| `SimplePasswordEncoderFactories` | `o.s.s.crypto.factory.PasswordEncoderFactories` | `crypto/.../factory/PasswordEncoderFactories.java:35` |

**What we skipped:** `AbstractValidatingPasswordEncoder` (the null-safety base class,
added in Spring Security 7.0) and `BCrypt.java` (the raw algorithm — 900 lines of
Blowfish key setup and bit manipulation; we use `jbcrypt` instead).

---

## Complete Code

### [NEW] `simple-security-crypto/src/main/java/simple/security/crypto/password/PasswordEncoder.java`

```java
package simple.security.crypto.password;

public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
    default boolean upgradeEncoding(String encodedPassword) { return false; }
}
```

### [NEW] `simple-security-crypto/src/main/java/simple/security/crypto/bcrypt/SimpleBCryptPasswordEncoder.java`

```java
package simple.security.crypto.bcrypt;

import org.mindrot.jbcrypt.BCrypt;
import simple.security.crypto.password.PasswordEncoder;
import java.util.regex.Pattern;

public class SimpleBCryptPasswordEncoder implements PasswordEncoder {

    private static final Pattern BCRYPT_PATTERN =
            Pattern.compile("\\A\\$2(a|y|b)?\\$(\\d\\d)\\$[./0-9A-Za-z]{53}");
    private static final int DEFAULT_STRENGTH = 10;
    private final int strength;

    public SimpleBCryptPasswordEncoder() { this(DEFAULT_STRENGTH); }
    public SimpleBCryptPasswordEncoder(int strength) { this.strength = strength; }

    @Override
    public String encode(CharSequence rawPassword) {
        if (rawPassword == null) return null;
        return BCrypt.hashpw(rawPassword.toString(), BCrypt.gensalt(this.strength));
    }

    @Override
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        if (rawPassword == null || encodedPassword == null || encodedPassword.isEmpty()) return false;
        if (!BCRYPT_PATTERN.matcher(encodedPassword).matches()) return false;
        return BCrypt.checkpw(rawPassword.toString(), encodedPassword);
    }
}
```

### [NEW] `simple-security-crypto/src/main/java/simple/security/crypto/password/NoOpPasswordEncoder.java`

```java
package simple.security.crypto.password;

public enum NoOpPasswordEncoder implements PasswordEncoder {
    INSTANCE;

    @Override
    public String encode(CharSequence raw) { return raw == null ? null : raw.toString(); }

    @Override
    public boolean matches(CharSequence raw, String encoded) {
        if (raw == null || encoded == null) return false;
        return raw.toString().equals(encoded);
    }
}
```

### [NEW] `simple-security-crypto/src/main/java/simple/security/crypto/password/SimpleDelegatingPasswordEncoder.java`

```java
package simple.security.crypto.password;

import java.util.HashMap;
import java.util.Map;

public class SimpleDelegatingPasswordEncoder implements PasswordEncoder {

    private final String idForEncode;
    private final PasswordEncoder encoderForEncode;
    private final Map<String, PasswordEncoder> idToEncoder;

    public SimpleDelegatingPasswordEncoder(String idForEncode, Map<String, PasswordEncoder> idToEncoder) {
        if (idForEncode == null) throw new IllegalArgumentException("idForEncode cannot be null");
        if (!idToEncoder.containsKey(idForEncode))
            throw new IllegalArgumentException("idForEncode '" + idForEncode + "' not found in idToEncoder map");
        this.idForEncode = idForEncode;
        this.encoderForEncode = idToEncoder.get(idForEncode);
        this.idToEncoder = new HashMap<>(idToEncoder);
    }

    @Override
    public String encode(CharSequence rawPassword) {
        if (rawPassword == null) return null;
        return "{" + idForEncode + "}" + encoderForEncode.encode(rawPassword);
    }

    @Override
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        if (rawPassword == null || encodedPassword == null || encodedPassword.isEmpty()) return false;
        String id = extractId(encodedPassword);
        PasswordEncoder delegate = idToEncoder.get(id);
        if (delegate == null)
            throw new IllegalArgumentException("No PasswordEncoder mapped for id '" + id + "'");
        return delegate.matches(rawPassword, extractEncodedPassword(encodedPassword));
    }

    private String extractId(String s) {
        int start = s.indexOf('{');
        if (start != 0) return null;
        int end = s.indexOf('}', start);
        return end < 0 ? null : s.substring(1, end);
    }

    private String extractEncodedPassword(String s) {
        return s.substring(s.indexOf('}') + 1);
    }
}
```

### [NEW] `simple-security-crypto/src/main/java/simple/security/crypto/factory/SimplePasswordEncoderFactories.java`

```java
package simple.security.crypto.factory;

import simple.security.crypto.bcrypt.SimpleBCryptPasswordEncoder;
import simple.security.crypto.password.NoOpPasswordEncoder;
import simple.security.crypto.password.PasswordEncoder;
import simple.security.crypto.password.SimpleDelegatingPasswordEncoder;
import java.util.HashMap;
import java.util.Map;

public final class SimplePasswordEncoderFactories {
    private SimplePasswordEncoderFactories() {}

    public static PasswordEncoder createDelegatingPasswordEncoder() {
        Map<String, PasswordEncoder> encoders = new HashMap<>();
        encoders.put("bcrypt", new SimpleBCryptPasswordEncoder());
        encoders.put("noop", NoOpPasswordEncoder.INSTANCE);
        return new SimpleDelegatingPasswordEncoder("bcrypt", encoders);
    }
}
```

### [MODIFIED] `simple-security-crypto/build.gradle`

```groovy
// simple-security-crypto — Password encoding
// Mirrors: spring-security-crypto

dependencies {
    implementation 'org.mindrot:jbcrypt:0.4'
}
```
