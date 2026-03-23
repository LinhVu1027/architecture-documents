# Chapter 17: Profiles & ConfigData

## Build Challenge

| | |
|---|---|
| **Current State** | `IrisApplication` loads a single `application.properties` from the classpath root via `PropertiesPropertySourceLoader.load()` |
| **Limitation** | No profile-aware configuration — you can't have `application-dev.properties` override `application-prod.properties`. No multi-location search. No `spring.config.import`. |
| **Objective** | Replace the simple property loader with a full config data pipeline: search multiple locations, determine active profiles, load profile-specific files, and process `spring.config.import` directives |

---

## 17.1 The Integration Point

The new feature plugs into **one exact method**: `IrisApplication.prepareEnvironment()`.

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java`
**Change:** Replace the simple `PropertiesPropertySourceLoader.load()` call with the ConfigData pipeline

Before (Feature 7):
```java
private Environment prepareEnvironment(IrisApplicationRunListeners listeners) {
    StandardEnvironment environment = new StandardEnvironment();

    // Simple: load one file from classpath root
    PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
    MapPropertySource appProps = loader.load(
            APPLICATION_PROPERTIES_SOURCE_NAME, APPLICATION_PROPERTIES_RESOURCE);
    if (appProps != null) {
        environment.getPropertySources().addLast(appProps);
    }

    listeners.environmentPrepared(environment);
    return environment;
}
```

After (Feature 17):
```java
private Environment prepareEnvironment(IrisApplicationRunListeners listeners) {
    StandardEnvironment environment = new StandardEnvironment();

    // Full pipeline: search locations → load base → determine profiles →
    // load profile-specific → process imports → apply
    ConfigDataEnvironmentPostProcessor.applyTo(environment);

    listeners.environmentPrepared(environment);
    return environment;
}
```

**Direction:** The `ConfigDataEnvironmentPostProcessor` is the entry point. It creates a `ConfigDataEnvironment` orchestrator, which uses two SPIs — `ConfigDataLocationResolver` (turns locations into resources) and `ConfigDataLoader` (turns resources into property sources) — to execute a three-phase algorithm. Everything we build supports this single integration point.

The `Environment` interface also gains profile methods (`getActiveProfiles()`, `setActiveProfiles()`, `acceptsProfiles()`), which the config data pipeline sets after determining profiles.

---

## 17.2 The Domain Model

Before building the pipeline, we need three value types to represent the data flowing through it.

### ConfigDataLocation — what the user specifies

A location string like `"classpath:/"` or `"optional:file:./config/"`. May be prefixed with `optional:` to silently skip missing resources.

```java
package com.iris.boot.context.config;

public class ConfigDataLocation {

    private static final String OPTIONAL_PREFIX = "optional:";

    private final String value;
    private final boolean optional;

    private ConfigDataLocation(String value, boolean optional) {
        this.value = value;
        this.optional = optional;
    }

    public static ConfigDataLocation of(String location) {
        if (location.startsWith(OPTIONAL_PREFIX)) {
            return new ConfigDataLocation(
                    location.substring(OPTIONAL_PREFIX.length()), true);
        }
        return new ConfigDataLocation(location, false);
    }

    public String getValue() { return this.value; }
    public boolean isOptional() { return this.optional; }
}
```

### ConfigDataResource — what the resolver produces

A concrete file reference like `"classpath:config/application-dev.properties"`.

```java
package com.iris.boot.context.config;

public class ConfigDataResource {

    private final String location;
    private final boolean optional;

    public ConfigDataResource(String location, boolean optional) {
        this.location = location;
        this.optional = optional;
    }

    public String getLocation() { return this.location; }
    public boolean isOptional() { return this.optional; }

    public boolean isClasspathResource() {
        return this.location.startsWith("classpath:");
    }

    public boolean isFileResource() {
        return this.location.startsWith("file:");
    }

    public String getResourcePath() {
        if (isClasspathResource()) return this.location.substring("classpath:".length());
        if (isFileResource()) return this.location.substring("file:".length());
        return this.location;
    }
}
```

### ConfigData — what the loader produces

Wraps the `MapPropertySource` instances loaded from a resource.

```java
package com.iris.boot.context.config;

import java.util.List;
import com.iris.framework.core.env.MapPropertySource;

public class ConfigData {

    public static final ConfigData EMPTY = new ConfigData(List.of());

    private final List<MapPropertySource> propertySources;

    public ConfigData(List<MapPropertySource> propertySources) {
        this.propertySources = List.copyOf(propertySources);
    }

    public ConfigData(MapPropertySource propertySource) {
        this(List.of(propertySource));
    }

    public List<MapPropertySource> getPropertySources() {
        return this.propertySources;
    }
}
```

---

## 17.3 The SPI Interfaces

The pipeline uses the **Strategy pattern** — two interfaces separate "where to look" from "how to load":

### ConfigDataLocationResolver — turns locations into resources

```java
package com.iris.boot.context.config;

import java.util.List;

public interface ConfigDataLocationResolver {

    boolean isResolvable(ConfigDataLocation location);

    List<ConfigDataResource> resolve(ConfigDataLocation location);

    List<ConfigDataResource> resolveProfileSpecific(
            ConfigDataLocation location, String[] profiles);
}
```

### ConfigDataLoader — turns resources into property sources

```java
package com.iris.boot.context.config;

public interface ConfigDataLoader {

    boolean isLoadable(ConfigDataResource resource);

    ConfigData load(ConfigDataResource resource);
}
```

The key design insight: by separating resolution from loading, external config sources (Consul, AWS Secrets Manager, Spring Cloud Config Server) only need to implement these two interfaces. The orchestrator doesn't care where properties come from.

---

## 17.4 The Implementations

### StandardConfigDataLocationResolver

Handles `classpath:` and `file:` directory/file locations. For directories, expands them using the config name `"application"` and `.properties` extension:

```java
package com.iris.boot.context.config;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class StandardConfigDataLocationResolver implements ConfigDataLocationResolver {

    static final String DEFAULT_CONFIG_NAME = "application";
    private static final String PROPERTIES_EXTENSION = ".properties";

    @Override
    public boolean isResolvable(ConfigDataLocation location) {
        return true; // Catch-all resolver
    }

    @Override
    public List<ConfigDataResource> resolve(ConfigDataLocation location) {
        String value = location.getValue();
        boolean optional = location.isOptional();

        if (isDirectory(value)) {
            String resourcePath = buildResourcePath(value,
                    DEFAULT_CONFIG_NAME + PROPERTIES_EXTENSION);
            return List.of(new ConfigDataResource(resourcePath, optional));
        }

        // Explicit file — assume classpath if no scheme
        if (!value.startsWith("classpath:") && !value.startsWith("file:")) {
            return List.of(new ConfigDataResource("classpath:" + value, optional));
        }
        return List.of(new ConfigDataResource(value, optional));
    }

    @Override
    public List<ConfigDataResource> resolveProfileSpecific(
            ConfigDataLocation location, String[] profiles) {
        String value = location.getValue();
        boolean optional = location.isOptional();

        if (!isDirectory(value) || profiles.length == 0) {
            return List.of();
        }

        List<ConfigDataResource> resources = new ArrayList<>();
        for (String profile : profiles) {
            String resourcePath = buildResourcePath(value,
                    DEFAULT_CONFIG_NAME + "-" + profile + PROPERTIES_EXTENSION);
            resources.add(new ConfigDataResource(resourcePath, optional));
        }
        return resources;
    }

    private boolean isDirectory(String value) {
        return value.endsWith("/") || value.endsWith(File.separator);
    }

    private String buildResourcePath(String directoryLocation, String filename) {
        if (directoryLocation.startsWith("classpath:")) {
            String path = directoryLocation.substring("classpath:".length());
            if (path.startsWith("/")) path = path.substring(1);
            return "classpath:" + path + filename;
        }
        if (directoryLocation.startsWith("file:")) {
            String path = directoryLocation.substring("file:".length());
            return "file:" + path + filename;
        }
        return "classpath:" + directoryLocation + filename;
    }
}
```

### StandardConfigDataLoader

Loads `.properties` files from classpath or filesystem:

```java
package com.iris.boot.context.config;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Properties;
import com.iris.framework.core.env.MapPropertySource;

public class StandardConfigDataLoader implements ConfigDataLoader {

    @Override
    public boolean isLoadable(ConfigDataResource resource) {
        return resource.getLocation().endsWith(".properties");
    }

    @Override
    public ConfigData load(ConfigDataResource resource) {
        if (resource.isClasspathResource()) {
            return loadFromClasspath(resource.getResourcePath(), resource.getLocation());
        }
        if (resource.isFileResource()) {
            return loadFromFileSystem(resource.getResourcePath(), resource.getLocation());
        }
        throw new IllegalArgumentException("Unsupported resource type: " + resource.getLocation());
    }

    private ConfigData loadFromClasspath(String path, String sourceName) {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (classLoader == null) classLoader = getClass().getClassLoader();

        try (InputStream is = classLoader.getResourceAsStream(path)) {
            if (is == null) return null;
            return parseProperties(is, sourceName);
        } catch (IOException ex) {
            throw new RuntimeException("Failed to load config data from " + sourceName, ex);
        }
    }

    private ConfigData loadFromFileSystem(String path, String sourceName) {
        Path filePath = Path.of(path);
        if (!Files.exists(filePath)) return null;

        try (InputStream is = Files.newInputStream(filePath)) {
            return parseProperties(is, sourceName);
        } catch (IOException ex) {
            throw new RuntimeException("Failed to load config data from " + sourceName, ex);
        }
    }

    private ConfigData parseProperties(InputStream is, String sourceName) throws IOException {
        Properties properties = new Properties();
        properties.load(is);

        Map<String, Object> map = new LinkedHashMap<>();
        for (String key : properties.stringPropertyNames()) {
            map.put(key, properties.getProperty(key));
        }

        if (map.isEmpty()) return ConfigData.EMPTY;

        String name = "Config resource '" + sourceName + "'";
        return new ConfigData(new MapPropertySource(name, map));
    }
}
```

### ConfigDataEnvironment — the three-phase orchestrator

This is the heart of the feature. It implements a simplified version of Spring Boot's three-phase algorithm:

```java
package com.iris.boot.context.config;

import java.util.*;
import com.iris.framework.core.env.*;

public class ConfigDataEnvironment {

    static final String[] DEFAULT_SEARCH_LOCATIONS = {
            "optional:file:./config/",
            "optional:file:./",
            "optional:classpath:/config/",
            "optional:classpath:/"
    };

    private static final String ACTIVE_PROFILES_PROPERTY = "spring.profiles.active";
    private static final String CONFIG_IMPORT_PROPERTY = "spring.config.import";

    private final Environment environment;
    private final ConfigDataLocationResolver resolver;
    private final ConfigDataLoader loader;

    public ConfigDataEnvironment(Environment environment) {
        this(environment, new StandardConfigDataLocationResolver(),
                new StandardConfigDataLoader());
    }

    public void processAndApply() {
        ConfigDataLocation[] searchLocations = parseSearchLocations();

        // Phase 1: Load base config files
        List<MapPropertySource> baseSources = new ArrayList<>();
        for (ConfigDataLocation location : searchLocations) {
            for (ConfigDataResource resource : this.resolver.resolve(location)) {
                ConfigData data = loadSafely(resource);
                if (data != null) baseSources.addAll(data.getPropertySources());
            }
        }

        // Phase 2: Determine active profiles
        String[] activeProfiles = findActiveProfiles(baseSources);
        String[] acceptedProfiles = activeProfiles.length > 0
                ? activeProfiles : this.environment.getDefaultProfiles();

        // Phase 3: Load profile-specific files (reverse order → later profiles win)
        List<MapPropertySource> profileSources = new ArrayList<>();
        for (int i = acceptedProfiles.length - 1; i >= 0; i--) {
            String profile = acceptedProfiles[i];
            for (ConfigDataLocation location : searchLocations) {
                for (ConfigDataResource resource :
                        this.resolver.resolveProfileSpecific(location, new String[]{profile})) {
                    ConfigData data = loadSafely(resource);
                    if (data != null) profileSources.addAll(data.getPropertySources());
                }
            }
        }

        // Phase 4: Process spring.config.import
        List<MapPropertySource> importedSources = processImports(baseSources, profileSources);

        // Apply to environment
        applyToEnvironment(importedSources, profileSources, baseSources);

        if (activeProfiles.length > 0) {
            this.environment.setActiveProfiles(activeProfiles);
        }
    }
    // ... (helper methods)
}
```

### ConfigDataEnvironmentPostProcessor — the entry point

```java
package com.iris.boot.context.config;

import com.iris.framework.core.env.Environment;

public class ConfigDataEnvironmentPostProcessor {

    public static void applyTo(Environment environment) {
        new ConfigDataEnvironment(environment).processAndApply();
    }
}
```

---

## 17.5 Environment Profile Support

The `Environment` interface gains profile management, and `StandardEnvironment` implements it:

**Modifying:** `iris-framework/src/main/java/com/iris/framework/core/env/Environment.java`
**Change:** Add profile methods to the interface

```java
public interface Environment extends PropertyResolver {

    MutablePropertySources getPropertySources();

    // ── New profile methods (Feature 17) ──

    String[] getActiveProfiles();
    String[] getDefaultProfiles();
    void setActiveProfiles(String... profiles);
    void addActiveProfile(String profile);
    boolean acceptsProfiles(String... profiles);
}
```

**Modifying:** `iris-framework/src/main/java/com/iris/framework/core/env/StandardEnvironment.java`
**Change:** Add profile tracking fields and implementations

```java
public class StandardEnvironment implements Environment {
    // ... existing fields ...

    private final Set<String> activeProfiles = new LinkedHashSet<>();
    private final Set<String> defaultProfiles = new LinkedHashSet<>(Set.of("default"));

    @Override
    public String[] getActiveProfiles() {
        return this.activeProfiles.toArray(String[]::new);
    }

    @Override
    public String[] getDefaultProfiles() {
        return this.defaultProfiles.toArray(String[]::new);
    }

    @Override
    public void setActiveProfiles(String... profiles) {
        this.activeProfiles.clear();
        for (String profile : profiles) {
            String trimmed = profile.trim();
            if (!trimmed.isEmpty()) this.activeProfiles.add(trimmed);
        }
    }

    @Override
    public void addActiveProfile(String profile) {
        String trimmed = profile.trim();
        if (!trimmed.isEmpty()) this.activeProfiles.add(trimmed);
    }

    @Override
    public boolean acceptsProfiles(String... profiles) {
        Set<String> accepted = new LinkedHashSet<>(this.activeProfiles);
        if (accepted.isEmpty()) accepted.addAll(this.defaultProfiles);
        for (String profile : profiles) {
            if (accepted.contains(profile)) return true;
        }
        return false;
    }
}
```

---

## 17.6 Try It Yourself

Before looking at the tests, try these challenges:

<details><summary>Challenge 1: What happens with <code>spring.profiles.active=dev,prod</code>?</summary>

Both `application-dev.properties` and `application-prod.properties` are loaded. The **later** profile (`prod`) has higher precedence — its values override `dev`'s values. This is because the pipeline iterates profiles in reverse order when loading, putting later profiles first in the property source list.

</details>

<details><summary>Challenge 2: If <code>application.properties</code> has <code>server.port=8080</code> and <code>application-dev.properties</code> has <code>server.port=8081</code>, which wins when <code>dev</code> is active?</summary>

`8081` wins. Profile-specific property sources are placed **before** base property sources in the environment. The `PropertySourcesPropertyResolver` returns the first match it finds — profile-specific sources are checked first.

</details>

<details><summary>Challenge 3: Where exactly do config data sources sit in the property source precedence?</summary>

After system properties and system environment variables:

```
[systemProperties, systemEnvironment, imported₁, ..., profile₁, ..., base₁, ...]
 ↑ highest priority                                          lowest priority ↑
```

System properties always win. Config data sources are the lowest-priority group. Within config data: imports > profile-specific > base.

</details>

<details><summary>Challenge 4: Implement a custom <code>ConfigDataLocationResolver</code> for <code>env:</code> prefixed locations</summary>

```java
public class EnvironmentVariableConfigDataLocationResolver
        implements ConfigDataLocationResolver {

    @Override
    public boolean isResolvable(ConfigDataLocation location) {
        return location.getValue().startsWith("env:");
    }

    @Override
    public List<ConfigDataResource> resolve(ConfigDataLocation location) {
        // env:MY_CONFIG_JSON → resolve from environment variable
        String envVar = location.getValue().substring("env:".length());
        return List.of(new ConfigDataResource("env:" + envVar, location.isOptional()));
    }

    @Override
    public List<ConfigDataResource> resolveProfileSpecific(
            ConfigDataLocation location, String[] profiles) {
        return List.of(); // No profile-specific env vars
    }
}
```

This shows the power of the SPI — anyone can add new config source types.

</details>

---

## 17.7 Tests

### Unit Tests

**ConfigDataLocationTest** — Verifies location parsing and the `optional:` prefix:

```java
@Test
void shouldParseOptionalLocation() {
    ConfigDataLocation location = ConfigDataLocation.of("optional:classpath:/config/");

    assertThat(location.getValue()).isEqualTo("classpath:/config/");
    assertThat(location.isOptional()).isTrue();
}
```

**StandardConfigDataLocationResolverTest** — Verifies directory expansion and profile resolution:

```java
@Test
void shouldResolveClasspathRootDirectory() {
    ConfigDataLocation location = ConfigDataLocation.of("classpath:/");

    List<ConfigDataResource> resources = resolver.resolve(location);

    assertThat(resources).hasSize(1);
    assertThat(resources.get(0).getLocation())
            .isEqualTo("classpath:application.properties");
}

@Test
void shouldResolveProfileSpecificResources() {
    ConfigDataLocation location = ConfigDataLocation.of("classpath:/");

    List<ConfigDataResource> resources =
            resolver.resolveProfileSpecific(location, new String[]{"dev"});

    assertThat(resources).hasSize(1);
    assertThat(resources.get(0).getLocation())
            .isEqualTo("classpath:application-dev.properties");
}
```

**StandardConfigDataLoaderTest** — Verifies classpath and filesystem loading:

```java
@Test
void shouldLoadClasspathResource() {
    ConfigDataResource resource =
            new ConfigDataResource("classpath:application.properties", false);

    ConfigData data = loader.load(resource);

    assertThat(data).isNotNull();
    assertThat(data.getPropertySources()).hasSize(1);
    assertThat(data.getPropertySources().get(0).getProperty("app.name"))
            .isEqualTo("Iris Boot Test");
}

@Test
void shouldReturnNull_WhenClasspathResourceNotFound() {
    ConfigDataResource resource =
            new ConfigDataResource("classpath:nonexistent.properties", true);

    assertThat(loader.load(resource)).isNull();
}
```

**ConfigDataEnvironmentTest** — Verifies the full pipeline:

```java
@Test
void shouldLoadBaseConfigFromClasspath() {
    StandardEnvironment env = new StandardEnvironment();
    new ConfigDataEnvironment(env).processAndApply();

    assertThat(env.getProperty("app.name")).isEqualTo("Iris Boot Test");
}

@Test
void shouldOverrideBaseWithProfileProperties() {
    StandardEnvironment env = new StandardEnvironment();
    env.getPropertySources().addFirst(new MapPropertySource("test",
            Map.of("spring.profiles.active", "dev")));

    new ConfigDataEnvironment(env).processAndApply();

    assertThat(env.getProperty("app.name")).isEqualTo("Iris Boot Dev");
}

@Test
void shouldSupportMultipleProfiles() {
    StandardEnvironment env = new StandardEnvironment();
    env.getPropertySources().addFirst(new MapPropertySource("test",
            Map.of("spring.profiles.active", "dev,prod")));

    new ConfigDataEnvironment(env).processAndApply();

    assertThat(env.getActiveProfiles()).containsExactly("dev", "prod");
    // prod properties override dev (later profile = higher priority)
    assertThat(env.getProperty("app.name")).isEqualTo("Iris Boot Prod");
}

@Test
void shouldProcessConfigImport() {
    StandardEnvironment env = new StandardEnvironment();
    env.getPropertySources().addFirst(new MapPropertySource("test",
            Map.of("spring.config.import", "classpath:import-source.properties")));

    new ConfigDataEnvironment(env).processAndApply();

    assertThat(env.getProperty("imported.key")).isEqualTo("imported-value");
}
```

### Integration Test

**ProfilesAndConfigDataIntegrationTest** — Full bootstrap with profiles:

```java
@Test
void shouldLoadProfileProperties_WhenProfileActivatedViaSystemProperty() {
    try {
        System.setProperty("spring.profiles.active", "dev");
        context = createApp(PropertyHolderConfig.class).run();

        PropertyHolder holder = context.getBean(PropertyHolder.class);
        assertThat(holder.appName).isEqualTo("Iris Boot Dev");
        assertThat(holder.serverPort).isEqualTo(8081);

        assertThat(context.getEnvironment().getActiveProfiles())
                .containsExactly("dev");
    } finally {
        System.clearProperty("spring.profiles.active");
    }
}
```

Run all tests:
```bash
./gradlew test
```

---

## 17.8 Why This Works

`★ Insight ─────────────────────────────────────`
**The three-phase algorithm solves a bootstrapping problem.** You can't know which profiles are active until you've loaded some configuration files (because `spring.profiles.active` might be in `application.properties`). But you can't load profile-specific files until you know which profiles are active. The solution: load base files first (Phase 1), extract the profile information (Phase 2), then go back and load profile-specific files (Phase 3). This is why the real Spring Boot uses an immutable contributor tree — each phase creates a new tree, and the profile determination step sits cleanly between the two loading phases.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**The Strategy pattern (Resolver + Loader) enables extensibility without modifying the core.** Spring Cloud Config, HashiCorp Vault, AWS Secrets Manager, and Kubernetes ConfigMaps all integrate via custom `ConfigDataLocationResolver` and `ConfigDataLoader` implementations. The user writes `spring.config.import=vault://secret/myapp` in their properties file, and a Vault resolver/loader pair handles it — the `ConfigDataEnvironment` orchestrator doesn't change. This is the Open/Closed principle in action: the pipeline is open for extension but closed for modification.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Profile ordering matters: last wins.** When you specify `spring.profiles.active=dev,prod`, `prod` overrides `dev`. This isn't just a convention — it's enforced by the loading order. The pipeline loads profile-specific files in reverse profile order, so the last-specified profile's sources end up closest to the front of the property source list (highest precedence). This means you can compose environments: `base ← dev ← staging` where each layer overrides the previous one.
`─────────────────────────────────────────────────`

---

## 17.9 What We Enhanced

| Class/Interface | Enhancement | Why |
|---|---|---|
| `Environment` | Added `getActiveProfiles()`, `getDefaultProfiles()`, `setActiveProfiles()`, `addActiveProfile()`, `acceptsProfiles()` | Profile management is the foundation that the config data pipeline writes to and that future features (like `@Profile` annotations) read from |
| `StandardEnvironment` | Added `activeProfiles` and `defaultProfiles` tracking with `LinkedHashSet` | Preserves insertion order (important for multi-profile precedence) while preventing duplicates |
| `IrisApplication.prepareEnvironment()` | Replaced `PropertiesPropertySourceLoader.load()` with `ConfigDataEnvironmentPostProcessor.applyTo()` | Single-location loading → multi-location, profile-aware, import-supporting pipeline |

---

## 17.10 Connection to Real Framework

| Iris Class | Real Spring Boot Class | File (commit `5922311a95a`) |
|---|---|---|
| `ConfigDataLocation` | `ConfigDataLocation` | `spring-boot/src/main/java/org/springframework/boot/context/config/ConfigDataLocation.java` |
| `ConfigDataResource` | `ConfigDataResource` + `StandardConfigDataResource` | `spring-boot/src/main/java/org/springframework/boot/context/config/StandardConfigDataResource.java` |
| `ConfigData` | `ConfigData` | `spring-boot/src/main/java/org/springframework/boot/context/config/ConfigData.java` |
| `ConfigDataLocationResolver` | `ConfigDataLocationResolver` | `spring-boot/src/main/java/org/springframework/boot/context/config/ConfigDataLocationResolver.java` |
| `ConfigDataLoader` | `ConfigDataLoader` | `spring-boot/src/main/java/org/springframework/boot/context/config/ConfigDataLoader.java` |
| `StandardConfigDataLocationResolver` | `StandardConfigDataLocationResolver` | `spring-boot/src/main/java/org/springframework/boot/context/config/StandardConfigDataLocationResolver.java` |
| `StandardConfigDataLoader` | `StandardConfigDataLoader` | `spring-boot/src/main/java/org/springframework/boot/context/config/StandardConfigDataLoader.java` |
| `ConfigDataEnvironment` | `ConfigDataEnvironment` | `spring-boot/src/main/java/org/springframework/boot/context/config/ConfigDataEnvironment.java` |
| `ConfigDataEnvironmentPostProcessor` | `ConfigDataEnvironmentPostProcessor` | `spring-boot/src/main/java/org/springframework/boot/context/config/ConfigDataEnvironmentPostProcessor.java` |
| `Environment.getActiveProfiles()` | `Environment.getActiveProfiles()` | Spring Framework `spring-core/src/main/java/org/springframework/core/env/Environment.java` (commit `11ab0b4351`) |
| `StandardEnvironment` (profiles) | `AbstractEnvironment` | Spring Framework `spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java` |

**Key differences from the real implementation:**

| Real Spring Boot | Our Simplification |
|---|---|
| Immutable contributor tree (`ConfigDataEnvironmentContributor`) with functional tree mutations | Simple `ArrayList` of property sources |
| Three separate processing methods with `ImportPhase` enum | One `processAndApply()` with sequential phases |
| `ConfigDataActivationContext` carrying cloud platform + profiles | Direct profile array |
| `ConfigDataProperties` binding `spring.config.activate.on-profile` per document | Profile-specific files via naming convention only |
| `spring.factories` SPI loading for resolvers and loaders | Direct instantiation |
| YAML support via `YamlPropertySourceLoader` | `.properties` files only |
| Profile groups (`spring.profiles.group`) and `spring.profiles.include` | Only `spring.profiles.active` |
| Multi-document properties files (`#---` separator) | One document per file |
| `OriginTrackedMapPropertySource` with file:line tracking | Plain `MapPropertySource` |

---

## 17.11 Complete Code

### Production Code

#### `[MODIFIED]` `iris-framework/src/main/java/com/iris/framework/core/env/Environment.java`

```java
package com.iris.framework.core.env;

/**
 * Interface representing the environment in which the application is running.
 *
 * <p>Extends {@link PropertyResolver} to add access to the underlying
 * {@link MutablePropertySources} and profile management.
 *
 * <p>The real hierarchy is:
 * <pre>
 * PropertyResolver
 *   → Environment           (adds profiles)
 *     → ConfigurableEnvironment  (adds getPropertySources(), setActiveProfiles())
 * </pre>
 * We collapse to {@code PropertyResolver → Environment} and put
 * both property sources and profile management here.
 *
 * @see PropertyResolver
 * @see StandardEnvironment
 * @see org.springframework.core.env.Environment
 */
public interface Environment extends PropertyResolver {

    /**
     * Return the property sources for this environment.
     *
     * <p>In the real Spring Framework, this method lives on
     * {@code ConfigurableEnvironment}, not {@code Environment}.
     * We promote it here because we collapse the two interfaces.
     */
    MutablePropertySources getPropertySources();

    /**
     * Return the set of profiles explicitly activated for this environment.
     * If no profiles have been explicitly activated, an empty array is returned.
     *
     * <p>In the real Spring Framework, this is on {@code Environment}.
     *
     * @return the active profiles, never {@code null}
     */
    String[] getActiveProfiles();

    /**
     * Return the set of default profiles. Default profiles are used for
     * profile-specific property loading when no profiles have been explicitly
     * activated. Defaults to {@code ["default"]}.
     *
     * <p>In the real Spring Framework, this is on {@code Environment}.
     *
     * @return the default profiles, never {@code null}
     */
    String[] getDefaultProfiles();

    /**
     * Set the active profiles for this environment.
     *
     * <p>In the real Spring Framework, this method lives on
     * {@code ConfigurableEnvironment}.
     *
     * @param profiles the set of profile names to activate
     */
    void setActiveProfiles(String... profiles);

    /**
     * Add a single active profile to the current set of active profiles.
     *
     * <p>In the real Spring Framework, this method lives on
     * {@code ConfigurableEnvironment}.
     *
     * @param profile the profile to add
     */
    void addActiveProfile(String profile);

    /**
     * Return whether one or more of the given profiles is active — or in
     * the case of no explicit active profiles, whether one or more of the
     * given profiles is in the set of default profiles.
     *
     * <p>In the real Spring Framework, this is
     * {@code Environment.acceptsProfiles(Profiles)} which supports
     * profile expressions ({@code "production & cloud"}). We simplify
     * to a plain string-based check.
     *
     * @param profiles the profiles to check
     * @return {@code true} if at least one of the given profiles is accepted
     */
    boolean acceptsProfiles(String... profiles);
}
```

#### `[MODIFIED]` `iris-framework/src/main/java/com/iris/framework/core/env/StandardEnvironment.java`

```java
package com.iris.framework.core.env;

import java.util.LinkedHashSet;
import java.util.Map;
import java.util.Set;

/**
 * Default {@link Environment} implementation suitable for non-web applications.
 *
 * <p>On construction, populates the {@link MutablePropertySources} with two
 * default property sources in precedence order:
 * <ol>
 *   <li>{@code "systemProperties"} — JVM system properties ({@code -Dproperty=value})</li>
 *   <li>{@code "systemEnvironment"} — OS environment variables</li>
 * </ol>
 *
 * <p>Also manages <strong>profiles</strong>: active profiles determine which
 * profile-specific configuration files (e.g., {@code application-dev.properties})
 * are loaded, and which {@code @Profile}-conditional beans are included. When
 * no active profiles are set, the default profiles ({@code ["default"]}) are used.
 *
 * <p>Delegates all {@link PropertyResolver} methods to a
 * {@link PropertySourcesPropertyResolver} that iterates the property sources.
 *
 * @see Environment
 * @see PropertySourcesPropertyResolver
 * @see org.springframework.core.env.StandardEnvironment
 */
public class StandardEnvironment implements Environment {

    public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
    public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";
    private static final String DEFAULT_PROFILE = "default";

    private final MutablePropertySources propertySources;
    private final PropertySourcesPropertyResolver propertyResolver;

    private final Set<String> activeProfiles = new LinkedHashSet<>();
    private final Set<String> defaultProfiles = new LinkedHashSet<>(Set.of(DEFAULT_PROFILE));

    public StandardEnvironment() {
        this.propertySources = new MutablePropertySources();
        customizePropertySources(this.propertySources);
        this.propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);
    }

    @SuppressWarnings({"unchecked", "rawtypes"})
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new MapPropertySource(
                SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME,
                (Map) System.getProperties()));
        propertySources.addLast(new MapPropertySource(
                SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                (Map) System.getenv()));
    }

    @Override
    public MutablePropertySources getPropertySources() {
        return this.propertySources;
    }

    @Override
    public String getProperty(String key) {
        return this.propertyResolver.getProperty(key);
    }

    @Override
    public String getProperty(String key, String defaultValue) {
        return this.propertyResolver.getProperty(key, defaultValue);
    }

    @Override
    public <T> T getProperty(String key, Class<T> targetType) {
        return this.propertyResolver.getProperty(key, targetType);
    }

    @Override
    public String getRequiredProperty(String key) {
        return this.propertyResolver.getRequiredProperty(key);
    }

    @Override
    public boolean containsProperty(String key) {
        return this.propertyResolver.containsProperty(key);
    }

    @Override
    public String resolvePlaceholders(String text) {
        return this.propertyResolver.resolvePlaceholders(text);
    }

    @Override
    public String resolveRequiredPlaceholders(String text) {
        return this.propertyResolver.resolveRequiredPlaceholders(text);
    }

    // ── Profiles ──

    @Override
    public String[] getActiveProfiles() {
        return this.activeProfiles.toArray(String[]::new);
    }

    @Override
    public String[] getDefaultProfiles() {
        return this.defaultProfiles.toArray(String[]::new);
    }

    @Override
    public void setActiveProfiles(String... profiles) {
        this.activeProfiles.clear();
        for (String profile : profiles) {
            String trimmed = profile.trim();
            if (!trimmed.isEmpty()) this.activeProfiles.add(trimmed);
        }
    }

    @Override
    public void addActiveProfile(String profile) {
        String trimmed = profile.trim();
        if (!trimmed.isEmpty()) this.activeProfiles.add(trimmed);
    }

    @Override
    public boolean acceptsProfiles(String... profiles) {
        Set<String> accepted = new LinkedHashSet<>(this.activeProfiles);
        if (accepted.isEmpty()) accepted.addAll(this.defaultProfiles);
        for (String profile : profiles) {
            if (accepted.contains(profile)) return true;
        }
        return false;
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + " {activeProfiles=" + this.activeProfiles
                + ", defaultProfiles=" + this.defaultProfiles
                + ", propertySources=" + this.propertySources + "}";
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigDataLocation.java`

```java
package com.iris.boot.context.config;

public class ConfigDataLocation {

    private static final String OPTIONAL_PREFIX = "optional:";

    private final String value;
    private final boolean optional;

    private ConfigDataLocation(String value, boolean optional) {
        this.value = value;
        this.optional = optional;
    }

    public static ConfigDataLocation of(String location) {
        if (location.startsWith(OPTIONAL_PREFIX)) {
            return new ConfigDataLocation(
                    location.substring(OPTIONAL_PREFIX.length()), true);
        }
        return new ConfigDataLocation(location, false);
    }

    public String getValue() { return this.value; }
    public boolean isOptional() { return this.optional; }

    @Override
    public String toString() {
        return (this.optional ? "optional:" : "") + this.value;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ConfigDataLocation that)) return false;
        return this.optional == that.optional && this.value.equals(that.value);
    }

    @Override
    public int hashCode() {
        return 31 * this.value.hashCode() + Boolean.hashCode(this.optional);
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigDataResource.java`

```java
package com.iris.boot.context.config;

public class ConfigDataResource {

    private final String location;
    private final boolean optional;

    public ConfigDataResource(String location, boolean optional) {
        this.location = location;
        this.optional = optional;
    }

    public String getLocation() { return this.location; }
    public boolean isOptional() { return this.optional; }

    public boolean isClasspathResource() {
        return this.location.startsWith("classpath:");
    }

    public boolean isFileResource() {
        return this.location.startsWith("file:");
    }

    public String getResourcePath() {
        if (isClasspathResource()) return this.location.substring("classpath:".length());
        if (isFileResource()) return this.location.substring("file:".length());
        return this.location;
    }

    @Override
    public String toString() { return this.location; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ConfigDataResource that)) return false;
        return this.location.equals(that.location);
    }

    @Override
    public int hashCode() { return this.location.hashCode(); }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigData.java`

```java
package com.iris.boot.context.config;

import java.util.List;
import com.iris.framework.core.env.MapPropertySource;

public class ConfigData {

    public static final ConfigData EMPTY = new ConfigData(List.of());

    private final List<MapPropertySource> propertySources;

    public ConfigData(List<MapPropertySource> propertySources) {
        this.propertySources = List.copyOf(propertySources);
    }

    public ConfigData(MapPropertySource propertySource) {
        this(List.of(propertySource));
    }

    public List<MapPropertySource> getPropertySources() {
        return this.propertySources;
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigDataLocationResolver.java`

```java
package com.iris.boot.context.config;

import java.util.List;

public interface ConfigDataLocationResolver {
    boolean isResolvable(ConfigDataLocation location);
    List<ConfigDataResource> resolve(ConfigDataLocation location);
    List<ConfigDataResource> resolveProfileSpecific(
            ConfigDataLocation location, String[] profiles);
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigDataLoader.java`

```java
package com.iris.boot.context.config;

public interface ConfigDataLoader {
    boolean isLoadable(ConfigDataResource resource);
    ConfigData load(ConfigDataResource resource);
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/StandardConfigDataLocationResolver.java`

*(Full source in section 17.4 above)*

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/StandardConfigDataLoader.java`

*(Full source in section 17.4 above)*

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigDataEnvironment.java`

*(Full source in section 17.4 above — see the actual source files for the complete implementation including all helper methods)*

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/context/config/ConfigDataEnvironmentPostProcessor.java`

```java
package com.iris.boot.context.config;

import com.iris.framework.core.env.Environment;

public class ConfigDataEnvironmentPostProcessor {

    public static void applyTo(Environment environment) {
        new ConfigDataEnvironment(environment).processAndApply();
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java` (excerpt)

```java
// Imports changed:
// - Removed: PropertiesPropertySourceLoader, MapPropertySource
// - Added: ConfigDataEnvironmentPostProcessor
// Constants removed: APPLICATION_PROPERTIES_SOURCE_NAME, APPLICATION_PROPERTIES_RESOURCE

private Environment prepareEnvironment(IrisApplicationRunListeners listeners) {
    StandardEnvironment environment = new StandardEnvironment();

    // Run the config data pipeline (replaces simple property loading)
    ConfigDataEnvironmentPostProcessor.applyTo(environment);

    listeners.environmentPrepared(environment);
    return environment;
}
```

### Test Code

#### Test Resources

**`application-dev.properties`:**
```properties
app.name=Iris Boot Dev
app.environment=development
server.port=8081
```

**`application-prod.properties`:**
```properties
app.name=Iris Boot Prod
app.environment=production
server.port=443
```

**`import-source.properties`:**
```properties
imported.key=imported-value
imported.message=Hello from imported config
```

#### `ConfigDataLocationTest.java`

*(7 tests — parsing, optional prefix, equality)*

#### `StandardConfigDataLocationResolverTest.java`

*(10 tests — directory resolution, profile resolution, catch-all behavior)*

#### `StandardConfigDataLoaderTest.java`

*(6 tests — classpath loading, file loading, missing resources)*

#### `ConfigDataEnvironmentTest.java`

*(11 tests — base loading, profile loading, precedence, imports, multi-profile)*

#### `ProfilesAndConfigDataIntegrationTest.java`

*(5 tests — full bootstrap with profiles, @Value injection, system property activation)*

---

## Summary

| What | Details |
|---|---|
| **Classes created** | `ConfigDataLocation`, `ConfigDataResource`, `ConfigData`, `ConfigDataLocationResolver`, `ConfigDataLoader`, `StandardConfigDataLocationResolver`, `StandardConfigDataLoader`, `ConfigDataEnvironment`, `ConfigDataEnvironmentPostProcessor` |
| **Classes modified** | `Environment` (added profile methods), `StandardEnvironment` (added profile tracking), `IrisApplication` (replaced property loading with ConfigData pipeline) |
| **Pattern** | Strategy (Resolver + Loader SPIs), Three-phase loading algorithm |
| **Key property** | `spring.profiles.active` — comma-separated list of profiles to activate |
| **Search locations** | `classpath:/`, `classpath:/config/`, `file:./`, `file:./config/` |
| **Precedence** | system props > system env > imported > profile-specific > base |

### Next Chapter Preview

**Chapter 18: Failure Analyzers** — Transform cryptic startup exceptions into human-readable error messages. When `IrisApplication.run()` catches a `BeanCreationException` or `BindException`, a chain of `FailureAnalyzer` implementations walks the exception cause chain, matches specific exception types, and produces a `FailureAnalysis` with a description and suggested action. You'll build the `FailureAnalyzer` SPI, an `AbstractFailureAnalyzer<T>` base class, and concrete analyzers for circular dependencies, port conflicts, and property binding errors.
