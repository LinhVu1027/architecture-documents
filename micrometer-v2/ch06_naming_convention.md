# Chapter 6: Naming Conventions

> **Feature 6** · Tier 2 · Depends on: Feature 1 (Meter.Id, MeterRegistry.Config)

After this chapter you will be able to **configure how meter names and tag keys are formatted**
for different monitoring backends — snake_case for Prometheus, camelCase for Datadog,
dot.separated for Graphite — all from a single canonical dot-separated name in your
application code.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.NamingConvention;
import simple.micrometer.api.MeterRegistry;
import simple.micrometer.api.Meter;
import simple.micrometer.internal.SimpleMeterRegistry;
```

### What the client writes

```java
MeterRegistry registry = new SimpleMeterRegistry();

// Use snake_case naming (e.g., for Prometheus)
registry.config().namingConvention(NamingConvention.snakeCase);

Counter counter = registry.counter("httpServerRequests", "statusCode", "200");

// Convention-formatted name uses the registry's configured convention
String name = counter.getId().getConventionName(registry.getNamingConvention());
// → "httpServerRequests" (no dots to split, so unchanged)

// With dot-separated canonical names, the convention kicks in
Counter dotCounter = registry.counter("http.server.requests", "status.code", "200");
String formatted = dotCounter.getId().getConventionName(registry.getNamingConvention());
// → "http_server_requests"

// Tag keys are also formatted
List<Tag> tags = dotCounter.getId().getConventionTags(registry.getNamingConvention());
// → [status_code=200]
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Read-time formatting | Convention formatting is applied when you _read_ `getConventionName()`/`getConventionTags()`, not when the meter is registered |
| Raw Id preserved | The meter's `getId().getName()` always returns the original canonical name |
| Convention on Config | Set via `registry.config().namingConvention(NamingConvention.snakeCase)` |
| Default is dot/identity | By default, `getNamingConvention()` returns `NamingConvention.dot` (identity) |
| Switchable at any time | Changing the convention after meters are registered affects future `getConventionName()` calls |
| Tag values unchanged | Built-in conventions transform names and tag keys, but leave tag _values_ as-is |
| Functional interface | `NamingConvention` has a single abstract method — you can implement custom conventions with a lambda |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Identity convention returns names unchanged

```java
@Test
void shouldReturnNameUnchanged() {
    String result = NamingConvention.identity.name(
            "http.server.requests", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("http.server.requests");
}
```

### Snake case replaces dots with underscores

```java
@Test
void shouldConvertDotSeparatedName() {
    String result = NamingConvention.snakeCase.name(
            "http.server.requests", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("http_server_requests");
}
```

### Snake case preserves segment casing

```java
@Test
void shouldPreserveCaseOfSegments() {
    String result = NamingConvention.snakeCase.name(
            "a.Name.with.Words", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("a_Name_with_Words");
}
```

### Camel case capitalizes segments after the first

```java
@Test
void shouldConvertDotSeparatedName() {
    String result = NamingConvention.camelCase.name(
            "http.server.requests", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("httpServerRequests");
}
```

### Upper camel case capitalizes the first character too

```java
@Test
void shouldCapitalizeFirstCharacter() {
    String result = NamingConvention.upperCamelCase.name(
            "a.name.with.words", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("ANameWithWords");
}
```

### Slashes replace dots

```java
@Test
void shouldConvertDotsToSlashes() {
    String result = NamingConvention.slashes.name(
            "http.server.requests", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("http/server/requests");
}
```

### Meter.Id returns convention-formatted name and tags

```java
@Test
void shouldReturnConventionFormattedName() {
    Meter.Id id = new Meter.Id("http.server.requests",
            Tags.of("method", "GET"), null, null, Meter.Type.COUNTER);

    String name = id.getConventionName(NamingConvention.snakeCase);
    assertThat(name).isEqualTo("http_server_requests");
}

@Test
void shouldReturnConventionFormattedTags() {
    Meter.Id id = new Meter.Id("http.server.requests",
            Tags.of("status.code", "200", "http.method", "GET"),
            null, null, Meter.Type.COUNTER);

    List<Tag> tags = id.getConventionTags(NamingConvention.snakeCase);
    assertThat(tags).hasSize(2);
    assertThat(tags.get(0).getKey()).isEqualTo("http_method");
    assertThat(tags.get(0).getValue()).isEqualTo("GET");
    assertThat(tags.get(1).getKey()).isEqualTo("status_code");
    assertThat(tags.get(1).getValue()).isEqualTo("200");
}
```

### Registry defaults to dot convention

```java
@Test
void shouldDefaultToDotConvention() {
    MeterRegistry registry = new SimpleMeterRegistry();
    assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.dot);
}
```

### Convention can be changed after registration

```java
@Test
void shouldAllowChangingConventionAfterRegistration() {
    MeterRegistry registry = new SimpleMeterRegistry();
    Counter counter = registry.counter("http.server.requests", "method", "GET");

    // Initially dot convention
    assertThat(counter.getId().getConventionName(registry.getNamingConvention()))
            .isEqualTo("http.server.requests");

    // Switch to snake_case
    registry.config().namingConvention(NamingConvention.snakeCase);
    assertThat(counter.getId().getConventionName(registry.getNamingConvention()))
            .isEqualTo("http_server_requests");
}
```

---

## 3. Implementing the Call Chain

This feature is unusual — it doesn't add a new meter type and doesn't modify the
registration pipeline. Instead, it adds a **read-time formatting layer** that sits
between the stored meter identity and how backends read it:

```
Client calls: counter.getId().getConventionName(registry.getNamingConvention())
  │
  ├─► [API Layer] MeterRegistry.getNamingConvention()
  │     Returns the NamingConvention configured via config().namingConvention(...)
  │
  ├─► [API Layer] Meter.Id.getConventionName(NamingConvention)
  │     Delegates to namingConvention.name(name, type, baseUnit)
  │
  └─► [Convention Layer] NamingConvention.name(String, Type, String)
        Transforms the canonical name to the backend's format
        (e.g., dots → underscores for snakeCase)
```

The same flow applies to tags via `getConventionTags()`:

```
Client calls: counter.getId().getConventionTags(registry.getNamingConvention())
  │
  └─► [API Layer] Meter.Id.getConventionTags(NamingConvention)
        For each tag:
          key   → namingConvention.tagKey(tag.getKey())
          value → namingConvention.tagValue(tag.getValue())
        Returns a new List<Tag> with transformed keys/values
```

### Layer 1: NamingConvention — the formatting interface

This is a `@FunctionalInterface` with one abstract method and two default methods:

```java
@FunctionalInterface
public interface NamingConvention {

    String name(String name, Meter.Type type, String baseUnit);

    default String name(String name, Meter.Type type) {
        return name(name, type, null);
    }

    default String tagKey(String key) {
        return key;  // unchanged by default
    }

    default String tagValue(String value) {
        return value;  // unchanged by default
    }
}
```

Why is `tagValue()` a no-op by default? Because tag values are **user data** ("200", "GET",
"/api/users"), not identifiers. Backends don't need to reformat "200" — it's already
meaningful. Tag _keys_, on the other hand, are structural identifiers that must match the
backend's naming rules (Prometheus requires `snake_case` keys, for example).

**Built-in conventions**

The `identity` convention is a simple lambda — it's the SAM method that just returns
the name unchanged:

```java
NamingConvention identity = (name, type, baseUnit) -> name;
NamingConvention dot = identity;  // alias — Micrometer's canonical format IS dot-separated
```

The more interesting conventions are **anonymous inner classes** that override both
`name()` and `tagKey()`. Here's `snakeCase`:

```java
NamingConvention snakeCase = new NamingConvention() {
    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        return toSnakeCase(name);
    }

    @Override
    public String tagKey(String key) {
        return toSnakeCase(key);
    }

    private String toSnakeCase(String value) {
        return Arrays.stream(value.split("\\."))
                .filter(Objects::nonNull)
                .collect(Collectors.joining("_"));
    }
};
```

Why can't `snakeCase` be a simple lambda like `identity`? Because a lambda can only
implement the single abstract method (`name()`). The `snakeCase` convention also needs
to override `tagKey()` — which requires an anonymous inner class.

The `camelCase` convention is more complex because it needs to capitalize the first
letter of each segment after the first:

```java
NamingConvention camelCase = new NamingConvention() {
    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        return toCamelCase(name);
    }

    @Override
    public String tagKey(String key) {
        return toCamelCase(key);
    }

    private String toCamelCase(String value) {
        String[] parts = value.split("\\.", 0);
        StringBuilder result = new StringBuilder(value.length());
        for (int i = 0; i < parts.length; i++) {
            String str = parts[i];
            if (str.isEmpty()) continue;
            if (i == 0) {
                result.append(str);
            } else {
                char firstChar = str.charAt(0);
                if (Character.isUpperCase(firstChar)) {
                    result.append(str);
                } else {
                    result.append(Character.toUpperCase(firstChar))
                          .append(str, 1, str.length());
                }
            }
        }
        return result.toString();
    }
};
```

A subtle detail: if a segment already starts with an uppercase letter (e.g., `"GET"`),
it's appended as-is. This prevents `"GET"` from becoming `"GET"` (no change) rather
than incorrectly double-capitalizing.

The `upperCamelCase` convention delegates to `camelCase` and then capitalizes the
first character — a clean example of convention composition:

```java
NamingConvention upperCamelCase = new NamingConvention() {
    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        return capitalize(camelCase.name(name, type, baseUnit));
    }

    @Override
    public String tagKey(String key) {
        return capitalize(camelCase.tagKey(key));
    }

    private String capitalize(String name) {
        if (name.isEmpty() || Character.isUpperCase(name.charAt(0))) return name;
        char[] chars = name.toCharArray();
        chars[0] = Character.toUpperCase(chars[0]);
        return new String(chars);
    }
};
```

### Layer 2: Meter.Id — convention-aware accessors

Two new methods on `Meter.Id` bridge the gap between the stored raw name/tags and
the convention-formatted output:

```java
public String getConventionName(NamingConvention namingConvention) {
    return namingConvention.name(name, type, baseUnit);
}

public List<Tag> getConventionTags(NamingConvention namingConvention) {
    List<Tag> conventionTags = new ArrayList<>();
    for (Tag tag : tags) {
        conventionTags.add(Tag.of(
                namingConvention.tagKey(tag.getKey()),
                namingConvention.tagValue(tag.getValue())));
    }
    return Collections.unmodifiableList(conventionTags);
}
```

Key design decisions:

- **The raw name is never mutated** — `getId().getName()` always returns the original.
  Convention formatting happens at read-time, not write-time. This means a single meter
  can be read with different conventions by different backends simultaneously.
- **`getConventionName` passes `type` and `baseUnit`** to the convention. Some backends
  format names differently by meter type (e.g., appending `_total` for counters in
  Prometheus, or appending the base unit).
- **`getConventionTags` returns an unmodifiable list** — the caller can read but not
  modify the formatted tags.

### Layer 3: MeterRegistry — storing the convention

The registry holds the convention as a `volatile` field:

```java
private volatile NamingConvention namingConvention = NamingConvention.dot;

public NamingConvention getNamingConvention() {
    return namingConvention;
}
```

The `Config` inner class gets a new setter:

```java
public Config namingConvention(NamingConvention convention) {
    namingConvention = Objects.requireNonNull(convention, "convention must not be null");
    return this;
}
```

Why `volatile`? Because the convention can be changed at any time (even after meters
are registered), and reads must see the most recent write. Without `volatile`, a
thread that changes the convention might not be visible to other threads reading it.

---

## 4. Try It Yourself

1. **Create a `NamingConvention` for Prometheus**: Prometheus has specific rules — counter
   names should end with `_total`, the base unit should be appended (e.g., `_bytes`),
   and everything must be lowercase `snake_case`. Implement a `NamingConvention` that
   handles all three rules by using the `type` and `baseUnit` parameters.

2. **What happens to `getConventionTags` order?** Our implementation iterates the tags
   in their stored (sorted-by-key) order, but the convention might change the sort order
   of keys (e.g., snake_case key `"a_b"` sorts differently than dot key `"a.b"`). Does
   this matter? Consider whether backends care about tag order.

3. **Try a custom convention with a lambda**:
   ```java
   NamingConvention loud = (name, type, baseUnit) ->
       name.toUpperCase().replace('.', '_');
   ```
   This works for names, but what about tag keys? Since it's a lambda, `tagKey()` uses
   the default (unchanged). How would you make it apply to tag keys too?

---

## 5. Why This Works

### Read-time vs write-time formatting

The naming convention is applied at **read time** (`getConventionName()`), not at
**registration time** (like MeterFilters). This is a deliberate architectural split:

- **MeterFilter**: transforms the identity at write time, affecting which meter gets
  stored and how it's deduplicated. You can't "un-filter" — it's permanent.
- **NamingConvention**: formats the identity at read time for display/export purposes.
  The raw identity is preserved, and different exporters can read with different conventions.

This split enables Micrometer's key value proposition: **one registry, multiple backends**.
A `CompositeMeterRegistry` (Feature 7) can hold a Prometheus exporter (snake_case) and
a Datadog exporter (camelCase) simultaneously, each reading the same meters with their
own convention.

### @FunctionalInterface enables lambdas — but only for name-only conventions

The `@FunctionalInterface` annotation tells the compiler that `NamingConvention` has
exactly one abstract method (`name(String, Type, String)`). This means you can write:

```java
NamingConvention custom = (name, type, baseUnit) -> name.toUpperCase();
```

But this only customizes `name()` — `tagKey()` and `tagValue()` stay at their defaults
(unchanged). For conventions that also need to transform tag keys (like `snakeCase`),
you need an anonymous inner class. This is why only `identity` is a lambda, while
`snakeCase`, `camelCase`, `upperCamelCase`, and `slashes` are anonymous classes.

### Why built-in conventions don't transform tag values

Tag values are **user data** — they carry domain meaning like `"GET"`, `"200"`,
`"/api/users"`. Reformatting `"GET"` to `"get"` or `"200"` to `"two_hundred"` would
lose information. Tag keys, on the other hand, are **structural identifiers** that must
follow the backend's formatting rules (`status_code` vs `statusCode`). This asymmetry
is reflected in the interface: `tagKey()` is overridden by built-in conventions, but
`tagValue()` is always the identity function.

---

## 6. What We Enhanced

| Component | State before (Feature 5) | Enhancement in Feature 6 | Will be enhanced by |
|-----------|-------------------------|--------------------------|---------------------|
| `Meter.Id` | Name, tags, metadata; `withName()`, `withTag()`, `withTags()` | +`getConventionName(NamingConvention)`, +`getConventionTags(NamingConvention)` | Stable |
| `MeterRegistry` | Filter chain, Config with `meterFilter()`, `commonTags()` | +`namingConvention` field (volatile), +`getNamingConvention()` | F7: CompositeMeterRegistry, F8: Search |
| `MeterRegistry.Config` | `meterFilter()`, `commonTags()` | +`namingConvention(NamingConvention)` setter | Stable |

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|--------------------|
| `api/NamingConvention` | `io.micrometer.core.instrument.config.NamingConvention` | Identical structure — same built-in conventions (identity, dot, snakeCase, camelCase, upperCamelCase, slashes), same SAM method signature, same default methods. We use `str.isEmpty()` instead of `StringUtils.isEmpty()` and skip `@Nullable` annotations |
| `api/Meter.Id` (modified) | `io.micrometer.core.instrument.Meter.Id` | `getConventionName()` and `getConventionTags()` match the real implementation. Real Micrometer also caches the convention name/tags via `syntheticAssociation` — we skip caching for simplicity |
| `api/MeterRegistry` (modified) | `io.micrometer.core.instrument.MeterRegistry` | Real Micrometer's Config has additional methods (`onMeterAdded`, `onMeterRemoved`, `pauseDetector`). Our `volatile` convention field matches the real implementation's approach |

**Real source files referenced** (Micrometer `1.x` / `main` branch):
- `NamingConvention.java` — `io.micrometer.core.instrument.config.NamingConvention`
- `Meter.java` — `io.micrometer.core.instrument.Meter` (Id inner class, lines ~324-338)

---

## 8. Complete Code

All files created or modified in this feature. Copy all `[NEW]` files and replace
all `[MODIFIED]` files to build the project from Features 1–5 + Feature 6.

### `src/main/java/simple/micrometer/api/NamingConvention.java` [NEW]

```java
package simple.micrometer.api;

import java.util.Arrays;
import java.util.Objects;
import java.util.stream.Collectors;

/**
 * Monitoring systems make different recommendations regarding naming convention.
 * For example, Prometheus recommends {@code snake_case} while Graphite uses
 * {@code dot.separated} names. This interface lets registries (or users) control
 * how meter names and tag keys are formatted for a specific backend.
 *
 * <p>Micrometer's canonical format is dot-separated lowercase
 * ({@code http.server.requests}). Built-in conventions transform this canonical
 * form to each backend's preferred style.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.config.NamingConvention}.
 */
@FunctionalInterface
public interface NamingConvention {

    /**
     * Identity convention — returns the name unchanged. Since Micrometer's canonical
     * format is dot-separated, this is also the "dot" convention.
     */
    NamingConvention identity = (name, type, baseUnit) -> name;

    /**
     * Alias for {@link #identity} — Micrometer's canonical format is already
     * dot-separated, so no transformation is needed.
     */
    NamingConvention dot = identity;

    /**
     * Converts dot-separated names to snake_case.
     * {@code "http.server.requests"} becomes {@code "http_server_requests"}.
     * Also applies to tag keys.
     */
    NamingConvention snakeCase = new NamingConvention() {
        @Override
        public String name(String name, Meter.Type type, String baseUnit) {
            return toSnakeCase(name);
        }

        @Override
        public String tagKey(String key) {
            return toSnakeCase(key);
        }

        private String toSnakeCase(String value) {
            return Arrays.stream(value.split("\\."))
                    .filter(Objects::nonNull)
                    .collect(Collectors.joining("_"));
        }
    };

    /**
     * Converts dot-separated names to camelCase.
     * {@code "http.server.requests"} becomes {@code "httpServerRequests"}.
     * The first segment is kept as-is; subsequent segments have their first
     * letter capitalized. Also applies to tag keys.
     */
    NamingConvention camelCase = new NamingConvention() {
        @Override
        public String name(String name, Meter.Type type, String baseUnit) {
            return toCamelCase(name);
        }

        @Override
        public String tagKey(String key) {
            return toCamelCase(key);
        }

        private String toCamelCase(String value) {
            String[] parts = value.split("\\.", 0);
            StringBuilder result = new StringBuilder(value.length());
            for (int i = 0; i < parts.length; i++) {
                String str = parts[i];
                if (str.isEmpty()) {
                    continue;
                }
                if (i == 0) {
                    result.append(str);
                } else {
                    char firstChar = str.charAt(0);
                    if (Character.isUpperCase(firstChar)) {
                        result.append(str);
                    } else {
                        result.append(Character.toUpperCase(firstChar)).append(str, 1, str.length());
                    }
                }
            }
            return result.toString();
        }
    };

    /**
     * Converts dot-separated names to UpperCamelCase (PascalCase).
     * {@code "http.server.requests"} becomes {@code "HttpServerRequests"}.
     * Delegates to {@link #camelCase} and capitalizes the first character.
     */
    NamingConvention upperCamelCase = new NamingConvention() {
        @Override
        public String name(String name, Meter.Type type, String baseUnit) {
            return capitalize(camelCase.name(name, type, baseUnit));
        }

        @Override
        public String tagKey(String key) {
            return capitalize(camelCase.tagKey(key));
        }

        private String capitalize(String name) {
            if (name.isEmpty() || Character.isUpperCase(name.charAt(0))) {
                return name;
            }
            char[] chars = name.toCharArray();
            chars[0] = Character.toUpperCase(chars[0]);
            return new String(chars);
        }
    };

    /**
     * Converts dot-separated names to slash-separated.
     * {@code "http.server.requests"} becomes {@code "http/server/requests"}.
     * Also applies to tag keys.
     */
    NamingConvention slashes = new NamingConvention() {
        @Override
        public String name(String name, Meter.Type type, String baseUnit) {
            return toSlashes(name);
        }

        @Override
        public String tagKey(String key) {
            return toSlashes(key);
        }

        private String toSlashes(String value) {
            return Arrays.stream(value.split("\\."))
                    .filter(Objects::nonNull)
                    .collect(Collectors.joining("/"));
        }
    };

    // ── Abstract method (the single SAM method) ─────────────────────

    /**
     * Transforms a meter name according to this convention.
     *
     * @param name     the canonical dot-separated meter name
     * @param type     the meter type (some conventions format differently by type)
     * @param baseUnit the base unit (may be null; some conventions append it)
     * @return the convention-formatted name
     */
    String name(String name, Meter.Type type, String baseUnit);

    // ── Default methods ─────────────────────────────────────────────

    /**
     * Convenience overload that passes {@code null} for baseUnit.
     */
    default String name(String name, Meter.Type type) {
        return name(name, type, null);
    }

    /**
     * Transforms a tag key according to this convention.
     * Defaults to returning the key unchanged.
     */
    default String tagKey(String key) {
        return key;
    }

    /**
     * Transforms a tag value according to this convention.
     * Defaults to returning the value unchanged — tag values are user data,
     * not identifiers, so most conventions leave them as-is.
     */
    default String tagValue(String value) {
        return value;
    }

}
```

### `src/main/java/simple/micrometer/api/Meter.java` [MODIFIED]

Changes: Added `import java.util.ArrayList`, added `getConventionName(NamingConvention)` and `getConventionTags(NamingConvention)` methods to the `Id` inner class.

```java
package simple.micrometer.api;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * The root abstraction for all meters — a named and dimensioned producer
 * of one or more measurements. Every meter has a unique {@link Id} composed
 * of a name and a set of {@link Tag}s.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.Meter}.
 */
public interface Meter {

    /**
     * Returns this meter's unique identity (name + tags + metadata).
     */
    Id getId();

    /**
     * Returns the current set of measurements produced by this meter.
     */
    Iterable<Measurement> measure();

    // ── Meter Type ───────────────────────────────────────────────────

    /**
     * Classifies meters for backend-specific exposition.
     */
    enum Type {
        COUNTER,
        GAUGE,
        TIMER,
        DISTRIBUTION_SUMMARY,
        LONG_TASK_TIMER,
        OTHER
    }

    // ── Meter.Id ─────────────────────────────────────────────────────

    /**
     * Uniquely identifies a meter by name + tags. Also carries metadata
     * (description, base unit, type) that is NOT part of the identity.
     *
     * <p>Two Ids with the same name and tags but different descriptions
     * are considered equal — this is by design, matching the real Micrometer.
     */
    class Id {

        private final String name;
        private final Tags tags;
        private final String baseUnit;
        private final String description;
        private final Type type;

        public Id(String name, Tags tags, String baseUnit, String description, Type type) {
            this.name = Objects.requireNonNull(name, "name must not be null");
            this.tags = tags != null ? tags : Tags.empty();
            this.baseUnit = baseUnit;
            this.description = description;
            this.type = type;
        }

        public String getName() {
            return name;
        }

        public List<Tag> getTags() {
            return Collections.unmodifiableList(tagsAsList());
        }

        public Iterable<Tag> getTagsAsIterable() {
            return tags;
        }

        public String getTag(String key) {
            for (Tag tag : tags) {
                if (tag.getKey().equals(key)) {
                    return tag.getValue();
                }
            }
            return null;
        }

        public String getBaseUnit() {
            return baseUnit;
        }

        public String getDescription() {
            return description;
        }

        public Type getType() {
            return type;
        }

        /**
         * Returns a new Id with a different name, preserving all other fields.
         */
        public Id withName(String newName) {
            return new Id(newName, tags, baseUnit, description, type);
        }

        /**
         * Returns a new Id with an additional tag merged in.
         */
        public Id withTag(Tag tag) {
            return new Id(name, tags.and(tag), baseUnit, description, type);
        }

        /**
         * Returns a new Id with additional tags merged in.
         */
        public Id withTags(Iterable<Tag> extraTags) {
            return new Id(name, tags.and(extraTags), baseUnit, description, type);
        }

        /**
         * Returns this meter's name formatted according to the given naming convention.
         * This is how monitoring backends read the "display name" — each backend's
         * convention transforms the canonical dot-separated name to its preferred format.
         */
        public String getConventionName(NamingConvention namingConvention) {
            return namingConvention.name(name, type, baseUnit);
        }

        /**
         * Returns this meter's tags with keys and values formatted according to the
         * given naming convention.
         */
        public List<Tag> getConventionTags(NamingConvention namingConvention) {
            List<Tag> conventionTags = new ArrayList<>();
            for (Tag tag : tags) {
                conventionTags.add(Tag.of(
                        namingConvention.tagKey(tag.getKey()),
                        namingConvention.tagValue(tag.getValue())));
            }
            return Collections.unmodifiableList(conventionTags);
        }

        // Equality is based on name + tags ONLY (not description, baseUnit, type)
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Id other)) return false;
            return name.equals(other.name) && tags.equals(other.tags);
        }

        @Override
        public int hashCode() {
            return Objects.hash(name, tags);
        }

        @Override
        public String toString() {
            return "Meter.Id{name='" + name + "', tags=" + tags + '}';
        }

        private List<Tag> tagsAsList() {
            java.util.ArrayList<Tag> list = new java.util.ArrayList<>();
            for (Tag tag : tags) {
                list.add(tag);
            }
            return list;
        }

    }

}
```

### `src/main/java/simple/micrometer/api/MeterRegistry.java` [MODIFIED]

Changes: Added `volatile NamingConvention namingConvention` field, added `getNamingConvention()` method, added `namingConvention(NamingConvention)` to `Config` inner class.

```java
package simple.micrometer.api;

import simple.micrometer.internal.NoopCounter;
import simple.micrometer.internal.NoopDistributionSummary;
import simple.micrometer.internal.NoopGauge;
import simple.micrometer.internal.NoopTimer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;

/**
 * The central hub that manages meter lifecycle — creation, deduplication,
 * storage, and retrieval. Concrete subclasses (like {@code SimpleMeterRegistry})
 * decide which implementation classes to instantiate for each meter type.
 *
 * <p>Registration uses double-checked locking: the first lookup is lock-free
 * via a {@link ConcurrentHashMap}, and the synchronized block is only entered
 * when creating a new meter.
 *
 * <p>Before a meter is created, any registered {@link MeterFilter}s are applied:
 * first {@link MeterFilter#map(Meter.Id)} transforms the identity (adding common
 * tags, renaming, etc.), then {@link MeterFilter#accept(Meter.Id)} decides whether
 * the meter should be real or a noop.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.MeterRegistry}.
 */
public abstract class MeterRegistry {

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();
    private final Object meterMapLock = new Object();
    private final List<MeterFilter> filters = new CopyOnWriteArrayList<>();
    private volatile NamingConvention namingConvention = NamingConvention.dot;
    private final Config config = new Config();

    protected final Clock clock;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // ── Configuration ─────────────────────────────────────────────────

    /**
     * Returns this registry's configuration handle, through which
     * {@link MeterFilter}s and common tags can be added.
     */
    public Config config() {
        return config;
    }

    // ── Public counter convenience methods ───────────────────────────

    /**
     * Shorthand: creates or retrieves a counter with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Counter c = registry.counter("cache.misses", "cache", "users");
     * }</pre>
     */
    public Counter counter(String name, String... tags) {
        return counter(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     */
    public Counter counter(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.COUNTER);
        return counter(id);
    }

    // ── Public timer convenience methods ──────────────────────────────

    /**
     * Shorthand: creates or retrieves a timer with the given name and
     * alternating key-value tag strings.
     *
     * <pre>{@code
     * Timer t = registry.timer("http.latency", "method", "GET");
     * }</pre>
     */
    public Timer timer(String name, String... tags) {
        return timer(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     */
    public Timer timer(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.TIMER);
        return timer(id);
    }

    // ── Public summary convenience methods ──────────────────────────

    /**
     * Shorthand: creates or retrieves a distribution summary with the given
     * name and alternating key-value tag strings.
     *
     * <pre>{@code
     * DistributionSummary s = registry.summary("payload.size", "uri", "/api");
     * }</pre>
     */
    public DistributionSummary summary(String name, String... tags) {
        return summary(name, Tags.of(tags));
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     */
    public DistributionSummary summary(String name, Iterable<Tag> tags) {
        Meter.Id id = new Meter.Id(name, Tags.of(tags), null, null, Meter.Type.DISTRIBUTION_SUMMARY);
        return summary(id);
    }

    // ── Public gauge convenience methods ─────────────────────────────

    /**
     * Creates or retrieves a gauge that observes {@code stateObject} through
     * {@code valueFunction}. Returns the state object (not the Gauge) for
     * fluent assignment patterns:
     *
     * <pre>{@code
     * AtomicInteger active = registry.gauge("connections", Tags.empty(),
     *     new AtomicInteger(0), AtomicInteger::doubleValue);
     * }</pre>
     *
     * @return the {@code stateObject} passed in (for assignment convenience)
     */
    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    /**
     * Shorthand for gauging a {@link Number} subclass (AtomicInteger, AtomicLong, etc.)
     * using its {@link Number#doubleValue()}.
     */
    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    /**
     * Shorthand: gauge a Number without tags.
     */
    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    /**
     * Shorthand: gauge an object without tags.
     */
    public <T> T gauge(String name, T stateObject, ToDoubleFunction<T> valueFunction) {
        return gauge(name, Collections.emptyList(), stateObject, valueFunction);
    }

    // ── Package-private registration entry points ─────────────────────

    /**
     * Called by {@link Counter.Builder#register} and the convenience methods above.
     */
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }

    /**
     * Called by {@link Gauge.Builder#register} and the convenience methods above.
     */
    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id,
                i -> newGauge(i, obj, valueFunction), NoopGauge::new);
    }

    /**
     * Called by {@link Timer.Builder#register} and the convenience methods above.
     */
    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
    }

    /**
     * Called by {@link DistributionSummary.Builder#register} and the convenience methods above.
     */
    DistributionSummary summary(Meter.Id id) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                this::newDistributionSummary, NoopDistributionSummary::new);
    }

    // ── Abstract factory methods (subclasses implement these) ────────

    /**
     * Creates a new Counter implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract Counter newCounter(Meter.Id id);

    /**
     * Creates a new Gauge implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     *
     * @param id             the gauge's identity
     * @param obj            the state object to observe (may be held via WeakReference)
     * @param valueFunction  extracts a double from the state object
     */
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    /**
     * Creates a new Timer implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract Timer newTimer(Meter.Id id);

    /**
     * Creates a new DistributionSummary implementation for the given id.
     * Called once per unique meter id, inside a synchronized block.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id);

    // ── Query ────────────────────────────────────────────────────────

    /**
     * Returns an unmodifiable snapshot of all registered meters.
     */
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    /**
     * Returns the clock used by this registry.
     */
    public Clock getClock() {
        return clock;
    }

    /**
     * Returns the naming convention configured for this registry.
     */
    public NamingConvention getNamingConvention() {
        return namingConvention;
    }

    // ── Core registration logic ──────────────────────────────────────

    /**
     * Applies the filter chain and then performs double-checked locking
     * registration. The flow is:
     * <ol>
     *   <li>Apply all {@link MeterFilter#map} transformations to the id</li>
     *   <li>Apply all {@link MeterFilter#accept} decisions — if denied,
     *       return a noop meter</li>
     *   <li>Look up the mapped id in the meter map</li>
     *   <li>If not found, create under lock via the meter factory</li>
     * </ol>
     *
     * @param meterClass    expected concrete type (for type-safety cast)
     * @param id            the meter's identity (pre-filter)
     * @param meterFactory  creates the concrete meter (e.g., {@code this::newCounter})
     * @param noopFactory   creates a noop meter for denied registrations
     * @return the registered (or pre-existing) meter, cast to M
     * @throws IllegalArgumentException if an existing meter has the same id
     *                                  but a different type
     */
    private <M extends Meter> M registerMeterIfNecessary(
            Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, M> meterFactory,
            Function<Meter.Id, M> noopFactory) {

        // Step 1: Apply map() filters to transform the id
        Meter.Id mappedId = applyMapFilters(id);

        // Step 2: Apply accept() filters — denied meters get a noop
        if (!applyAcceptFilters(mappedId)) {
            return noopFactory.apply(mappedId);
        }

        // Step 3: Fast path — lock-free lookup with the mapped id
        Meter existing = meterMap.get(mappedId);
        if (existing != null) {
            return castOrThrow(meterClass, existing, mappedId);
        }

        // Step 4: Slow path — create under lock
        synchronized (meterMapLock) {
            existing = meterMap.get(mappedId);
            if (existing != null) {
                return castOrThrow(meterClass, existing, mappedId);
            }

            M newMeter = meterFactory.apply(mappedId);
            meterMap.put(mappedId, newMeter);
            return newMeter;
        }
    }

    /**
     * Runs every filter's {@link MeterFilter#map} in insertion order.
     * Each filter receives the output of the previous one.
     */
    private Meter.Id applyMapFilters(Meter.Id id) {
        Meter.Id mappedId = id;
        for (MeterFilter filter : filters) {
            mappedId = filter.map(mappedId);
        }
        return mappedId;
    }

    /**
     * Runs every filter's {@link MeterFilter#accept} in insertion order.
     * The first non-{@link MeterFilterReply#NEUTRAL} reply wins.
     * If all filters return NEUTRAL, the meter is accepted by default.
     *
     * @return true if the meter should be registered, false if denied
     */
    private boolean applyAcceptFilters(Meter.Id id) {
        for (MeterFilter filter : filters) {
            MeterFilterReply reply = filter.accept(id);
            if (reply == MeterFilterReply.DENY) {
                return false;
            }
            if (reply == MeterFilterReply.ACCEPT) {
                return true;
            }
            // NEUTRAL → continue to next filter
        }
        return true; // default: accept if all filters are neutral
    }

    private <M extends Meter> M castOrThrow(Class<M> meterClass, Meter existing, Meter.Id id) {
        if (!meterClass.isInstance(existing)) {
            throw new IllegalArgumentException(
                    "Meter with id '" + id.getName() + "' already exists with type "
                            + existing.getClass().getSimpleName()
                            + " but was expected to be " + meterClass.getSimpleName());
        }
        return meterClass.cast(existing);
    }

    // ── Config inner class ──────────────────────────────────────────

    /**
     * Configuration handle for a {@link MeterRegistry}. Provides methods
     * to add {@link MeterFilter}s and common tags.
     *
     * <pre>{@code
     * registry.config()
     *     .commonTags("app", "order-service", "env", "prod")
     *     .meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));
     * }</pre>
     */
    public class Config {

        /**
         * Adds a meter filter to this registry. Filters are applied in
         * insertion order during meter registration.
         *
         * @param filter the filter to add
         * @return this Config for chaining
         */
        public Config meterFilter(MeterFilter filter) {
            filters.add(Objects.requireNonNull(filter, "filter must not be null"));
            return this;
        }

        /**
         * Adds common tags that will be applied to every meter registered
         * with this registry. Meter-specific tags with the same key take
         * precedence over common tags.
         *
         * <p>This is a shorthand for
         * {@code meterFilter(MeterFilter.commonTags(Tags.of(tags)))}.
         *
         * @param tags alternating key-value strings
         * @return this Config for chaining
         */
        public Config commonTags(String... tags) {
            return meterFilter(MeterFilter.commonTags(Tags.of(tags)));
        }

        /**
         * Adds common tags that will be applied to every meter registered
         * with this registry.
         *
         * @param tags the common tags to add
         * @return this Config for chaining
         */
        public Config commonTags(Iterable<Tag> tags) {
            return meterFilter(MeterFilter.commonTags(tags));
        }

        /**
         * Sets the naming convention used by this registry to format meter
         * names and tag keys for a specific monitoring backend.
         *
         * @param convention the naming convention to use
         * @return this Config for chaining
         */
        public Config namingConvention(NamingConvention convention) {
            namingConvention = Objects.requireNonNull(convention, "convention must not be null");
            return this;
        }

    }

}
```

### `src/test/java/simple/micrometer/api/NamingConventionTest.java` [NEW]

```java
package simple.micrometer.api;

import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import simple.micrometer.internal.SimpleMeterRegistry;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for {@link NamingConvention}.
 *
 * <p>These tests verify the API contract from the user's point of view:
 * "I configure a naming convention on my registry, and when I read back
 * convention-formatted names/tags, they match my backend's expectations."
 */
class NamingConventionTest {

    // ── Identity / Dot ──────────────────────────────────────────────

    @Nested
    class IdentityConvention {

        @Test
        void shouldReturnNameUnchanged() {
            String result = NamingConvention.identity.name(
                    "http.server.requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("http.server.requests");
        }

        @Test
        void shouldReturnTagKeyUnchanged() {
            String result = NamingConvention.identity.tagKey("status.code");
            assertThat(result).isEqualTo("status.code");
        }

        @Test
        void shouldReturnTagValueUnchanged() {
            String result = NamingConvention.identity.tagValue("200");
            assertThat(result).isEqualTo("200");
        }

        @Test
        void dotShouldBeSameAsIdentity() {
            assertThat(NamingConvention.dot).isSameAs(NamingConvention.identity);
        }
    }

    // ── Snake Case ──────────────────────────────────────────────────

    @Nested
    class SnakeCaseConvention {

        @Test
        void shouldConvertDotSeparatedName() {
            String result = NamingConvention.snakeCase.name(
                    "http.server.requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("http_server_requests");
        }

        @Test
        void shouldPreserveCaseOfSegments() {
            String result = NamingConvention.snakeCase.name(
                    "a.Name.with.Words", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("a_Name_with_Words");
        }

        @Test
        void shouldConvertTagKey() {
            String result = NamingConvention.snakeCase.tagKey("a.Name.with.Words");
            assertThat(result).isEqualTo("a_Name_with_Words");
        }

        @Test
        void shouldLeaveTagValueUnchanged() {
            String result = NamingConvention.snakeCase.tagValue("/api/users");
            assertThat(result).isEqualTo("/api/users");
        }

        @Test
        void shouldHandleSingleSegment() {
            String result = NamingConvention.snakeCase.name(
                    "requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("requests");
        }
    }

    // ── Camel Case ──────────────────────────────────────────────────

    @Nested
    class CamelCaseConvention {

        @Test
        void shouldConvertDotSeparatedName() {
            String result = NamingConvention.camelCase.name(
                    "http.server.requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("httpServerRequests");
        }

        @Test
        void shouldPreserveFirstSegmentCase() {
            String result = NamingConvention.camelCase.name(
                    "a.Name.with.Words", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("aNameWithWords");
        }

        @Test
        void shouldConvertTagKey() {
            String result = NamingConvention.camelCase.tagKey("a.Name.with.Words");
            assertThat(result).isEqualTo("aNameWithWords");
        }

        @Test
        void shouldNotDoublecapitalizeAlreadyUppercaseSegments() {
            String result = NamingConvention.camelCase.name(
                    "http.GET.requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("httpGETRequests");
        }

        @Test
        void shouldHandleSingleSegment() {
            String result = NamingConvention.camelCase.name(
                    "requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("requests");
        }
    }

    // ── Upper Camel Case ────────────────────────────────────────────

    @Nested
    class UpperCamelCaseConvention {

        @Test
        void shouldCapitalizeFirstCharacter() {
            String result = NamingConvention.upperCamelCase.name(
                    "a.name.with.words", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("ANameWithWords");
        }

        @Test
        void shouldCapitalizeTagKey() {
            String result = NamingConvention.upperCamelCase.tagKey("a.name.with.words");
            assertThat(result).isEqualTo("ANameWithWords");
        }

        @Test
        void shouldNotDoublecapitalizeAlreadyUppercase() {
            String result = NamingConvention.upperCamelCase.name(
                    "Http.server.requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("HttpServerRequests");
        }
    }

    // ── Slashes ─────────────────────────────────────────────────────

    @Nested
    class SlashesConvention {

        @Test
        void shouldConvertDotsToSlashes() {
            String result = NamingConvention.slashes.name(
                    "http.server.requests", Meter.Type.COUNTER, null);
            assertThat(result).isEqualTo("http/server/requests");
        }

        @Test
        void shouldConvertTagKey() {
            String result = NamingConvention.slashes.tagKey("status.code");
            assertThat(result).isEqualTo("status/code");
        }

        @Test
        void shouldLeaveTagValueUnchanged() {
            String result = NamingConvention.slashes.tagValue("200");
            assertThat(result).isEqualTo("200");
        }
    }

    // ── Meter.Id Convention Methods ─────────────────────────────────

    @Nested
    class MeterIdConventionMethods {

        @Test
        void shouldReturnConventionFormattedName() {
            Meter.Id id = new Meter.Id("http.server.requests",
                    Tags.of("method", "GET"), null, null, Meter.Type.COUNTER);

            String name = id.getConventionName(NamingConvention.snakeCase);
            assertThat(name).isEqualTo("http_server_requests");
        }

        @Test
        void shouldReturnConventionFormattedTags() {
            Meter.Id id = new Meter.Id("http.server.requests",
                    Tags.of("status.code", "200", "http.method", "GET"),
                    null, null, Meter.Type.COUNTER);

            List<Tag> tags = id.getConventionTags(NamingConvention.snakeCase);
            assertThat(tags).hasSize(2);
            assertThat(tags.get(0).getKey()).isEqualTo("http_method");
            assertThat(tags.get(0).getValue()).isEqualTo("GET");
            assertThat(tags.get(1).getKey()).isEqualTo("status_code");
            assertThat(tags.get(1).getValue()).isEqualTo("200");
        }

        @Test
        void shouldPassTypeAndBaseUnitToConvention() {
            NamingConvention custom = (name, type, baseUnit) ->
                    baseUnit != null ? name + "." + baseUnit : name;

            Meter.Id id = new Meter.Id("http.response.size",
                    Tags.empty(), "bytes", null, Meter.Type.DISTRIBUTION_SUMMARY);

            assertThat(id.getConventionName(custom)).isEqualTo("http.response.size.bytes");
        }

        @Test
        void shouldReturnUnmodifiableTagList() {
            Meter.Id id = new Meter.Id("test", Tags.of("k", "v"),
                    null, null, Meter.Type.COUNTER);
            List<Tag> tags = id.getConventionTags(NamingConvention.identity);

            org.junit.jupiter.api.Assertions.assertThrows(
                    UnsupportedOperationException.class,
                    () -> tags.add(Tag.of("extra", "tag")));
        }
    }

    // ── Registry Integration ────────────────────────────────────────

    @Nested
    class RegistryIntegration {

        @Test
        void shouldConfigureNamingConventionOnRegistry() {
            MeterRegistry registry = new SimpleMeterRegistry();
            registry.config().namingConvention(NamingConvention.snakeCase);

            assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.snakeCase);
        }

        @Test
        void shouldDefaultToDotConvention() {
            MeterRegistry registry = new SimpleMeterRegistry();
            assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.dot);
        }

        @Test
        void shouldUseConventionWithRegisteredMeter() {
            MeterRegistry registry = new SimpleMeterRegistry();
            registry.config().namingConvention(NamingConvention.snakeCase);

            Counter counter = registry.counter("httpServerRequests", "statusCode", "200");

            assertThat(counter.getId().getName()).isEqualTo("httpServerRequests");

            String formatted = counter.getId()
                    .getConventionName(registry.getNamingConvention());
            assertThat(formatted).isEqualTo("httpServerRequests");

            Counter dotCounter = registry.counter("http.server.requests", "status.code", "200");
            assertThat(dotCounter.getId()
                    .getConventionName(registry.getNamingConvention()))
                    .isEqualTo("http_server_requests");
        }

        @Test
        void shouldSupportCustomNamingConvention() {
            NamingConvention screaming = new NamingConvention() {
                @Override
                public String name(String name, Meter.Type type, String baseUnit) {
                    return name.toUpperCase().replace('.', '_');
                }

                @Override
                public String tagKey(String key) {
                    return key.toUpperCase().replace('.', '_');
                }
            };

            MeterRegistry registry = new SimpleMeterRegistry();
            registry.config().namingConvention(screaming);

            Counter counter = registry.counter("http.server.requests", "status.code", "200");
            assertThat(counter.getId().getConventionName(screaming))
                    .isEqualTo("HTTP_SERVER_REQUESTS");
            assertThat(counter.getId().getConventionTags(screaming))
                    .extracting(Tag::getKey)
                    .containsExactly("STATUS_CODE");
        }

        @Test
        void shouldAllowChangingConventionAfterRegistration() {
            MeterRegistry registry = new SimpleMeterRegistry();
            Counter counter = registry.counter("http.server.requests", "method", "GET");

            assertThat(counter.getId().getConventionName(registry.getNamingConvention()))
                    .isEqualTo("http.server.requests");

            registry.config().namingConvention(NamingConvention.snakeCase);
            assertThat(counter.getId().getConventionName(registry.getNamingConvention()))
                    .isEqualTo("http_server_requests");
        }

        @Test
        void namingConventionConfigShouldSupportChaining() {
            MeterRegistry registry = new SimpleMeterRegistry();

            registry.config()
                    .namingConvention(NamingConvention.snakeCase)
                    .commonTags("app", "my-service");

            Counter counter = registry.counter("http.requests", "method", "GET");
            assertThat(counter.getId().getConventionName(registry.getNamingConvention()))
                    .isEqualTo("http_requests");
            assertThat(counter.getId().getTag("app")).isEqualTo("my-service");
        }
    }

    // ── Two-Argument Name Convenience ───────────────────────────────

    @Test
    void twoArgNameShouldDelegateToThreeArg() {
        NamingConvention custom = (name, type, baseUnit) ->
                name + "_" + type.name().toLowerCase();

        assertThat(custom.name("test", Meter.Type.COUNTER))
                .isEqualTo("test_counter");
    }

}
```
