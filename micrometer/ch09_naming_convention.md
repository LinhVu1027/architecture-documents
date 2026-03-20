# Chapter 9: NamingConvention

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Meters are registered with dot-notation names like `http.server.requests` and tags like `response.status` — these raw names are stored directly in `Meter.Id` | When exporting to different backends, names must match the backend's format: Prometheus expects `http_server_requests` (snake_case), Atlas expects `httpServerRequests` (camelCase), Graphite expects `http/server/requests` (slashes) — but there's no translation layer | Build a `NamingConvention` interface that translates metric names and tag keys at export time, with built-in implementations for common formats |

---

## 9.1 The Integration Point

The naming convention plugs into two places: `Meter.Id` gets new methods to apply the convention, and `MeterRegistry` gets a field to hold the registry's configured convention.

Unlike MeterFilter (which transforms IDs *during* registration), NamingConvention transforms names and tags *at export time*. The raw dot-notation name is always preserved in the Id — the convention is applied on-the-fly when a backend needs to export the metric. This means the same meter can be exported to Prometheus and Atlas simultaneously, each applying its own convention.

**Modifying:** `src/main/java/dev/linhvu/micrometer/Meter.java`
**Change:** Add `getConventionName(NamingConvention)` and `getConventionTags(NamingConvention)` methods to the `Id` inner class

```java
// --- NamingConvention support ---

public String getConventionName(NamingConvention namingConvention) {
    return namingConvention.name(name, type, baseUnit);
}

public List<Tag> getConventionTags(NamingConvention namingConvention) {
    return StreamSupport.stream(tags.spliterator(), false)
            .map(t -> Tag.of(
                    namingConvention.tagKey(t.getKey()),
                    namingConvention.tagValue(t.getValue())))
            .toList();
}
```

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add a `NamingConvention` field with getter/setter between the filter field and the `closed` field

```java
private volatile NamingConvention namingConvention = NamingConvention.identity;

public NamingConvention getNamingConvention() {
    return namingConvention;
}

public MeterRegistry namingConvention(NamingConvention namingConvention) {
    this.namingConvention = Objects.requireNonNull(namingConvention, "namingConvention must not be null");
    return this;
}
```

Two key decisions here:

1. **Why on `Meter.Id` and not on the registry's registration pipeline?** The convention is about *export format*, not meter identity. Two backends exporting the same meter should produce different names without affecting how the meter is stored or deduplicated. By keeping the convention on `Id` as a pass-through delegation, the raw name is always the canonical identity.

2. **Why `volatile` for the naming convention field?** The convention can be changed at runtime, and the export methods may be called from different threads. `volatile` ensures visibility without synchronization — the same pattern we use for the filter array.

This connects the **naming convention** to the **meter identity and registry**. To make it work, we need to build:
- `NamingConvention` — the interface with `name()`, `tagKey()`, `tagValue()` methods and built-in implementations
- `getConventionName()` / `getConventionTags()` — the delegation methods on `Meter.Id`
- `getNamingConvention()` / `namingConvention()` — the getter/setter on `MeterRegistry`

```
User code: counter("http.server.requests", "response.code", "200")
    │
    ▼
Meter.Id stores raw: name="http.server.requests", tags=[response.code=200]
    │
    │  At export time:
    ▼
id.getConventionName(registry.getNamingConvention())
    │
    ├── NamingConvention.snakeCase  →  "http_server_requests"
    ├── NamingConvention.camelCase  →  "httpServerRequests"
    └── NamingConvention.slashes    →  "http/server/requests"
```

## 9.2 NamingConvention Interface

The core interface has one abstract method (`name`) and two default methods (`tagKey`, `tagValue`). Six built-in implementations are defined as static constants on the interface.

**New file:** `src/main/java/dev/linhvu/micrometer/config/NamingConvention.java`

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Meter;

import java.util.Arrays;
import java.util.Objects;
import java.util.stream.Collectors;

public interface NamingConvention {

    NamingConvention identity = (name, type, baseUnit) -> name;

    NamingConvention dot = identity;

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
                String part = parts[i];
                if (part.isEmpty()) {
                    continue;
                }
                if (i == 0) {
                    result.append(part);
                }
                else {
                    char firstChar = part.charAt(0);
                    if (Character.isUpperCase(firstChar)) {
                        result.append(part);
                    }
                    else {
                        result.append(Character.toUpperCase(firstChar))
                                .append(part, 1, part.length());
                    }
                }
            }
            return result.toString();
        }
    };

    NamingConvention upperCamelCase = new NamingConvention() {
        @Override
        public String name(String name, Meter.Type type, String baseUnit) {
            return capitalize(camelCase.name(name, type, baseUnit));
        }

        @Override
        public String tagKey(String key) {
            return capitalize(camelCase.tagKey(key));
        }

        private String capitalize(String value) {
            if (value.isEmpty() || Character.isUpperCase(value.charAt(0))) {
                return value;
            }
            char[] chars = value.toCharArray();
            chars[0] = Character.toUpperCase(chars[0]);
            return new String(chars);
        }
    };

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

    default String name(String name, Meter.Type type) {
        return name(name, type, null);
    }

    String name(String name, Meter.Type type, String baseUnit);

    default String tagKey(String key) {
        return key;
    }

    default String tagValue(String value) {
        return value;
    }

}
```

Key points about this interface:
- **`identity`** is defined as a lambda `(name, type, baseUnit) -> name`. Since `NamingConvention` has exactly one abstract method, it qualifies as a functional interface.
- **`dot`** is aliased to `identity` — Micrometer recommends dot notation in user code, so "dot convention" and "no conversion" are the same thing.
- **`snakeCase`** and **`slashes`** split on `"\\."` (literal dot) and rejoin with `_` or `/`. The `filter(Objects::nonNull)` guards against edge cases with leading/trailing dots.
- **`camelCase`** uses a `StringBuilder` preallocated to the original length (camelCase output is always ≤ the dot-separated input). It preserves already-capitalized segments to avoid double-capitalization.
- **`upperCamelCase`** delegates to `camelCase` and capitalizes the first character — composition, not duplication.
- **None** of the built-in conventions override `tagValue()` — tag values pass through unchanged.

## 9.3 Meter.Id Convention Methods

The `Id` class gets two new methods that delegate to the convention. These are called at export time, not during registration.

**Modifying:** `src/main/java/dev/linhvu/micrometer/Meter.java`
**Change:** Add import for `NamingConvention` and `StreamSupport`, add two methods after `withBaseUnit()`

New import:
```java
import dev.linhvu.micrometer.config.NamingConvention;
import java.util.stream.StreamSupport;
```

New methods in the `Id` class:
```java
// --- NamingConvention support ---

/**
 * Returns the meter name transformed by the given naming convention.
 * Called at export time, not during registration — the raw name is preserved
 * in the Id, and the convention is applied on-the-fly.
 */
public String getConventionName(NamingConvention namingConvention) {
    return namingConvention.name(name, type, baseUnit);
}

/**
 * Returns the tags with keys and values transformed by the given naming convention.
 * Like {@link #getConventionName(NamingConvention)}, this is applied at export time.
 */
public List<Tag> getConventionTags(NamingConvention namingConvention) {
    return StreamSupport.stream(tags.spliterator(), false)
            .map(t -> Tag.of(
                    namingConvention.tagKey(t.getKey()),
                    namingConvention.tagValue(t.getValue())))
            .toList();
}
```

`getConventionName()` passes all three pieces of identity (`name`, `type`, `baseUnit`) to the convention — this is critical because backend-specific conventions like Prometheus use `type` and `baseUnit` to append suffixes (e.g., `_total` for counters, `_seconds` for timers).

`getConventionTags()` streams over the immutable `Tags` collection, applying `tagKey()` and `tagValue()` to each tag. The result is a new `List<Tag>` — the original tags are untouched.

## 9.4 MeterRegistry Convention Holder

The registry holds a configurable `NamingConvention` that backend registries use at export time.

**Modifying:** `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`
**Change:** Add import for `NamingConvention`, add field + getter + setter

New import:
```java
import dev.linhvu.micrometer.config.NamingConvention;
```

New field and methods (added between the `filters` field and the `closed` field):
```java
private volatile NamingConvention namingConvention = NamingConvention.identity;

/**
 * Returns the naming convention used by this registry when exporting metric
 * names and tag keys/values.
 */
public NamingConvention getNamingConvention() {
    return namingConvention;
}

/**
 * Sets the naming convention for this registry. Subclasses (backend registries)
 * typically set a default in their constructor (e.g., snakeCase for Prometheus).
 *
 * @param namingConvention The convention to use.
 * @return This registry, for chaining.
 */
public MeterRegistry namingConvention(NamingConvention namingConvention) {
    this.namingConvention = Objects.requireNonNull(namingConvention,
            "namingConvention must not be null");
    return this;
}
```

The default is `NamingConvention.identity` — names pass through unchanged. Backend registries (like a hypothetical Prometheus registry) would set their own default in their constructor: `this.namingConvention(NamingConvention.snakeCase)`.

---

## 9.5 Try It Yourself

Before looking at the solutions below, try implementing these on your own:

<details><summary>Challenge 1: Implement a kebab-case convention</summary>

A kebab-case convention splits on dots and joins with hyphens: `http.server.requests` → `http-server-requests`.

```java
NamingConvention kebabCase = new NamingConvention() {
    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        return toKebabCase(name);
    }

    @Override
    public String tagKey(String key) {
        return toKebabCase(key);
    }

    private String toKebabCase(String value) {
        return Arrays.stream(value.split("\\."))
                .filter(Objects::nonNull)
                .collect(Collectors.joining("-"));
    }
};
```

</details>

<details><summary>Challenge 2: Implement a Prometheus-like convention that appends _total for counters</summary>

```java
NamingConvention prometheusLike = new NamingConvention() {
    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        String snaked = name.replace('.', '_');
        if (type == Meter.Type.COUNTER) {
            return snaked + "_total";
        }
        if (baseUnit != null && !name.endsWith("." + baseUnit)) {
            return snaked + "_" + baseUnit;
        }
        return snaked;
    }

    @Override
    public String tagKey(String key) {
        return key.replace('.', '_');
    }
};

// Usage:
Meter.Id counterId = new Meter.Id("http.requests", Tags.empty(),
        Meter.Type.COUNTER, null, null);
assertThat(counterId.getConventionName(prometheusLike))
        .isEqualTo("http_requests_total");

Meter.Id timerId = new Meter.Id("http.duration", Tags.empty(),
        Meter.Type.TIMER, null, "seconds");
assertThat(timerId.getConventionName(prometheusLike))
        .isEqualTo("http_duration_seconds");
```

</details>

<details><summary>Challenge 3: Use getConventionName/Tags to "export" a meter</summary>

Write a method that takes a `Meter` and a `NamingConvention` and formats the meter as a single export line:

```java
static String export(Meter meter, NamingConvention convention) {
    Meter.Id id = meter.getId();
    String name = id.getConventionName(convention);
    List<Tag> tags = id.getConventionTags(convention);

    StringBuilder sb = new StringBuilder(name);
    if (!tags.isEmpty()) {
        sb.append('{');
        for (int i = 0; i < tags.size(); i++) {
            if (i > 0) sb.append(',');
            sb.append(tags.get(i).getKey()).append("=\"")
              .append(tags.get(i).getValue()).append('"');
        }
        sb.append('}');
    }
    return sb.toString();
}

// Usage:
SimpleMeterRegistry registry = new SimpleMeterRegistry();
Counter counter = registry.counter("http.server.requests", "method", "GET");

assertThat(export(counter, NamingConvention.snakeCase))
        .isEqualTo("http_server_requests{method=\"GET\"}");
assertThat(export(counter, NamingConvention.camelCase))
        .isEqualTo("httpServerRequests{method=\"GET\"}");
```

</details>

---

## 9.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/config/NamingConventionTest.java`

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Meter;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class NamingConventionTest {

    // --- Identity / Dot ---

    @Test
    void shouldReturnNameUnchanged_WhenUsingIdentityConvention() {
        assertThat(NamingConvention.identity.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("http.server.requests");
    }

    @Test
    void shouldReturnTagKeyUnchanged_WhenUsingIdentityConvention() {
        assertThat(NamingConvention.identity.tagKey("response.status"))
                .isEqualTo("response.status");
    }

    @Test
    void shouldReturnTagValueUnchanged_WhenUsingIdentityConvention() {
        assertThat(NamingConvention.identity.tagValue("some.value"))
                .isEqualTo("some.value");
    }

    @Test
    void shouldBeSameAsIdentity_WhenUsingDotConvention() {
        assertThat(NamingConvention.dot).isSameAs(NamingConvention.identity);
    }

    // --- Snake Case ---

    @Test
    void shouldConvertDotsToUnderscores_WhenUsingSnakeCaseConventionForName() {
        assertThat(NamingConvention.snakeCase.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("http_server_requests");
    }

    @Test
    void shouldConvertDotsToUnderscores_WhenUsingSnakeCaseConventionForTagKey() {
        assertThat(NamingConvention.snakeCase.tagKey("response.status"))
                .isEqualTo("response_status");
    }

    @Test
    void shouldReturnTagValueUnchanged_WhenUsingSnakeCaseConvention() {
        assertThat(NamingConvention.snakeCase.tagValue("some.value"))
                .isEqualTo("some.value");
    }

    @Test
    void shouldHandleSingleSegment_WhenUsingSnakeCaseConvention() {
        assertThat(NamingConvention.snakeCase.name("requests", Meter.Type.COUNTER))
                .isEqualTo("requests");
    }

    // --- Camel Case ---

    @Test
    void shouldConvertToCamelCase_WhenUsingCamelCaseConventionForName() {
        assertThat(NamingConvention.camelCase.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("httpServerRequests");
    }

    @Test
    void shouldConvertToCamelCase_WhenUsingCamelCaseConventionForTagKey() {
        assertThat(NamingConvention.camelCase.tagKey("response.status"))
                .isEqualTo("responseStatus");
    }

    @Test
    void shouldNotDoubleCapitalize_WhenSegmentAlreadyStartsWithUpperCase() {
        assertThat(NamingConvention.camelCase.name("http.Server.Requests", Meter.Type.TIMER))
                .isEqualTo("httpServerRequests");
    }

    @Test
    void shouldHandleSingleSegment_WhenUsingCamelCaseConvention() {
        assertThat(NamingConvention.camelCase.name("requests", Meter.Type.COUNTER))
                .isEqualTo("requests");
    }

    // --- Upper Camel Case ---

    @Test
    void shouldConvertToUpperCamelCase_WhenUsingUpperCamelCaseConventionForName() {
        assertThat(NamingConvention.upperCamelCase.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("HttpServerRequests");
    }

    @Test
    void shouldConvertToUpperCamelCase_WhenUsingUpperCamelCaseConventionForTagKey() {
        assertThat(NamingConvention.upperCamelCase.tagKey("response.status"))
                .isEqualTo("ResponseStatus");
    }

    @Test
    void shouldNotDoubleCapitalize_WhenAlreadyUpperCamelCase() {
        assertThat(NamingConvention.upperCamelCase.name("Http.server.requests", Meter.Type.TIMER))
                .isEqualTo("HttpServerRequests");
    }

    // --- Slashes ---

    @Test
    void shouldConvertDotsToSlashes_WhenUsingSlashesConventionForName() {
        assertThat(NamingConvention.slashes.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("http/server/requests");
    }

    @Test
    void shouldConvertDotsToSlashes_WhenUsingSlashesConventionForTagKey() {
        assertThat(NamingConvention.slashes.tagKey("response.status"))
                .isEqualTo("response/status");
    }

    // --- Functional interface / Lambda ---

    @Test
    void shouldSupportLambdaDefinition_WhenUsedAsFunctionalInterface() {
        NamingConvention upper = (name, type, baseUnit) -> name.toUpperCase();

        assertThat(upper.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("HTTP.SERVER.REQUESTS");
        assertThat(upper.tagKey("response.status")).isEqualTo("response.status");
        assertThat(upper.tagValue("200")).isEqualTo("200");
    }

    // --- Convenience two-arg overload ---

    @Test
    void shouldDelegateToThreeArgMethod_WhenCallingTwoArgName() {
        NamingConvention convention = (name, type, baseUnit) ->
                name + (baseUnit != null ? "." + baseUnit : "");

        assertThat(convention.name("request.size", Meter.Type.DISTRIBUTION_SUMMARY, "bytes"))
                .isEqualTo("request.size.bytes");
        assertThat(convention.name("request.size", Meter.Type.DISTRIBUTION_SUMMARY))
                .isEqualTo("request.size");
    }

    // --- Type and baseUnit parameters ---

    @Test
    void shouldPassTypeAndBaseUnit_WhenConventionUsesThemForSuffixing() {
        NamingConvention prometheuLike = (name, type, baseUnit) -> {
            String snaked = name.replace('.', '_');
            if (type == Meter.Type.COUNTER) {
                return snaked + "_total";
            }
            return snaked;
        };

        assertThat(prometheuLike.name("http.requests", Meter.Type.COUNTER))
                .isEqualTo("http_requests_total");
        assertThat(prometheuLike.name("http.requests", Meter.Type.TIMER))
                .isEqualTo("http_requests");
    }

}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/NamingConventionIntegrationTest.java`

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class NamingConventionIntegrationTest {

    // --- Meter.Id convention methods ---

    @Test
    void shouldTransformNameViaConvention_WhenCallingGetConventionName() {
        Meter.Id id = new Meter.Id("http.server.requests",
                Tags.of("method", "GET"), Meter.Type.TIMER, null, "seconds");

        assertThat(id.getConventionName(NamingConvention.snakeCase))
                .isEqualTo("http_server_requests");
        assertThat(id.getConventionName(NamingConvention.camelCase))
                .isEqualTo("httpServerRequests");
        assertThat(id.getConventionName(NamingConvention.identity))
                .isEqualTo("http.server.requests");
    }

    @Test
    void shouldTransformTagKeysViaConvention_WhenCallingGetConventionTags() {
        Meter.Id id = new Meter.Id("http.server.requests",
                Tags.of("response.status", "200", "http.method", "GET"),
                Meter.Type.TIMER, null, null);

        List<Tag> snakeTags = id.getConventionTags(NamingConvention.snakeCase);

        assertThat(snakeTags).hasSize(2);
        assertThat(snakeTags.get(0).getKey()).isEqualTo("http_method");
        assertThat(snakeTags.get(0).getValue()).isEqualTo("GET");
        assertThat(snakeTags.get(1).getKey()).isEqualTo("response_status");
        assertThat(snakeTags.get(1).getValue()).isEqualTo("200");
    }

    @Test
    void shouldPreserveTagValues_WhenConventionOnlyTransformsKeys() {
        Meter.Id id = new Meter.Id("test",
                Tags.of("env.name", "prod.us-east"), Meter.Type.GAUGE, null, null);

        List<Tag> tags = id.getConventionTags(NamingConvention.snakeCase);

        assertThat(tags.get(0).getKey()).isEqualTo("env_name");
        assertThat(tags.get(0).getValue()).isEqualTo("prod.us-east");
    }

    @Test
    void shouldPassTypeAndBaseUnit_WhenCallingGetConventionName() {
        Meter.Id id = new Meter.Id("http.requests",
                Tags.empty(), Meter.Type.COUNTER, null, "bytes");

        NamingConvention custom = (name, type, baseUnit) -> {
            String result = name.replace('.', '_');
            if (type == Meter.Type.COUNTER) result += "_total";
            if (baseUnit != null) result += "_" + baseUnit;
            return result;
        };

        assertThat(id.getConventionName(custom)).isEqualTo("http_requests_total_bytes");
    }

    // --- Same meter, multiple conventions ---

    @Test
    void shouldApplyDifferentConventions_WhenSameMeterExportedMultipleTimes() {
        Meter.Id id = new Meter.Id("http.server.requests",
                Tags.of("response.code", "200"), Meter.Type.TIMER, null, null);

        assertThat(id.getConventionName(NamingConvention.snakeCase))
                .isEqualTo("http_server_requests");
        assertThat(id.getConventionTags(NamingConvention.snakeCase).get(0).getKey())
                .isEqualTo("response_code");

        assertThat(id.getConventionName(NamingConvention.camelCase))
                .isEqualTo("httpServerRequests");
        assertThat(id.getConventionTags(NamingConvention.camelCase).get(0).getKey())
                .isEqualTo("responseCode");

        // Original raw name is unchanged:
        assertThat(id.getName()).isEqualTo("http.server.requests");
    }

    // --- MeterRegistry holds a convention ---

    @Test
    void shouldDefaultToIdentity_WhenNoConventionConfigured() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.identity);
    }

    @Test
    void shouldStoreConfiguredConvention_WhenSetOnRegistry() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.namingConvention(NamingConvention.snakeCase);
        assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.snakeCase);
    }

    @Test
    void shouldApplyRegistryConvention_WhenExportingRegisteredMeter() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.namingConvention(NamingConvention.snakeCase);

        Counter counter = registry.counter("http.server.requests", "response.code", "200");

        assertThat(counter.getId().getName()).isEqualTo("http.server.requests");

        NamingConvention convention = registry.getNamingConvention();
        assertThat(counter.getId().getConventionName(convention))
                .isEqualTo("http_server_requests");
        assertThat(counter.getId().getConventionTags(convention).get(0).getKey())
                .isEqualTo("response_code");
    }

    @Test
    void shouldSupportChainingConventionSetter_WhenConfiguringRegistry() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        MeterRegistry result = registry.namingConvention(NamingConvention.camelCase);
        assertThat(result).isSameAs(registry);
    }

    @Test
    void shouldAllowConventionChange_WhenRegistryAlreadyHasMeters() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        Timer timer = registry.timer("http.server.requests", "method", "GET");

        registry.namingConvention(NamingConvention.upperCamelCase);

        assertThat(timer.getId().getConventionName(registry.getNamingConvention()))
                .isEqualTo("HttpServerRequests");
    }

}
```

---

## 9.7 Why This Works

`★ Insight ─────────────────────────────────────`
**Export-time vs registration-time transformation.** MeterFilter and NamingConvention look similar — both transform meter names and tags. But they operate at fundamentally different points: MeterFilter transforms the raw identity *during registration* (affecting deduplication and storage), while NamingConvention transforms the exported format *at scrape/publish time* (leaving the stored identity untouched). This means a single meter stored as `http.server.requests` can simultaneously appear as `http_server_requests` in Prometheus and `httpServerRequests` in Atlas. If the convention were baked into the Id at registration time, you'd need separate meter instances per backend — which would break `CompositeMeterRegistry` (Feature 14).
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Functional interface with default methods.** `NamingConvention` has one abstract method (`name`) and two default methods (`tagKey`, `tagValue`). This makes it a functional interface — you can define a custom convention with a lambda: `(name, type, baseUnit) -> name.toUpperCase()`. The `type` and `baseUnit` parameters exist because backend-specific conventions need them. Prometheus appends `_total` for counters and `_seconds` for timers. Most conventions ignore these parameters, but having them in the interface means Prometheus doesn't need a separate convention contract — it just implements the same interface with richer logic.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Dot as canonical form.** Micrometer's convention is that users always write dot-separated names (`http.server.requests`). The `dot` constant is literally aliased to `identity` — they are the *same object*. All other conventions convert *from* dot notation. This establishes a clear contract: instrumentation code writes dots, backends translate from dots. No convention needs to handle arbitrary input formats (snake_case input → camelCase output) because the input is always dots.
`─────────────────────────────────────────────────`

---

## 9.8 What We Enhanced

| File | What Changed | Why |
|------|-------------|-----|
| `Meter.java` | Added `getConventionName()` and `getConventionTags()` to `Id`; added imports for `NamingConvention` and `StreamSupport` | These methods are the bridge between the raw identity and the backend-specific export format |
| `MeterRegistry.java` | Added `NamingConvention` field (volatile), `getNamingConvention()` getter, `namingConvention()` setter; added import for `NamingConvention` | Each registry holds its own convention, enabling backend registries to set a default (e.g., snakeCase for Prometheus) |

---

## 9.9 Connection to Real Framework

| Simplified | Real Micrometer | Location |
|-----------|----------------|----------|
| `NamingConvention` interface | `NamingConvention` | `micrometer-core/.../config/NamingConvention.java:38` |
| `identity` constant | `NamingConvention.identity` | `NamingConvention.java:40` |
| `dot` alias | `NamingConvention.dot` | `NamingConvention.java:46` |
| `snakeCase` implementation | `NamingConvention.snakeCase` | `NamingConvention.java:48` |
| `camelCase` implementation | `NamingConvention.camelCase` | `NamingConvention.java:64` |
| `upperCamelCase` implementation | `NamingConvention.upperCamelCase` | `NamingConvention.java:100` |
| `slashes` implementation | `NamingConvention.slashes` | `NamingConvention.java:125` |
| `Id.getConventionName()` | `Meter.Id.getConventionName()` | `Meter.java:324` |
| `Id.getConventionTags()` | `Meter.Id.getConventionTags()` | `Meter.java:334` |

**Simplifications vs real framework:**
- The real `camelCase` uses `StringUtils.isEmpty()` from Micrometer's common module; we use `part.isEmpty()` (equivalent for non-null strings)
- The real `name()` method accepts `@Nullable String baseUnit`; we use plain `String` (no nullability annotations)
- The real framework has more extensive NamingConvention subclasses in backend modules (e.g., `PrometheusNamingConvention`, `DatadogNamingConvention`) — we only implement the built-in constants

---

## 9.10 Complete Code

### Production Code

#### [NEW] `src/main/java/dev/linhvu/micrometer/config/NamingConvention.java`

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Meter;

import java.util.Arrays;
import java.util.Objects;
import java.util.stream.Collectors;

/**
 * Translates metric names and tag keys/values for different monitoring backends.
 * <p>
 * Monitoring systems have different naming conventions: Prometheus uses snake_case,
 * Atlas uses camelCase, Graphite uses dot-separated names. This interface abstracts
 * the translation so instrumentation code can use a single canonical format (dot notation)
 * and each backend applies its own convention at export time.
 * <p>
 * The single abstract method is {@link #name(String, Meter.Type, String)}, making
 * this a functional interface — you can define a convention with a lambda.
 * {@link #tagKey(String)} and {@link #tagValue(String)} have sensible defaults (identity).
 */
public interface NamingConvention {

    /**
     * Identity convention — returns names and tags unchanged.
     */
    NamingConvention identity = (name, type, baseUnit) -> name;

    /**
     * Dot notation convention. Aliased to {@link #identity} because Micrometer recommends
     * dot-separated names in user code — so dot convention IS identity.
     */
    NamingConvention dot = identity;

    /**
     * Snake_case convention — splits on dots and joins with underscores.
     * "http.server.requests" → "http_server_requests"
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
     * camelCase convention — splits on dots, keeps the first segment lowercase,
     * capitalizes the first character of subsequent segments.
     * "http.server.requests" → "httpServerRequests"
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
                String part = parts[i];
                if (part.isEmpty()) {
                    continue;
                }
                if (i == 0) {
                    result.append(part);
                }
                else {
                    char firstChar = part.charAt(0);
                    if (Character.isUpperCase(firstChar)) {
                        result.append(part);
                    }
                    else {
                        result.append(Character.toUpperCase(firstChar)).append(part, 1, part.length());
                    }
                }
            }
            return result.toString();
        }
    };

    /**
     * UpperCamelCase convention — same as camelCase but capitalizes the first character.
     * "http.server.requests" → "HttpServerRequests"
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

        private String capitalize(String value) {
            if (value.isEmpty() || Character.isUpperCase(value.charAt(0))) {
                return value;
            }
            char[] chars = value.toCharArray();
            chars[0] = Character.toUpperCase(chars[0]);
            return new String(chars);
        }
    };

    /**
     * Slash convention — splits on dots and joins with slashes.
     * "http.server.requests" → "http/server/requests"
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

    /**
     * Convenience overload that delegates to the 3-arg form with null baseUnit.
     */
    default String name(String name, Meter.Type type) {
        return name(name, type, null);
    }

    /**
     * Transforms a raw metric name for the monitoring backend.
     *
     * @param name     The raw metric name in dot notation (e.g., "http.server.requests").
     * @param type     The meter type — some backends append suffixes based on type
     *                 (e.g., Prometheus appends "_total" for counters).
     * @param baseUnit The meter's base unit (e.g., "bytes", "seconds"), or null.
     * @return The transformed name.
     */
    String name(String name, Meter.Type type, String baseUnit);

    /**
     * Transforms a tag key. Default implementation returns the key unchanged.
     */
    default String tagKey(String key) {
        return key;
    }

    /**
     * Transforms a tag value. Default implementation returns the value unchanged.
     * None of the built-in conventions override this — tag values are passed through as-is.
     */
    default String tagValue(String value) {
        return value;
    }

}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/Meter.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.config.NamingConvention;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.stream.StreamSupport;

/**
 * A named and tagged set of measurements. This is the base type for all meters.
 * <p>
 * Every meter has a unique {@link Id} (name + tags) and produces an {@link Iterable}
 * of {@link Measurement}s. The {@code match/use} visitor methods provide type-safe
 * dispatching across the concrete meter types without instanceof chains.
 */
public interface Meter {

    /**
     * Returns the unique identity of this meter — its name and tags.
     */
    Id getId();

    /**
     * Returns the current set of measurements for this meter.
     * A counter returns one measurement (COUNT); a timer returns three
     * (COUNT, TOTAL_TIME, MAX). The number and order of measurements
     * for a given meter never changes.
     */
    Iterable<Measurement> measure();

    /**
     * Type-safe visitor that returns a value. One function per meter type, plus a
     * fallback for custom meters. If new meter types are added, this method signature
     * will change — intentionally forcing a compile error at every call site so nothing
     * is silently missed.
     * <p>
     * Ordering matters: TimeGauge is checked before Gauge because TimeGauge extends Gauge.
     */
    @SuppressWarnings("unchecked")
    default <T> T match(
            Function<Counter, T> visitCounter,
            Function<Gauge, T> visitGauge,
            Function<Timer, T> visitTimer,
            Function<DistributionSummary, T> visitDistributionSummary,
            Function<LongTaskTimer, T> visitLongTaskTimer,
            Function<FunctionCounter, T> visitFunctionCounter,
            Function<FunctionTimer, T> visitFunctionTimer,
            Function<Meter, T> defaultFunction) {
        // Order matters: more specific types must be checked first
        if (this instanceof Counter c) return visitCounter.apply(c);
        if (this instanceof Timer t) return visitTimer.apply(t);
        if (this instanceof LongTaskTimer ltt) return visitLongTaskTimer.apply(ltt);
        if (this instanceof DistributionSummary ds) return visitDistributionSummary.apply(ds);
        if (this instanceof FunctionCounter fc) return visitFunctionCounter.apply(fc);
        if (this instanceof FunctionTimer ft) return visitFunctionTimer.apply(ft);
        if (this instanceof Gauge g) return visitGauge.apply(g);
        return defaultFunction.apply(this);
    }

    /**
     * Type-safe visitor for side-effect operations (no return value).
     */
    default void use(
            Consumer<Counter> visitCounter,
            Consumer<Gauge> visitGauge,
            Consumer<Timer> visitTimer,
            Consumer<DistributionSummary> visitDistributionSummary,
            Consumer<LongTaskTimer> visitLongTaskTimer,
            Consumer<FunctionCounter> visitFunctionCounter,
            Consumer<FunctionTimer> visitFunctionTimer,
            Consumer<Meter> defaultConsumer) {
        match(
                counter -> { visitCounter.accept(counter); return null; },
                gauge -> { visitGauge.accept(gauge); return null; },
                timer -> { visitTimer.accept(timer); return null; },
                ds -> { visitDistributionSummary.accept(ds); return null; },
                ltt -> { visitLongTaskTimer.accept(ltt); return null; },
                fc -> { visitFunctionCounter.accept(fc); return null; },
                ft -> { visitFunctionTimer.accept(ft); return null; },
                meter -> { defaultConsumer.accept(meter); return null; }
        );
    }

    /**
     * Enumerates the different types of meters.
     */
    enum Type {

        COUNTER,
        GAUGE,
        LONG_TASK_TIMER,
        TIMER,
        DISTRIBUTION_SUMMARY,
        OTHER

    }

    /**
     * Uniquely identifies a meter by its name and tags.
     * <p>
     * Two IDs with the same name and tags are considered equal, regardless of
     * type, description, or base unit. This is intentional — the registry uses
     * name+tags as the lookup key, and metadata is supplementary.
     * <p>
     * All {@code with*} methods return new instances — Id is immutable.
     */
    class Id {

        private final String name;

        private final Tags tags;

        private final Type type;

        private final String description;

        private final String baseUnit;

        public Id(String name, Tags tags, Type type, String description, String baseUnit) {
            Objects.requireNonNull(name, "name must not be null");
            this.name = name;
            this.tags = (tags != null) ? tags : Tags.empty();
            this.type = type;
            this.description = description;
            this.baseUnit = baseUnit;
        }

        // --- Accessors ---

        public String getName() {
            return name;
        }

        public Tags getTagsAsIterable() {
            return tags;
        }

        /**
         * Returns an unmodifiable list of tags. Prefer {@link #getTagsAsIterable()}
         * for iteration to avoid the copy.
         */
        public List<Tag> getTags() {
            List<Tag> list = new ArrayList<>();
            tags.forEach(list::add);
            return Collections.unmodifiableList(list);
        }

        /**
         * Looks up a tag value by key via linear scan. Returns null if not found.
         */
        public String getTag(String key) {
            for (Tag tag : tags) {
                if (tag.getKey().equals(key)) {
                    return tag.getValue();
                }
            }
            return null;
        }

        public Type getType() {
            return type;
        }

        public String getDescription() {
            return description;
        }

        public String getBaseUnit() {
            return baseUnit;
        }

        // --- Immutable copy methods ---

        public Id withName(String newName) {
            return new Id(newName, tags, type, description, baseUnit);
        }

        public Id withTag(Tag tag) {
            return new Id(name, tags.and(tag), type, description, baseUnit);
        }

        public Id withTags(Iterable<Tag> newTags) {
            return new Id(name, tags.and(newTags), type, description, baseUnit);
        }

        public Id replaceTags(Iterable<Tag> newTags) {
            return new Id(name, Tags.of(newTags), type, description, baseUnit);
        }

        /**
         * Adds a "statistic" tag with the given statistic's tag-value representation.
         * Used when exporting individual measurements as separate time series.
         */
        public Id withTag(Statistic statistic) {
            return withTag(Tag.of("statistic", statistic.getTagValueRepresentation()));
        }

        public Id withBaseUnit(String newBaseUnit) {
            return new Id(name, tags, type, description, newBaseUnit);
        }

        // --- NamingConvention support ---

        /**
         * Returns the meter name transformed by the given naming convention.
         * Called at export time, not during registration — the raw name is preserved
         * in the Id, and the convention is applied on-the-fly.
         */
        public String getConventionName(NamingConvention namingConvention) {
            return namingConvention.name(name, type, baseUnit);
        }

        /**
         * Returns the tags with keys and values transformed by the given naming convention.
         * Like {@link #getConventionName(NamingConvention)}, this is applied at export time.
         */
        public List<Tag> getConventionTags(NamingConvention namingConvention) {
            return StreamSupport.stream(tags.spliterator(), false)
                    .map(t -> Tag.of(
                            namingConvention.tagKey(t.getKey()),
                            namingConvention.tagValue(t.getValue())))
                    .toList();
        }

        // --- Identity: name + tags only ---

        @Override
        public boolean equals(Object o) {
            if (this == o)
                return true;
            if (!(o instanceof Id other))
                return false;
            return name.equals(other.name) && tags.equals(other.tags);
        }

        @Override
        public int hashCode() {
            int result = name.hashCode();
            result = 31 * result + tags.hashCode();
            return result;
        }

        @Override
        public String toString() {
            return "Id{name='" + name + "', tags=" + tags + '}';
        }

    }

}
```

#### [MODIFIED] `src/main/java/dev/linhvu/micrometer/MeterRegistry.java`

```java
package dev.linhvu.micrometer;

import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.config.MeterFilterReply;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.noop.NoopCounter;
import dev.linhvu.micrometer.noop.NoopDistributionSummary;
import dev.linhvu.micrometer.noop.NoopGauge;
import dev.linhvu.micrometer.noop.NoopTimer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;
import java.util.function.ToDoubleFunction;

/**
 * Central registry that manages meter lifecycle — creation, deduplication, and lookup.
 * <p>
 * Uses the <b>Template Method</b> pattern: this class defines the complete registration
 * lifecycle (ID construction, deduplication via ConcurrentHashMap, factory dispatch),
 * while subclasses implement factory methods like {@code newCounter()} and {@code newGauge()}.
 * <p>
 * The deduplication algorithm uses <b>double-check locking</b>:
 * <ol>
 *     <li>Lock-free fast path: read from ConcurrentHashMap</li>
 *     <li>Synchronized slow path: re-check, create, and store the new meter</li>
 * </ol>
 * This ensures that {@code Counter.builder("x").register(registry)} called from multiple
 * threads always returns the same Counter instance.
 * <p>
 * If a meter is already registered with a given name+tags but as a different type
 * (e.g., trying to register a Timer when a Counter already exists), an
 * {@link IllegalArgumentException} is thrown.
 */
public abstract class MeterRegistry {

    protected final Clock clock;

    private final ConcurrentHashMap<Meter.Id, Meter> meterMap = new ConcurrentHashMap<>();

    private final Object meterMapLock = new Object();

    /**
     * Volatile array of filters — enables lock-free iteration during meter registration
     * while still allowing thread-safe mutation via {@link #meterFilter(MeterFilter)}.
     * Copy-on-write: adding a filter creates a new array.
     */
    private volatile MeterFilter[] filters = new MeterFilter[0];

    private volatile NamingConvention namingConvention = NamingConvention.identity;

    private volatile boolean closed = false;

    protected MeterRegistry(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock must not be null");
    }

    // -----------------------------------------------------------------------
    // NamingConvention configuration
    // -----------------------------------------------------------------------

    /**
     * Returns the naming convention used by this registry when exporting metric
     * names and tag keys/values.
     */
    public NamingConvention getNamingConvention() {
        return namingConvention;
    }

    /**
     * Sets the naming convention for this registry. Subclasses (backend registries)
     * typically set a default in their constructor (e.g., snakeCase for Prometheus).
     *
     * @param namingConvention The convention to use.
     * @return This registry, for chaining.
     */
    public MeterRegistry namingConvention(NamingConvention namingConvention) {
        this.namingConvention = Objects.requireNonNull(namingConvention, "namingConvention must not be null");
        return this;
    }

    // -----------------------------------------------------------------------
    // Filter configuration
    // -----------------------------------------------------------------------

    /**
     * Adds a meter filter to this registry. Filters are applied in the order they
     * are added, forming a chain:
     * <ol>
     *   <li>{@link MeterFilter#map(Meter.Id)} — transforms the ID (all filters)</li>
     *   <li>{@link MeterFilter#accept(Meter.Id)} — accept/deny decision (short-circuits)</li>
     * </ol>
     * <p>
     * Filters should be configured before any meters are registered. Filters added
     * after meters are registered will only affect new registrations.
     *
     * @param filter The filter to add.
     * @return This registry, for chaining.
     */
    public synchronized MeterRegistry meterFilter(MeterFilter filter) {
        Objects.requireNonNull(filter, "filter must not be null");
        MeterFilter[] newFilters = Arrays.copyOf(filters, filters.length + 1);
        newFilters[filters.length] = filter;
        filters = newFilters;
        return this;
    }

    // -----------------------------------------------------------------------
    // Public convenience registration methods
    // -----------------------------------------------------------------------

    /**
     * Creates or retrieves a counter with the given name and tags.
     * @param name The counter name.
     * @param tags Must be an even number of arguments representing key/value pairs.
     * @return A new or existing Counter.
     */
    public Counter counter(String name, String... tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a counter with the given name and tags.
     * @param name The counter name.
     * @param tags The tag collection.
     * @return A new or existing Counter.
     */
    public Counter counter(String name, Iterable<Tag> tags) {
        return Counter.builder(name).tags(tags).register(this);
    }

    /**
     * Register a gauge that reports the value of the {@code obj} as determined by the
     * function {@code valueFunction}. The state object is held via a WeakReference.
     *
     * @param name          The gauge name.
     * @param tags          Tags for the gauge.
     * @param stateObject   The state object to observe.
     * @param valueFunction Function that extracts a double value from the state object.
     * @param <T>           The type of the state object.
     * @return The state object, for inline assignment.
     */
    public <T> T gauge(String name, Iterable<Tag> tags, T stateObject, ToDoubleFunction<T> valueFunction) {
        Gauge.builder(name, stateObject, valueFunction).tags(tags).register(this);
        return stateObject;
    }

    /**
     * Register a gauge that reports the value of the {@link Number}.
     *
     * @param name   The gauge name.
     * @param tags   Tags for the gauge.
     * @param number The Number whose {@code doubleValue()} to report.
     * @param <T>    The type of the number.
     * @return The number, for inline assignment.
     */
    public <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
        return gauge(name, tags, number, Number::doubleValue);
    }

    /**
     * Register a gauge that reports the value of the {@link Number}.
     *
     * @param name   The gauge name.
     * @param number The Number whose {@code doubleValue()} to report.
     * @param <T>    The type of the number.
     * @return The number, for inline assignment.
     */
    public <T extends Number> T gauge(String name, T number) {
        return gauge(name, Collections.emptyList(), number);
    }

    /**
     * Register a gauge that reports the size of the {@link Collection}.
     *
     * @param name       The gauge name.
     * @param tags       Tags for the gauge.
     * @param collection The collection whose size to report.
     * @param <T>        The type of the collection.
     * @return The collection, for inline assignment.
     */
    public <T extends Collection<?>> T gaugeCollectionSize(String name, Iterable<Tag> tags, T collection) {
        return gauge(name, tags, collection, Collection::size);
    }

    /**
     * Register a gauge that reports the size of the {@link Map}.
     *
     * @param name The gauge name.
     * @param tags Tags for the gauge.
     * @param map  The map whose size to report.
     * @param <T>  The type of the map.
     * @return The map, for inline assignment.
     */
    public <T extends Map<?, ?>> T gaugeMapSize(String name, Iterable<Tag> tags, T map) {
        return gauge(name, tags, map, Map::size);
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     * @param name The timer name.
     * @param tags Must be an even number of arguments representing key/value pairs.
     * @return A new or existing Timer.
     */
    public Timer timer(String name, String... tags) {
        return Timer.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a timer with the given name and tags.
     * @param name The timer name.
     * @param tags The tag collection.
     * @return A new or existing Timer.
     */
    public Timer timer(String name, Iterable<Tag> tags) {
        return Timer.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     * @param name The summary name.
     * @param tags Must be an even number of arguments representing key/value pairs.
     * @return A new or existing DistributionSummary.
     */
    public DistributionSummary summary(String name, String... tags) {
        return DistributionSummary.builder(name).tags(tags).register(this);
    }

    /**
     * Creates or retrieves a distribution summary with the given name and tags.
     * @param name The summary name.
     * @param tags The tag collection.
     * @return A new or existing DistributionSummary.
     */
    public DistributionSummary summary(String name, Iterable<Tag> tags) {
        return DistributionSummary.builder(name).tags(tags).register(this);
    }

    // -----------------------------------------------------------------------
    // Package-private registration (called by Builder.register())
    // -----------------------------------------------------------------------

    /**
     * Registers a counter using the pre-built ID. Called by {@link Counter.Builder#register(MeterRegistry)}.
     */
    Counter counter(Meter.Id id) {
        return registerMeterIfNecessary(Counter.class, id, this::newCounter, NoopCounter::new);
    }

    /**
     * Registers a gauge using the pre-built ID. Called by {@link Gauge.Builder#register(MeterRegistry)}.
     * The gauge factory is a lambda because {@code newGauge()} requires additional parameters
     * (the state object and value function) that standard method references cannot capture.
     */
    <T> Gauge gauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return registerMeterIfNecessary(Gauge.class, id,
                mappedId -> newGauge(mappedId, obj, valueFunction), NoopGauge::new);
    }

    /**
     * Registers a timer using the pre-built ID. Called by {@link Timer.Builder#register(MeterRegistry)}.
     */
    Timer timer(Meter.Id id) {
        return registerMeterIfNecessary(Timer.class, id, this::newTimer, NoopTimer::new);
    }

    /**
     * Registers a distribution summary using the pre-built ID and scale.
     * Called by {@link DistributionSummary.Builder#register(MeterRegistry)}.
     */
    DistributionSummary summary(Meter.Id id, double scale) {
        return registerMeterIfNecessary(DistributionSummary.class, id,
                mappedId -> newDistributionSummary(mappedId, scale), NoopDistributionSummary::new);
    }

    // -----------------------------------------------------------------------
    // Template Method factories — subclasses implement these
    // -----------------------------------------------------------------------

    /**
     * Create a new counter implementation for this registry.
     * Called only when no counter with this ID exists yet.
     */
    protected abstract Counter newCounter(Meter.Id id);

    /**
     * Create a new gauge implementation for this registry.
     * Will be functional after Feature 5 (Gauge) is implemented.
     */
    protected abstract <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction);

    /**
     * Create a new timer implementation for this registry.
     * Will be functional after Feature 6 (Timer) is implemented.
     */
    protected abstract Timer newTimer(Meter.Id id);

    /**
     * Create a new distribution summary implementation for this registry.
     * @param id    The meter identity.
     * @param scale Factor to scale each recorded value by.
     */
    protected abstract DistributionSummary newDistributionSummary(Meter.Id id, double scale);

    /**
     * Returns the base time unit for this registry. Backends that work in
     * seconds (e.g., Prometheus) return {@link TimeUnit#SECONDS}; backends
     * that work in milliseconds return {@link TimeUnit#MILLISECONDS}.
     */
    protected abstract TimeUnit getBaseTimeUnit();

    // -----------------------------------------------------------------------
    // Registration lifecycle — the heart of deduplication
    // -----------------------------------------------------------------------

    /**
     * Registers a meter if one with the same ID does not already exist. If a meter
     * with the same name+tags already exists but is a different type, throws
     * {@link IllegalArgumentException}.
     * <p>
     * The filter chain runs here: the ID is mapped (transformed), then the accept
     * decision is checked. If denied, a no-op meter is returned immediately without
     * being stored in the registry.
     *
     * @param meterClass    The expected meter type (e.g., Counter.class)
     * @param id            The meter's identity (name + tags)
     * @param meterFactory  Factory to create the real meter (calls a template method)
     * @param noopFactory   Factory to create a no-op meter (for closed/denied meters)
     * @return The existing or newly created meter, cast to M
     */
    private <M extends Meter> M registerMeterIfNecessary(Class<M> meterClass, Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        // Stage 1: MAP — transform the ID through the filter chain
        Meter.Id mappedId = mapId(id);

        // Stage 2: ACCEPT — check if the meter should be registered
        if (!accept(mappedId)) {
            return meterClass.cast(noopFactory.apply(mappedId));
        }

        Meter meter = getOrCreateMeter(mappedId, meterFactory, noopFactory);

        if (!meterClass.isInstance(meter)) {
            throw new IllegalArgumentException(
                    "There is already a meter registered with name '" + mappedId.getName()
                            + "' and tags " + mappedId.getTagsAsIterable()
                            + " but with type " + meter.getClass().getSimpleName()
                            + ". Attempted to register type " + meterClass.getSimpleName() + ".");
        }

        return meterClass.cast(meter);
    }

    /**
     * Stage 1 of the filter pipeline: transforms the meter ID through all filters.
     * Each filter's output becomes the next filter's input.
     */
    private Meter.Id mapId(Meter.Id id) {
        Meter.Id mappedId = id;
        for (MeterFilter filter : filters) {
            mappedId = filter.map(mappedId);
        }
        return mappedId;
    }

    /**
     * Stage 2 of the filter pipeline: determines whether to create a real meter
     * or a no-op. Uses short-circuit evaluation — the first non-NEUTRAL reply wins.
     * If all filters return NEUTRAL, the meter is accepted.
     */
    private boolean accept(Meter.Id id) {
        for (MeterFilter filter : filters) {
            MeterFilterReply reply = filter.accept(id);
            if (reply == MeterFilterReply.DENY) {
                return false;
            }
            if (reply == MeterFilterReply.ACCEPT) {
                return true;
            }
        }
        // All filters returned NEUTRAL → accept by default
        return true;
    }

    /**
     * The core deduplication algorithm. Uses double-check locking:
     * <ol>
     *     <li>Lock-free read from ConcurrentHashMap (fast path — no contention)</li>
     *     <li>Synchronized creation block with re-check (slow path — only on first registration)</li>
     * </ol>
     *
     * When the registry is closed, returns a no-op meter instead of creating a real one.
     * This ensures that late registrations after shutdown are silently absorbed rather
     * than throwing exceptions.
     */
    private Meter getOrCreateMeter(Meter.Id id,
            Function<Meter.Id, ? extends Meter> meterFactory,
            Function<Meter.Id, ? extends Meter> noopFactory) {

        // Fast path: ConcurrentHashMap.get() is lock-free
        Meter existing = meterMap.get(id);
        if (existing != null) {
            return existing;
        }

        // Slow path: synchronized creation to prevent duplicate meters
        synchronized (meterMapLock) {
            // Double-check: another thread may have created it while we waited
            existing = meterMap.get(id);
            if (existing != null) {
                return existing;
            }

            // Registry is closed → return a no-op meter
            if (closed) {
                return noopFactory.apply(id);
            }

            // Create the meter via the template method
            Meter newMeter = meterFactory.apply(id);
            meterMap.put(id, newMeter);
            return newMeter;
        }
    }

    // -----------------------------------------------------------------------
    // Query methods
    // -----------------------------------------------------------------------

    /**
     * Returns a snapshot of all registered meters. The returned list is a copy;
     * modifications to it do not affect the registry.
     */
    public List<Meter> getMeters() {
        return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
    }

    /**
     * Returns the clock used by this registry for time measurements.
     */
    public Clock getClock() {
        return clock;
    }

    // -----------------------------------------------------------------------
    // Lifecycle
    // -----------------------------------------------------------------------

    /**
     * Returns true if this registry has been closed. A closed registry returns
     * no-op meters for all new registrations.
     */
    public boolean isClosed() {
        return closed;
    }

    /**
     * Closes this registry. After this call, any new meter registration will
     * return a no-op meter. Existing meters remain functional.
     */
    public void close() {
        closed = true;
    }

}
```

### Test Code

#### [NEW] `src/test/java/dev/linhvu/micrometer/config/NamingConventionTest.java`

```java
package dev.linhvu.micrometer.config;

import dev.linhvu.micrometer.Meter;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class NamingConventionTest {

    // --- Identity / Dot ---

    @Test
    void shouldReturnNameUnchanged_WhenUsingIdentityConvention() {
        assertThat(NamingConvention.identity.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("http.server.requests");
    }

    @Test
    void shouldReturnTagKeyUnchanged_WhenUsingIdentityConvention() {
        assertThat(NamingConvention.identity.tagKey("response.status"))
                .isEqualTo("response.status");
    }

    @Test
    void shouldReturnTagValueUnchanged_WhenUsingIdentityConvention() {
        assertThat(NamingConvention.identity.tagValue("some.value"))
                .isEqualTo("some.value");
    }

    @Test
    void shouldBeSameAsIdentity_WhenUsingDotConvention() {
        assertThat(NamingConvention.dot).isSameAs(NamingConvention.identity);
    }

    // --- Snake Case ---

    @Test
    void shouldConvertDotsToUnderscores_WhenUsingSnakeCaseConventionForName() {
        assertThat(NamingConvention.snakeCase.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("http_server_requests");
    }

    @Test
    void shouldConvertDotsToUnderscores_WhenUsingSnakeCaseConventionForTagKey() {
        assertThat(NamingConvention.snakeCase.tagKey("response.status"))
                .isEqualTo("response_status");
    }

    @Test
    void shouldReturnTagValueUnchanged_WhenUsingSnakeCaseConvention() {
        assertThat(NamingConvention.snakeCase.tagValue("some.value"))
                .isEqualTo("some.value");
    }

    @Test
    void shouldHandleSingleSegment_WhenUsingSnakeCaseConvention() {
        assertThat(NamingConvention.snakeCase.name("requests", Meter.Type.COUNTER))
                .isEqualTo("requests");
    }

    // --- Camel Case ---

    @Test
    void shouldConvertToCamelCase_WhenUsingCamelCaseConventionForName() {
        assertThat(NamingConvention.camelCase.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("httpServerRequests");
    }

    @Test
    void shouldConvertToCamelCase_WhenUsingCamelCaseConventionForTagKey() {
        assertThat(NamingConvention.camelCase.tagKey("response.status"))
                .isEqualTo("responseStatus");
    }

    @Test
    void shouldNotDoubleCapitalize_WhenSegmentAlreadyStartsWithUpperCase() {
        assertThat(NamingConvention.camelCase.name("http.Server.Requests", Meter.Type.TIMER))
                .isEqualTo("httpServerRequests");
    }

    @Test
    void shouldHandleSingleSegment_WhenUsingCamelCaseConvention() {
        assertThat(NamingConvention.camelCase.name("requests", Meter.Type.COUNTER))
                .isEqualTo("requests");
    }

    // --- Upper Camel Case ---

    @Test
    void shouldConvertToUpperCamelCase_WhenUsingUpperCamelCaseConventionForName() {
        assertThat(NamingConvention.upperCamelCase.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("HttpServerRequests");
    }

    @Test
    void shouldConvertToUpperCamelCase_WhenUsingUpperCamelCaseConventionForTagKey() {
        assertThat(NamingConvention.upperCamelCase.tagKey("response.status"))
                .isEqualTo("ResponseStatus");
    }

    @Test
    void shouldNotDoubleCapitalize_WhenAlreadyUpperCamelCase() {
        assertThat(NamingConvention.upperCamelCase.name("Http.server.requests", Meter.Type.TIMER))
                .isEqualTo("HttpServerRequests");
    }

    // --- Slashes ---

    @Test
    void shouldConvertDotsToSlashes_WhenUsingSlashesConventionForName() {
        assertThat(NamingConvention.slashes.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("http/server/requests");
    }

    @Test
    void shouldConvertDotsToSlashes_WhenUsingSlashesConventionForTagKey() {
        assertThat(NamingConvention.slashes.tagKey("response.status"))
                .isEqualTo("response/status");
    }

    // --- Functional interface / Lambda ---

    @Test
    void shouldSupportLambdaDefinition_WhenUsedAsFunctionalInterface() {
        NamingConvention upper = (name, type, baseUnit) -> name.toUpperCase();

        assertThat(upper.name("http.server.requests", Meter.Type.TIMER))
                .isEqualTo("HTTP.SERVER.REQUESTS");
        assertThat(upper.tagKey("response.status")).isEqualTo("response.status");
        assertThat(upper.tagValue("200")).isEqualTo("200");
    }

    // --- Convenience two-arg overload ---

    @Test
    void shouldDelegateToThreeArgMethod_WhenCallingTwoArgName() {
        NamingConvention convention = (name, type, baseUnit) ->
                name + (baseUnit != null ? "." + baseUnit : "");

        assertThat(convention.name("request.size", Meter.Type.DISTRIBUTION_SUMMARY, "bytes"))
                .isEqualTo("request.size.bytes");
        assertThat(convention.name("request.size", Meter.Type.DISTRIBUTION_SUMMARY))
                .isEqualTo("request.size");
    }

    // --- Type and baseUnit parameters ---

    @Test
    void shouldPassTypeAndBaseUnit_WhenConventionUsesThemForSuffixing() {
        NamingConvention prometheuLike = (name, type, baseUnit) -> {
            String snaked = name.replace('.', '_');
            if (type == Meter.Type.COUNTER) {
                return snaked + "_total";
            }
            return snaked;
        };

        assertThat(prometheuLike.name("http.requests", Meter.Type.COUNTER))
                .isEqualTo("http_requests_total");
        assertThat(prometheuLike.name("http.requests", Meter.Type.TIMER))
                .isEqualTo("http_requests");
    }

}
```

#### [NEW] `src/test/java/dev/linhvu/micrometer/integration/NamingConventionIntegrationTest.java`

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class NamingConventionIntegrationTest {

    // --- Meter.Id convention methods ---

    @Test
    void shouldTransformNameViaConvention_WhenCallingGetConventionName() {
        Meter.Id id = new Meter.Id("http.server.requests",
                Tags.of("method", "GET"), Meter.Type.TIMER, null, "seconds");

        assertThat(id.getConventionName(NamingConvention.snakeCase))
                .isEqualTo("http_server_requests");
        assertThat(id.getConventionName(NamingConvention.camelCase))
                .isEqualTo("httpServerRequests");
        assertThat(id.getConventionName(NamingConvention.identity))
                .isEqualTo("http.server.requests");
    }

    @Test
    void shouldTransformTagKeysViaConvention_WhenCallingGetConventionTags() {
        Meter.Id id = new Meter.Id("http.server.requests",
                Tags.of("response.status", "200", "http.method", "GET"),
                Meter.Type.TIMER, null, null);

        List<Tag> snakeTags = id.getConventionTags(NamingConvention.snakeCase);

        assertThat(snakeTags).hasSize(2);
        assertThat(snakeTags.get(0).getKey()).isEqualTo("http_method");
        assertThat(snakeTags.get(0).getValue()).isEqualTo("GET");
        assertThat(snakeTags.get(1).getKey()).isEqualTo("response_status");
        assertThat(snakeTags.get(1).getValue()).isEqualTo("200");
    }

    @Test
    void shouldPreserveTagValues_WhenConventionOnlyTransformsKeys() {
        Meter.Id id = new Meter.Id("test",
                Tags.of("env.name", "prod.us-east"), Meter.Type.GAUGE, null, null);

        List<Tag> tags = id.getConventionTags(NamingConvention.snakeCase);

        assertThat(tags.get(0).getKey()).isEqualTo("env_name");
        assertThat(tags.get(0).getValue()).isEqualTo("prod.us-east");
    }

    @Test
    void shouldPassTypeAndBaseUnit_WhenCallingGetConventionName() {
        Meter.Id id = new Meter.Id("http.requests",
                Tags.empty(), Meter.Type.COUNTER, null, "bytes");

        NamingConvention custom = (name, type, baseUnit) -> {
            String result = name.replace('.', '_');
            if (type == Meter.Type.COUNTER) result += "_total";
            if (baseUnit != null) result += "_" + baseUnit;
            return result;
        };

        assertThat(id.getConventionName(custom)).isEqualTo("http_requests_total_bytes");
    }

    // --- Same meter, multiple conventions ---

    @Test
    void shouldApplyDifferentConventions_WhenSameMeterExportedMultipleTimes() {
        Meter.Id id = new Meter.Id("http.server.requests",
                Tags.of("response.code", "200"), Meter.Type.TIMER, null, null);

        assertThat(id.getConventionName(NamingConvention.snakeCase))
                .isEqualTo("http_server_requests");
        assertThat(id.getConventionTags(NamingConvention.snakeCase).get(0).getKey())
                .isEqualTo("response_code");

        assertThat(id.getConventionName(NamingConvention.camelCase))
                .isEqualTo("httpServerRequests");
        assertThat(id.getConventionTags(NamingConvention.camelCase).get(0).getKey())
                .isEqualTo("responseCode");

        assertThat(id.getName()).isEqualTo("http.server.requests");
    }

    // --- MeterRegistry holds a convention ---

    @Test
    void shouldDefaultToIdentity_WhenNoConventionConfigured() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.identity);
    }

    @Test
    void shouldStoreConfiguredConvention_WhenSetOnRegistry() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.namingConvention(NamingConvention.snakeCase);
        assertThat(registry.getNamingConvention()).isSameAs(NamingConvention.snakeCase);
    }

    @Test
    void shouldApplyRegistryConvention_WhenExportingRegisteredMeter() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        registry.namingConvention(NamingConvention.snakeCase);

        Counter counter = registry.counter("http.server.requests", "response.code", "200");

        assertThat(counter.getId().getName()).isEqualTo("http.server.requests");

        NamingConvention convention = registry.getNamingConvention();
        assertThat(counter.getId().getConventionName(convention))
                .isEqualTo("http_server_requests");
        assertThat(counter.getId().getConventionTags(convention).get(0).getKey())
                .isEqualTo("response_code");
    }

    @Test
    void shouldSupportChainingConventionSetter_WhenConfiguringRegistry() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        MeterRegistry result = registry.namingConvention(NamingConvention.camelCase);
        assertThat(result).isSameAs(registry);
    }

    @Test
    void shouldAllowConventionChange_WhenRegistryAlreadyHasMeters() {
        SimpleMeterRegistry registry = new SimpleMeterRegistry();
        Timer timer = registry.timer("http.server.requests", "method", "GET");

        registry.namingConvention(NamingConvention.upperCamelCase);

        assertThat(timer.getId().getConventionName(registry.getNamingConvention()))
                .isEqualTo("HttpServerRequests");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **NamingConvention** | A functional interface that translates metric names and tag keys for monitoring backends |
| **identity / dot** | The canonical form — dot-separated names pass through unchanged; all other conventions convert *from* dots |
| **snakeCase** | Splits on dots, joins with underscores — used by Prometheus, InfluxDB |
| **camelCase** | Splits on dots, capitalizes subsequent segments — used by Atlas, Elastic |
| **upperCamelCase** | Delegates to camelCase, then capitalizes the first character |
| **slashes** | Splits on dots, joins with slashes — used by Graphite, StatsD |
| **getConventionName()** | On `Meter.Id` — delegates to the convention, passing name, type, and baseUnit |
| **getConventionTags()** | On `Meter.Id` — maps each tag through tagKey() and tagValue() |
| **Export-time application** | The convention is NOT baked into the Id — it's applied on-the-fly when exporting, so the same meter can be exported to multiple backends |

**Next: Chapter 10 — Distribution Statistics** — Histogram and percentile computation that enhances Timer and DistributionSummary with rich distribution data. You'll build `DistributionStatisticConfig` with cascading configuration, `TimeWindowFixedBoundaryHistogram` using a ring buffer of bucket arrays, and `HistogramSnapshot` for capturing count/total/max/percentiles.
