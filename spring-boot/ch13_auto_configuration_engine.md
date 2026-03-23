# Chapter 13: Auto-Configuration Engine

> Line references based on commits `11ab0b4351` (Spring Framework) and `5922311a95a` (Spring Boot).

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| `ConfigurationClassProcessor` processes user `@Configuration` classes with conditional evaluation (Features 4 + 12), but every auto-configuration class must be explicitly registered by hand — there is no discovery mechanism | No way to automatically discover and apply library-provided `@Configuration` classes from JARs on the classpath; the "just add a dependency and it works" experience that Spring Boot is known for does not exist | Build an auto-configuration engine that discovers `@AutoConfiguration` classes from `META-INF/iris/AutoConfiguration.imports` files, sorts them by ordering constraints, and processes them in a deferred second pass so that `@ConditionalOnMissingBean` can see user-defined beans and back off appropriately |

---

## 13.1 The Integration Point: Two-Pass Configuration Processing

The integration point is `AnnotationConfigApplicationContext.invokeBeanFactoryPostProcessors()` — the method that kicks off `@Configuration` class processing during `refresh()`. It changes from a single-pass method to a **two-pass** method. The first pass processes user `@Configuration` classes (everything explicitly registered). The second pass invokes a `DeferredConfigurationLoader` callback to discover additional configuration classes (auto-configuration candidates), registers them, and re-runs the processor. Because the second pass happens *after* user beans are registered, `@ConditionalOnMissingBean` on auto-configuration `@Bean` methods can detect user beans and back off.

Two subsystems connect here: the existing `ConfigurationClassProcessor` (user config processing) and the new `AutoConfigurationImportSelector` (auto-config candidate loading). They meet inside `invokeBeanFactoryPostProcessors()`.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java`
**Change:** Add `deferredConfigurationLoader` field, change `invokeBeanFactoryPostProcessors()` from `private` to `protected` with two-pass processing, add `setDeferredConfigurationLoader()` method

```java
/**
 * Optional loader for deferred configuration classes (e.g., auto-configuration).
 */
private DeferredConfigurationLoader deferredConfigurationLoader;

protected void invokeBeanFactoryPostProcessors() {
    ConfigurationClassProcessor processor = new ConfigurationClassProcessor();

    // First pass: process user @Configuration classes
    processor.processConfigurationClasses(this.beanFactory, this.environment);

    // Second pass: process deferred configurations (auto-configuration)
    if (this.deferredConfigurationLoader != null) {
        List<Class<?>> deferredConfigs =
                this.deferredConfigurationLoader.loadConfigurations(
                        this.beanFactory, this.environment);
        if (!deferredConfigs.isEmpty()) {
            for (Class<?> configClass : deferredConfigs) {
                String beanName = ConfigurationClassProcessor.deriveConfigBeanName(configClass);
                if (!this.beanFactory.containsBeanDefinition(beanName)) {
                    this.beanFactory.registerBeanDefinition(
                            beanName, new BeanDefinition(configClass));
                }
            }
            // Re-run: processes newly registered auto-config classes;
            // already-processed user classes are re-processed idempotently
            processor.processConfigurationClasses(this.beanFactory, this.environment);
        }
    }
}

public void setDeferredConfigurationLoader(DeferredConfigurationLoader loader) {
    this.deferredConfigurationLoader = loader;
}
```

The diff against the previous version:

```diff
-private void invokeBeanFactoryPostProcessors() {
-    ConfigurationClassProcessor processor = new ConfigurationClassProcessor();
-    processor.processConfigurationClasses(this.beanFactory, this.environment);
-}
+private DeferredConfigurationLoader deferredConfigurationLoader;
+
+protected void invokeBeanFactoryPostProcessors() {
+    ConfigurationClassProcessor processor = new ConfigurationClassProcessor();
+
+    // First pass: process user @Configuration classes
+    processor.processConfigurationClasses(this.beanFactory, this.environment);
+
+    // Second pass: process deferred configurations (auto-configuration)
+    if (this.deferredConfigurationLoader != null) {
+        List<Class<?>> deferredConfigs =
+                this.deferredConfigurationLoader.loadConfigurations(
+                        this.beanFactory, this.environment);
+        if (!deferredConfigs.isEmpty()) {
+            for (Class<?> configClass : deferredConfigs) {
+                String beanName = ConfigurationClassProcessor
+                        .deriveConfigBeanName(configClass);
+                if (!this.beanFactory.containsBeanDefinition(beanName)) {
+                    this.beanFactory.registerBeanDefinition(
+                            beanName, new BeanDefinition(configClass));
+                }
+            }
+            processor.processConfigurationClasses(
+                    this.beanFactory, this.environment);
+        }
+    }
+}
+
+public void setDeferredConfigurationLoader(DeferredConfigurationLoader loader) {
+    this.deferredConfigurationLoader = loader;
+}
```

Three key decisions here:

1. **Why two passes instead of one?** The entire auto-configuration concept depends on user beans being visible before auto-config conditions are evaluated. If we processed everything in one pass, `@ConditionalOnMissingBean` on an auto-config `@Bean` method would not see the user's bean (it might not be registered yet), and both beans would be registered. The two-pass design guarantees that all user `@Configuration` classes and their `@Bean` methods are fully processed before any auto-configuration class is even registered.

2. **Why re-run `processConfigurationClasses()` instead of a targeted process?** Re-running the full processor is idempotent for already-processed user classes (their `@Bean` methods are already registered, so the processor skips them). Only the newly registered auto-config classes produce new bean definitions. This keeps the code simple — no need for a separate "process only these classes" API.

3. **Why `containsBeanDefinition()` check before registering?** Multiple `.imports` files across JARs might list the same class. The deduplication in `AutoConfigurationImportSelector` handles most cases, but this guard prevents overwriting a user-registered bean definition with an auto-config one if they happen to share a name.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`
**Change:** Switch from `beanClass.isAnnotationPresent(Configuration.class)` to `AnnotationUtils.hasAnnotation(beanClass, Configuration.class)` in both Phase 2 and `performComponentScanning`

```diff
-if (!beanClass.isAnnotationPresent(Configuration.class)) {
+if (!AnnotationUtils.hasAnnotation(beanClass, Configuration.class)) {
```

This change is necessary because `@AutoConfiguration` is meta-annotated with `@Configuration` — it does not directly carry `@Configuration`. The `isAnnotationPresent()` check only finds direct annotations, while `AnnotationUtils.hasAnnotation()` traverses the meta-annotation tree. Without this change, auto-configuration classes would be silently ignored by the processor.

This connects the **context lifecycle** to the **auto-configuration discovery system**. To make it work, we need to build:
- `DeferredConfigurationLoader` — the callback interface (in `iris-framework`)
- `@AutoConfiguration`, `@AutoConfigureBefore`, `@AutoConfigureAfter` — the ordering annotations
- `@EnableAutoConfiguration` — the trigger annotation
- `ImportCandidates` — the `.imports` file reader
- `AutoConfigurationImportSelector` — the loader that ties everything together
- `AutoConfigurationSorter` — the alphabetical + topological sorter
- `IrisApplication.configureAutoConfiguration()` — the bootstrap wiring

---

## 13.2 The Deferred Configuration Loader Callback (iris-framework)

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/DeferredConfigurationLoader.java`

```java
package com.iris.framework.context.annotation;

import java.util.List;

import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.env.Environment;

@FunctionalInterface
public interface DeferredConfigurationLoader {

    List<Class<?>> loadConfigurations(BeanDefinitionRegistry registry,
                                       Environment environment);
}
```

This is the simplified equivalent of Spring Framework's `DeferredImportSelector`. The "deferred" aspect is critical: by running after user configurations, conditions like `@ConditionalOnMissingBean` can detect user-defined beans and back off.

The real `DeferredImportSelector`:
- Extends `ImportSelector` (the general mechanism for programmatic imports)
- Supports a `Group` mechanism for batching and sorting multiple selectors
- Is processed inside `ConfigurationClassParser` after all direct configs are parsed

We simplify to a single `@FunctionalInterface` callback that returns fully-loaded `Class<?>` objects, eliminating the need for `@Import`, `ImportSelector`, and the `Group` mechanism.

---

## 13.3 Auto-Configuration Annotations (iris-boot-core)

### @AutoConfiguration

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfiguration.java`

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.*;
import com.iris.framework.context.annotation.Configuration;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration                     // <-- meta-annotated with @Configuration
public @interface AutoConfiguration {
    Class<?>[] before() default {};
    String[] beforeName() default {};
    Class<?>[] after() default {};
    String[] afterName() default {};
}
```

The `@Configuration` meta-annotation is the key detail. It means any class annotated with `@AutoConfiguration` is *also* a `@Configuration` class — the `ConfigurationClassProcessor` will process its `@Bean` methods. The `before`/`after` attributes declare ordering relative to other auto-configuration classes, and `beforeName`/`afterName` allow ordering by class name (for classes that might not be on the classpath at compile time).

### @AutoConfigureBefore and @AutoConfigureAfter

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigureBefore.java`

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AutoConfigureBefore {
    Class<?>[] value() default {};
    String[] name() default {};
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigureAfter.java`

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AutoConfigureAfter {
    Class<?>[] value() default {};
    String[] name() default {};
}
```

These are standalone ordering annotations that can be placed directly on `@AutoConfiguration` classes. The `AutoConfigurationSorter` reads both these annotations and the `before`/`after` attributes on `@AutoConfiguration` itself. In the real Spring Boot, `@AliasFor` bridges between the two — we read both directly since we do not have `@AliasFor`.

### @EnableAutoConfiguration

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/EnableAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableAutoConfiguration {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

This is the trigger annotation. When present on the primary source class, the bootstrap layer (`IrisApplication`) sets up the `AutoConfigurationImportSelector` as a deferred loader. In the real Spring Boot, this annotation carries `@Import(AutoConfigurationImportSelector.class)` which triggers auto-config loading via the `DeferredImportSelector` mechanism. We use direct detection since we do not have `@Import`.

---

## 13.4 The .imports File Reader: ImportCandidates

**New file:** `iris-boot-core/src/main/java/com/iris/boot/context/annotation/ImportCandidates.java`

```java
package com.iris.boot.context.annotation;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import java.util.*;

public class ImportCandidates {

    private static final String LOCATION_PATTERN = "META-INF/iris/%s.imports";

    private final List<String> candidates;

    private ImportCandidates(List<String> candidates) {
        this.candidates = Collections.unmodifiableList(candidates);
    }

    public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
        String location = String.format(LOCATION_PATTERN, annotation.getName());
        List<String> candidates = new ArrayList<>();

        try {
            Enumeration<URL> urls = classLoader.getResources(location);
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                candidates.addAll(readCandidateConfigurations(url));
            }
        } catch (IOException ex) {
            throw new IllegalArgumentException(
                    "Unable to load auto-configuration candidates from '"
                            + location + "'", ex);
        }

        return new ImportCandidates(candidates);
    }

    private static List<String> readCandidateConfigurations(URL url)
            throws IOException {
        List<String> candidates = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(url.openStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                int commentIdx = line.indexOf('#');
                if (commentIdx >= 0) {
                    line = line.substring(0, commentIdx);
                }
                line = line.trim();
                if (!line.isEmpty()) {
                    candidates.add(line);
                }
            }
        }
        return candidates;
    }

    public List<String> getCandidates() {
        return this.candidates;
    }
}
```

The file location follows the pattern `META-INF/iris/{annotation-fully-qualified-name}.imports`. For `@AutoConfiguration`, this becomes `META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`.

Key details:
- **`classLoader.getResources()`** returns *all* matching resources across all JARs — this is how multiple modules can each contribute their own auto-configuration candidates
- **Comment support** (`#`) and blank line handling make the file format human-friendly
- **Unmodifiable list** prevents accidental mutation after loading
- This replaced the older `spring.factories` SPI mechanism (removed in Spring Boot 3.x)

---

## 13.5 The Sorter: AutoConfigurationSorter

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigurationSorter.java`

```java
package com.iris.boot.autoconfigure;

import java.util.*;
import java.util.stream.Collectors;

public class AutoConfigurationSorter {

    private final Map<String, Class<?>> classMap;

    public AutoConfigurationSorter(Map<String, Class<?>> classes) {
        this.classMap = classes;
    }

    public List<String> getInPriorityOrder(List<String> classNames) {
        // Phase 1: Sort alphabetically (stable baseline)
        List<String> sorted = new ArrayList<>(classNames);
        Collections.sort(sorted);

        // Phase 2: Topological sort by @AutoConfigureBefore / @AutoConfigureAfter
        sorted = sortByAnnotation(sorted);

        return sorted;
    }

    private List<String> sortByAnnotation(List<String> classNames) {
        Set<String> candidates = new LinkedHashSet<>(classNames);
        Map<String, Set<String>> afterMap = buildAfterMap(candidates);

        LinkedHashSet<String> sorted = new LinkedHashSet<>();
        Set<String> processing = new LinkedHashSet<>();

        for (String className : classNames) {
            if (!sorted.contains(className)) {
                doSortByAfterAnnotation(className, candidates, afterMap,
                        sorted, processing);
            }
        }

        return new ArrayList<>(sorted);
    }

    private void doSortByAfterAnnotation(String current,
                                          Set<String> candidates,
                                          Map<String, Set<String>> afterMap,
                                          LinkedHashSet<String> sorted,
                                          Set<String> processing) {
        if (sorted.contains(current)) {
            return;
        }
        if (!processing.add(current)) {
            throw new IllegalStateException(
                    "Auto-configuration cycle detected involving: " + current
                            + ". Processing stack: " + processing);
        }

        Set<String> afterClasses =
                getClassesRequestedAfter(current, candidates, afterMap);
        for (String after : afterClasses) {
            doSortByAfterAnnotation(after, candidates, afterMap,
                    sorted, processing);
        }

        processing.remove(current);
        sorted.add(current);
    }

    // ... annotation reading methods for before/after constraints
}
```

The sorting algorithm has two phases:

1. **Alphabetical sort** — provides a deterministic baseline when no ordering constraints exist
2. **Topological sort** — applies `@AutoConfigureBefore`/`@AutoConfigureAfter` constraints using depth-first traversal with cycle detection

The `getClassesRequestedAfter()` method merges two sources for each class:
- Classes listed in `@AutoConfigureAfter` / `@AutoConfiguration(after=...)`: "I should run after X" means X must come before me
- Classes whose `@AutoConfigureBefore` / `@AutoConfiguration(before=...)` lists this class: "X should run before me" means X must come before me

In the real Spring Boot, `AutoConfigurationSorter` has three phases (alphabetical, `@AutoConfigureOrder` numeric priority, then topological). We skip the numeric priority phase for simplicity.

---

## 13.6 The Selector: AutoConfigurationImportSelector

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigurationImportSelector.java`

```java
package com.iris.boot.autoconfigure;

import java.util.*;

import com.iris.boot.context.annotation.ImportCandidates;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.context.annotation.DeferredConfigurationLoader;
import com.iris.framework.core.env.Environment;

public class AutoConfigurationImportSelector
        implements DeferredConfigurationLoader {

    private static final String EXCLUDE_PROPERTY = "iris.autoconfigure.exclude";

    private final Set<String> exclusions;

    public AutoConfigurationImportSelector(Set<String> exclusions) {
        this.exclusions = (exclusions != null)
                ? Collections.unmodifiableSet(exclusions)
                : Collections.emptySet();
    }

    public AutoConfigurationImportSelector() {
        this(Collections.emptySet());
    }

    @Override
    public List<Class<?>> loadConfigurations(BeanDefinitionRegistry registry,
                                              Environment environment) {
        // 1. Load candidate class names from .imports files
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (classLoader == null) {
            classLoader = getClass().getClassLoader();
        }

        ImportCandidates importCandidates =
                ImportCandidates.load(AutoConfiguration.class, classLoader);
        List<String> candidates =
                new ArrayList<>(importCandidates.getCandidates());

        if (candidates.isEmpty()) {
            return List.of();
        }

        // 2. Remove duplicates (preserve order)
        candidates = new ArrayList<>(new LinkedHashSet<>(candidates));

        // 3. Apply exclusions
        Set<String> allExclusions = collectExclusions(environment);
        candidates.removeAll(allExclusions);

        if (candidates.isEmpty()) {
            return List.of();
        }

        // 4. Load classes (skip those whose dependencies aren't available)
        Map<String, Class<?>> classMap = new LinkedHashMap<>();
        for (String candidate : candidates) {
            try {
                Class<?> clazz = Class.forName(candidate, false, classLoader);
                classMap.put(candidate, clazz);
            } catch (ClassNotFoundException ex) {
                // Class or its dependency not on classpath — skip silently
            }
        }

        if (classMap.isEmpty()) {
            return List.of();
        }

        // 5. Sort by ordering constraints
        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classMap);
        List<String> sorted =
                sorter.getInPriorityOrder(new ArrayList<>(classMap.keySet()));

        // 6. Return loaded classes in sorted order
        List<Class<?>> result = new ArrayList<>(sorted.size());
        for (String name : sorted) {
            Class<?> clazz = classMap.get(name);
            if (clazz != null) {
                result.add(clazz);
            }
        }

        return result;
    }

    private Set<String> collectExclusions(Environment environment) {
        Set<String> allExclusions = new LinkedHashSet<>(this.exclusions);

        if (environment != null) {
            String propertyExclusions =
                    environment.getProperty(EXCLUDE_PROPERTY);
            if (propertyExclusions != null && !propertyExclusions.isBlank()) {
                for (String name : propertyExclusions.split(",")) {
                    String trimmed = name.trim();
                    if (!trimmed.isEmpty()) {
                        allExclusions.add(trimmed);
                    }
                }
            }
        }

        return allExclusions;
    }
}
```

The `loadConfigurations()` method is a six-step pipeline:

1. **Load candidates** from `.imports` files via `ImportCandidates`
2. **Deduplicate** — multiple JARs may list the same class
3. **Exclude** — from both `@EnableAutoConfiguration(exclude=...)` and the `iris.autoconfigure.exclude` property
4. **Load classes** — `Class.forName()` with `false` (no initialization); silently skip classes whose dependencies are not on the classpath
5. **Sort** — delegate to `AutoConfigurationSorter` for alphabetical + topological ordering
6. **Return** the loaded classes in sorted order

---

## 13.7 The Bootstrap Wiring: IrisApplication.configureAutoConfiguration()

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java`
**Change:** Add `configureAutoConfiguration()` and `getAutoConfigExclusions()` methods, call from `prepareContext()`

```java
private void prepareContext(ConfigurableApplicationContext context,
                             Environment environment,
                             ApplicationArguments applicationArguments,
                             IrisApplicationRunListeners listeners) {
    context.setEnvironment(environment);

    if (context instanceof AnnotationConfigApplicationContext annotationContext) {
        annotationContext.register(this.primarySource);

        BeanDefinition argsBd = new BeanDefinition(DefaultApplicationArguments.class);
        argsBd.setSupplier(() -> applicationArguments);
        annotationContext.getBeanFactory().registerBeanDefinition(
                "irisApplicationArguments", argsBd);

        // NEW: Set up auto-configuration if @EnableAutoConfiguration is present
        configureAutoConfiguration(annotationContext);
    }

    listeners.contextPrepared(context);
}

private void configureAutoConfiguration(
        AnnotationConfigApplicationContext context) {
    if (!AnnotationUtils.hasAnnotation(
            this.primarySource, EnableAutoConfiguration.class)) {
        return;
    }

    Set<String> exclusions = getAutoConfigExclusions();
    context.setDeferredConfigurationLoader(
            new AutoConfigurationImportSelector(exclusions));
}

private Set<String> getAutoConfigExclusions() {
    Set<String> exclusions = new LinkedHashSet<>();

    EnableAutoConfiguration annotation =
            this.primarySource.getAnnotation(EnableAutoConfiguration.class);
    if (annotation != null) {
        for (Class<?> excluded : annotation.exclude()) {
            exclusions.add(excluded.getName());
        }
        for (String name : annotation.excludeName()) {
            if (!name.isBlank()) {
                exclusions.add(name.trim());
            }
        }
    }

    return exclusions;
}
```

The flow is:
1. `IrisApplication.prepareContext()` calls `configureAutoConfiguration()`
2. `configureAutoConfiguration()` checks for `@EnableAutoConfiguration` on the primary source (using `AnnotationUtils.hasAnnotation` for meta-annotation support — e.g., via a future `@IrisBootApplication`)
3. If present, it reads exclusions and creates an `AutoConfigurationImportSelector`
4. The selector is set on the context as the deferred loader
5. During `refresh()` → `invokeBeanFactoryPostProcessors()`, the second pass invokes the loader

This is the integration point between the bootstrap layer and the auto-configuration engine. In the real Spring Boot, this wiring happens implicitly via `@Import(AutoConfigurationImportSelector.class)` on `@EnableAutoConfiguration`.

---

## 13.8 Try It Yourself

<details>
<summary>Challenge 1: Trace the full auto-configuration lifecycle for a DataSource auto-config class</summary>

Given this `.imports` file:

```
com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration
com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration
```

And this user code:

```java
@Configuration
@EnableAutoConfiguration
public class MyApp {
    // No user beans
}
```

Trace the steps:

1. `IrisApplication.run()` calls `prepareContext()`, which calls `configureAutoConfiguration()`
2. `@EnableAutoConfiguration` is detected on `MyApp`, so an `AutoConfigurationImportSelector` is set as the deferred loader
3. `refresh()` calls `invokeBeanFactoryPostProcessors()`
4. **First pass:** `ConfigurationClassProcessor` processes `MyApp` — no `@Bean` methods, no component scan, nothing registered except `MyApp` itself
5. **Second pass:** The deferred loader is invoked:
   - `ImportCandidates.load()` reads the `.imports` file, returning both class names
   - Duplicates removed, exclusions applied (none)
   - `Class.forName()` loads both classes
   - `AutoConfigurationSorter` sorts: `TestDataSourceAutoConfiguration` has `@AutoConfigureAfter(TestJacksonAutoConfiguration.class)`, so Jackson goes first
   - Both classes are registered as bean definitions
6. `ConfigurationClassProcessor` re-runs:
   - `TestJacksonAutoConfiguration`: `@ConditionalOnClass(name = "com.fasterxml.jackson.databind.ObjectMapper")` — Jackson is on the classpath, passes. `@ConditionalOnMissingBean` on `objectMapperHolder()` — no existing bean, passes. Bean registered.
   - `TestDataSourceAutoConfiguration`: `@ConditionalOnProperty(name = "datasource.enabled", matchIfMissing = true)` — property missing but `matchIfMissing=true`, passes. `@ConditionalOnMissingBean` on `dataSourceHolder()` — no existing bean, passes. Bean registered.

Both auto-configured beans are now in the container.

</details>

<details>
<summary>Challenge 2: What happens if a user defines their own ObjectMapperHolder bean?</summary>

```java
@Configuration
@EnableAutoConfiguration
public class MyApp {
    @Bean
    public ObjectMapperHolder objectMapperHolder() {
        return new ObjectMapperHolder("custom");
    }
}
```

1. **First pass:** `ConfigurationClassProcessor` processes `MyApp` — registers `objectMapperHolder` bean definition with type `ObjectMapperHolder`
2. **Second pass:** `AutoConfigurationImportSelector` loads and sorts candidates; `TestJacksonAutoConfiguration` is registered
3. `ConfigurationClassProcessor` re-runs on `TestJacksonAutoConfiguration`:
   - Class-level `@ConditionalOnClass` passes (Jackson on classpath)
   - Method-level `@ConditionalOnMissingBean` on `objectMapperHolder()`: deduces type `ObjectMapperHolder` from method return type, finds existing bean definition of that type in registry → condition **fails** → bean **skipped**
4. The user's custom `ObjectMapperHolder("custom")` wins; the auto-configured default is never registered

This is the "back-off" pattern — auto-configuration defers to the user.

</details>

<details>
<summary>Challenge 3: Create a new .imports file for a hypothetical module</summary>

Suppose you have a `iris-security` module with two auto-configuration classes. Create the file:

**File:** `iris-security/src/main/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`

```
# Iris Security Auto-Configuration
com.iris.security.autoconfigure.SecurityAutoConfiguration
com.iris.security.autoconfigure.OAuth2AutoConfiguration
```

When this JAR is on the classpath, `ImportCandidates.load()` will discover *both* this file and any other `.imports` files from other JARs (via `classLoader.getResources()`). All candidates are merged into a single list, deduplicated, and sorted.

</details>

---

## 13.9 Tests

### Unit Tests -- ImportCandidates

**New file:** `iris-boot-core/src/test/java/com/iris/boot/context/annotation/ImportCandidatesTest.java`

```java
@Test
@DisplayName("should load candidates from .imports file on classpath")
void shouldLoadCandidates_WhenImportsFileExists() {
    ImportCandidates candidates = ImportCandidates.load(
            AutoConfiguration.class, getClass().getClassLoader());

    List<String> names = candidates.getCandidates();
    assertThat(names).isNotEmpty();
    assertThat(names).contains(
            "com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration",
            "com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration");
}

@Test
@DisplayName("should return empty list when no .imports file found")
void shouldReturnEmptyList_WhenNoImportsFileFound() {
    ImportCandidates candidates = ImportCandidates.load(
            Override.class, getClass().getClassLoader());
    assertThat(candidates.getCandidates()).isEmpty();
}

@Test
@DisplayName("should skip comment lines and blank lines")
void shouldSkipCommentsAndBlankLines() {
    ImportCandidates candidates = ImportCandidates.load(
            AutoConfiguration.class, getClass().getClassLoader());
    for (String name : candidates.getCandidates()) {
        assertThat(name).doesNotStartWith("#");
        assertThat(name).isNotBlank();
    }
}

@Test
@DisplayName("should return unmodifiable list")
void shouldReturnUnmodifiableList() {
    ImportCandidates candidates = ImportCandidates.load(
            AutoConfiguration.class, getClass().getClassLoader());
    Assertions.assertThrows(UnsupportedOperationException.class,
            () -> candidates.getCandidates().add("com.example.Foo"));
}
```

### Unit Tests -- AutoConfigurationSorter

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/AutoConfigurationSorterTest.java`

```java
@Test
@DisplayName("should sort alphabetically when no ordering constraints")
void shouldSortAlphabetically_WhenNoOrderingConstraints() {
    Map<String, Class<?>> classes = new LinkedHashMap<>();
    classes.put(GammaConfig.class.getName(), GammaConfig.class);
    classes.put(AlphaConfig.class.getName(), AlphaConfig.class);
    classes.put(BetaConfig.class.getName(), BetaConfig.class);

    AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
    List<String> sorted = sorter.getInPriorityOrder(List.of(
            GammaConfig.class.getName(),
            AlphaConfig.class.getName(),
            BetaConfig.class.getName()));

    assertThat(sorted).containsExactly(
            AlphaConfig.class.getName(),
            BetaConfig.class.getName(),
            GammaConfig.class.getName());
}

@Test
@DisplayName("should respect @AutoConfiguration(after=...) ordering")
void shouldRespectAfterOrdering() {
    // ... DependsOnAlphaConfig has @AutoConfiguration(after = AlphaConfig.class)
    assertThat(sorted.indexOf(AlphaConfig.class.getName()))
            .isLessThan(sorted.indexOf(DependsOnAlphaConfig.class.getName()));
}

@Test
@DisplayName("should respect combined before and after ordering")
void shouldRespectCombinedOrdering() {
    // MiddleConfig has @AutoConfiguration(after = Alpha, before = Gamma)
    assertThat(alphaIdx).isLessThan(middleIdx);
    assertThat(middleIdx).isLessThan(gammaIdx);
}

@Test
@DisplayName("should detect cycle and throw exception")
void shouldDetectCycle_AndThrowException() {
    // CycleA after CycleB, CycleB after CycleA
    assertThatThrownBy(() -> sorter.getInPriorityOrder(List.of(
            CycleA.class.getName(), CycleB.class.getName())))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("cycle");
}
```

### Unit Tests -- AutoConfigurationImportSelector

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/AutoConfigurationImportSelectorTest.java`

```java
@Test
@DisplayName("should load all auto-configuration candidates from .imports file")
void shouldLoadAllCandidates() {
    AutoConfigurationImportSelector selector =
            new AutoConfigurationImportSelector();
    List<Class<?>> configs =
            selector.loadConfigurations(registry, new StandardEnvironment());

    assertThat(configs).contains(
            TestJacksonAutoConfiguration.class,
            TestDataSourceAutoConfiguration.class,
            TestNonExistentLibAutoConfiguration.class);
}

@Test
@DisplayName("should exclude classes specified in constructor")
void shouldExcludeSpecifiedClasses() {
    Set<String> exclusions = Set.of(
            TestNonExistentLibAutoConfiguration.class.getName());
    AutoConfigurationImportSelector selector =
            new AutoConfigurationImportSelector(exclusions);
    List<Class<?>> configs =
            selector.loadConfigurations(registry, new StandardEnvironment());

    assertThat(configs).doesNotContain(TestNonExistentLibAutoConfiguration.class);
    assertThat(configs).contains(TestJacksonAutoConfiguration.class);
}

@Test
@DisplayName("should exclude classes specified via property")
void shouldExcludeViaProperty() {
    StandardEnvironment env = new StandardEnvironment();
    env.getPropertySources().addLast(new MapPropertySource("test", Map.of(
            "iris.autoconfigure.exclude",
            TestNonExistentLibAutoConfiguration.class.getName())));

    List<Class<?>> configs = selector.loadConfigurations(registry, env);
    assertThat(configs).doesNotContain(TestNonExistentLibAutoConfiguration.class);
}

@Test
@DisplayName("should sort by ordering constraints - DataSource after Jackson")
void shouldSortByOrderingConstraints() {
    List<Class<?>> configs = selector.loadConfigurations(registry, env);

    int jacksonIdx = configs.indexOf(TestJacksonAutoConfiguration.class);
    int dataSourceIdx = configs.indexOf(TestDataSourceAutoConfiguration.class);
    assertThat(jacksonIdx).isLessThan(dataSourceIdx);
}
```

### Integration Tests -- Full Auto-Configuration Lifecycle

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/integration/AutoConfigurationIntegrationTest.java`

```java
@Test
@DisplayName("should apply auto-configuration when no user beans defined")
void shouldApplyAutoConfig_WhenNoUserBeansDefined() {
    var ctx = createAutoConfigContext();

    assertThat(ctx.containsBean("objectMapperHolder")).isTrue();
    ObjectMapperHolder holder = ctx.getBean(ObjectMapperHolder.class);
    assertThat(holder.getName()).isEqualTo("auto-configured-jackson");

    assertThat(ctx.containsBean("dataSourceHolder")).isTrue();
    DataSourceHolder ds = ctx.getBean(DataSourceHolder.class);
    assertThat(ds.getUrl()).isEqualTo("jdbc:h2:mem:auto-configured");

    ctx.close();
}

@Test
@DisplayName("should back off when user provides same bean type")
void shouldBackOff_WhenUserProvidesSameBeanType() {
    var ctx = createAutoConfigContext(UserJacksonOverrideConfig.class);

    ObjectMapperHolder holder = ctx.getBean(ObjectMapperHolder.class);
    assertThat(holder.getName()).isEqualTo("user-defined-jackson");

    DataSourceHolder ds = ctx.getBean(DataSourceHolder.class);
    assertThat(ds.getUrl()).isEqualTo("jdbc:h2:mem:auto-configured");

    ctx.close();
}

@Test
@DisplayName("should skip auto-config when class condition fails")
void shouldSkipAutoConfig_WhenClassConditionFails() {
    var ctx = createAutoConfigContext();
    // TestNonExistentLibAutoConfiguration requires a non-existent class
    assertThat(ctx.containsBean("nonExistentBean")).isFalse();
    ctx.close();
}

@Test
@DisplayName("should process auto-configs AFTER user configs (deferred loading)")
void shouldProcessAutoConfigsAfterUserConfigs() {
    var ctx = createAutoConfigContext(UserJacksonOverrideConfig.class);

    ObjectMapperHolder holder = ctx.getBean(ObjectMapperHolder.class);
    assertThat(holder.getName())
            .as("Auto-config should back off when user bean exists "
                    + "(deferred processing)")
            .isEqualTo("user-defined-jackson");

    ctx.close();
}

@Test
@DisplayName("should work without deferred loader (no auto-configuration)")
void shouldWorkWithoutDeferredLoader() {
    var ctx = new AnnotationConfigApplicationContext(
            UserJacksonOverrideConfig.class);

    assertThat(ctx.getBean(ObjectMapperHolder.class).getName())
            .isEqualTo("user-defined-jackson");
    assertThat(ctx.containsBean("dataSourceHolder")).isFalse();

    ctx.close();
}
```

### Test Auto-Configuration Classes

Three test auto-configuration classes demonstrate the core patterns:

**`TestJacksonAutoConfiguration`** -- classpath condition + missing-bean back-off:

```java
@AutoConfiguration
@ConditionalOnClass(name = "com.fasterxml.jackson.databind.ObjectMapper")
public class TestJacksonAutoConfiguration {

    public static class ObjectMapperHolder {
        private final String name;
        public ObjectMapperHolder(String name) { this.name = name; }
        public String getName() { return name; }
    }

    @Bean
    @ConditionalOnMissingBean
    public ObjectMapperHolder objectMapperHolder() {
        return new ObjectMapperHolder("auto-configured-jackson");
    }
}
```

**`TestDataSourceAutoConfiguration`** -- ordering + property gate + missing-bean back-off:

```java
@AutoConfiguration
@AutoConfigureAfter(TestJacksonAutoConfiguration.class)
@ConditionalOnProperty(name = "datasource.enabled", matchIfMissing = true)
public class TestDataSourceAutoConfiguration {

    public static class DataSourceHolder {
        private final String url;
        public DataSourceHolder(String url) { this.url = url; }
        public String getUrl() { return url; }
    }

    @Bean
    @ConditionalOnMissingBean
    public DataSourceHolder dataSourceHolder() {
        return new DataSourceHolder("jdbc:h2:mem:auto-configured");
    }
}
```

**`TestNonExistentLibAutoConfiguration`** -- always skipped:

```java
@AutoConfiguration
@ConditionalOnClass(name = "com.example.NonExistentLibrary")
public class TestNonExistentLibAutoConfiguration {

    @Bean
    public String nonExistentBean() {
        return "should-never-exist";
    }
}
```

### Test Resource

**File:** `iris-boot-core/src/test/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`

```
# Test auto-configuration candidates
# Used by ImportCandidatesTest and AutoConfigurationIntegrationTest

com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration
com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration
com.iris.boot.autoconfigure.integration.TestNonExistentLibAutoConfiguration
```

**Run:** `./gradlew test` -- expected: all 28 new tests pass (4 ImportCandidates + 9 Sorter + 5 Selector + 10 Integration), along with all prior features' tests

---

## 13.10 Why This Works

> **Insight** -------------------------------------------
> **The two-pass design is what makes "just add a dependency and it works" possible.** The first pass fully processes user `@Configuration` classes, populating the registry with user-defined bean definitions. The second pass loads auto-configuration candidates and processes them with condition evaluation. At this point, `@ConditionalOnMissingBean` on an auto-config `@Bean` method can see the user's bean in the registry and back off. If both passes ran simultaneously, there would be a race: the auto-config's condition might evaluate before the user's bean is registered, causing both to be created. This ordering invariant -- user configs first, auto-configs second -- is the fundamental contract that the entire Spring Boot auto-configuration ecosystem depends on.
>
> In the real Spring Framework, the equivalent mechanism is `DeferredImportSelector`, which is processed in `ConfigurationClassParser.processDeferredImportSelectors()` -- explicitly after all other `@Import`s and `@Configuration` classes have been fully parsed.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> **The `.imports` file acts as a service provider interface (SPI) without Java's `ServiceLoader`.** Each JAR can ship its own `META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports` file, and `classLoader.getResources()` discovers them all. This is the same pattern as Java's `ServiceLoader` (`META-INF/services/`) but with comment support, a friendlier format, and integration with the auto-configuration pipeline. The `.imports` mechanism replaced Spring Boot's older `spring.factories` approach in Boot 3.0, moving from a single key-value properties file to per-annotation import files.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> **Topological sorting with cycle detection prevents misconfigured ordering from creating silent bugs.** Without the cycle check, two classes that each declare `@AutoConfigureAfter` pointing at each other would cause infinite recursion during sorting. The `processing` set in `doSortByAfterAnnotation()` tracks the DFS stack -- if we revisit a node already on the stack, we have found a cycle and throw a clear `IllegalStateException` with the involved class names. This makes ordering mistakes fail fast with a clear message rather than causing subtle, hard-to-debug infinite loops.
> -----------------------------------------------------------

---

## 13.11 What We Enhanced

| Aspect | Before (ch12) | Current (ch13) | Real Framework |
|--------|---------------|-----------------|----------------|
| Configuration processing | Single pass -- all `@Configuration` classes processed at once | Two-pass -- user configs first, deferred auto-configs second via `DeferredConfigurationLoader` | `ConfigurationClassParser` processes `DeferredImportSelector`s after all direct configs (`ConfigurationClassParser.java:206`) |
| `invokeBeanFactoryPostProcessors()` | Single `processConfigurationClasses()` call | Two calls: first for user configs, second for deferred auto-configs after loader invocation | `PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()` with `BeanDefinitionRegistryPostProcessor` priority ordering |
| `@Configuration` detection | `beanClass.isAnnotationPresent(Configuration.class)` | `AnnotationUtils.hasAnnotation(beanClass, Configuration.class)` -- traverses meta-annotations | `MergedAnnotations.from()` with full annotation hierarchy traversal (`ConfigurationClassUtils.java:134`) |
| Auto-config discovery | None -- all configs must be explicitly registered | `.imports` files on classpath discovered via `ImportCandidates.load()` | `ImportCandidates.load()` reading from `META-INF/spring/{annotation}.imports` (`ImportCandidates.java:81`) |
| Auto-config ordering | N/A | Alphabetical + topological sort via `@AutoConfigureBefore`/`@AutoConfigureAfter` with cycle detection | Three-phase sort: alphabetical + `@AutoConfigureOrder` + topological (`AutoConfigurationSorter.java:65`) |
| Auto-config exclusion | N/A | `@EnableAutoConfiguration(exclude=...)` + `iris.autoconfigure.exclude` property | Same two sources plus `spring.autoconfigure.exclude` property (`AutoConfigurationImportSelector.java:142`) |
| Bootstrap wiring | `IrisApplication` creates context and calls `refresh()` | `IrisApplication.configureAutoConfiguration()` detects `@EnableAutoConfiguration` and wires the selector | `@Import(AutoConfigurationImportSelector.class)` on `@EnableAutoConfiguration` -- implicit via framework `@Import` mechanism (`EnableAutoConfiguration.java:77`) |

---

## 13.12 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `DeferredConfigurationLoader` | `DeferredImportSelector` | `spring-context/.../DeferredImportSelector.java:54` | Real extends `ImportSelector`, supports `Group` mechanism for batching/sorting; we use a single callback returning all configs |
| `@AutoConfiguration` | `@AutoConfiguration` | `spring-boot-autoconfigure/.../AutoConfiguration.java:58` | Real uses `@Configuration(proxyBeanMethods = false)` and `@AliasFor` to bridge before/after to separate annotations |
| `@EnableAutoConfiguration` | `@EnableAutoConfiguration` | `spring-boot-autoconfigure/.../EnableAutoConfiguration.java:77` | Real has `@Import(AutoConfigurationImportSelector.class)` -- wiring is implicit via framework; we use explicit detection in bootstrap |
| `AutoConfigurationImportSelector` | `AutoConfigurationImportSelector` | `spring-boot-autoconfigure/.../AutoConfigurationImportSelector.java:78` | Real implements `DeferredImportSelector` with `Group` mechanism (`AutoConfigurationGroup`), applies `AutoConfigurationImportFilter` for fast-path filtering, fires `AutoConfigurationImportEvent` |
| `AutoConfigurationImportSelector.loadConfigurations()` | `AutoConfigurationImportSelector.getAutoConfigurationEntry()` | `spring-boot-autoconfigure/.../AutoConfigurationImportSelector.java:142` | Real also handles `@Exclude` from `@SpringBootApplication`, `AutoConfigurationMetadata` for ASM-based pre-filtering |
| `ImportCandidates` | `ImportCandidates` | `spring-boot/.../ImportCandidates.java:47` | Nearly identical -- same `LOCATION` pattern, same `load()` method, same comment stripping; real uses `META-INF/spring/` prefix instead of `META-INF/iris/` |
| `ImportCandidates.load()` | `ImportCandidates.load()` | `spring-boot/.../ImportCandidates.java:81` | Real uses `SpringFactoriesLoader`-style `Enumeration<URL>` iteration -- identical to our approach |
| `AutoConfigurationSorter` | `AutoConfigurationSorter` | `spring-boot-autoconfigure/.../AutoConfigurationSorter.java:65` | Real has three-phase sort (alphabetical + `@AutoConfigureOrder` numeric priority + topological); we skip numeric priority |
| `AnnotationUtils.hasAnnotation()` in `ConfigurationClassProcessor` | `ConfigurationClassUtils.checkConfigurationClassCandidate()` | `spring-context/.../ConfigurationClassUtils.java:134` | Real uses `MergedAnnotations` with `SearchStrategy.TYPE_HIERARCHY` for deep traversal; we use recursive `hasAnnotation()` |
| Two-pass processing in `invokeBeanFactoryPostProcessors()` | `ConfigurationClassParser.processDeferredImportSelectors()` | `spring-context/.../ConfigurationClassParser.java:206` | Real processes deferred selectors inside the parser after all direct configs; we re-run the entire processor for simplicity |
| `IrisApplication.configureAutoConfiguration()` | `@Import` on `@EnableAutoConfiguration` | `spring-boot-autoconfigure/.../EnableAutoConfiguration.java:77` | Real relies on framework `@Import` mechanism; we use explicit bootstrap detection |

---

## 13.13 Complete Code

### Production Code

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/DeferredConfigurationLoader.java` [NEW]

```java
package com.iris.framework.context.annotation;

import java.util.List;

import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.env.Environment;

/**
 * Callback interface for loading configuration classes in a <em>deferred</em>
 * manner — after all explicitly registered {@code @Configuration} classes have
 * been processed.
 *
 * <p>This is the simplified equivalent of Spring Framework's
 * {@link org.springframework.context.annotation.DeferredImportSelector
 * DeferredImportSelector}. The "deferred" aspect is critical for
 * auto-configuration: by running after user configurations, conditions like
 * {@code @ConditionalOnMissingBean} can detect user-defined beans and back
 * off appropriately.
 *
 * <h3>How it works</h3>
 *
 * <p>During {@code ApplicationContext.refresh()}, the context:
 * <ol>
 *   <li>Processes all explicitly registered {@code @Configuration} classes
 *       (component scanning + {@code @Bean} methods)</li>
 *   <li>Invokes this loader to obtain additional configuration classes</li>
 *   <li>Registers and processes those deferred classes in a second pass</li>
 * </ol>
 *
 * <p>The second pass re-runs the {@code ConfigurationClassProcessor}, which
 * is idempotent for already-processed user classes. Only the newly-registered
 * deferred classes produce new bean definitions. Condition evaluation during
 * this second pass can see all user-defined beans, enabling the "back-off"
 * pattern.
 *
 * <p>In the real Spring Framework, {@code DeferredImportSelector}:
 * <ul>
 *   <li>Extends {@code ImportSelector} — the general mechanism for
 *       programmatic imports</li>
 *   <li>Supports a {@code Group} mechanism for batching and sorting multiple
 *       selectors</li>
 *   <li>Is processed inside {@code ConfigurationClassParser} after all direct
 *       configs are parsed</li>
 * </ul>
 *
 * <p>We simplify to a single callback that returns fully-loaded
 * {@code Class<?>} objects, eliminating the need for {@code @Import},
 * {@code ImportSelector}, and the {@code Group} mechanism.
 *
 * @see org.springframework.context.annotation.DeferredImportSelector
 */
@FunctionalInterface
public interface DeferredConfigurationLoader {

    /**
     * Load configuration classes to be processed after user configurations.
     *
     * <p>The returned classes should be {@code @Configuration}-annotated
     * (directly or via meta-annotation like {@code @AutoConfiguration}).
     * They will be registered as bean definitions and processed by
     * {@code ConfigurationClassProcessor} with full condition evaluation.
     *
     * @param registry    the bean definition registry (for inspecting registered beans)
     * @param environment the environment (for reading properties)
     * @return the list of configuration classes to process, in priority order;
     *         may be empty but never {@code null}
     */
    List<Class<?>> loadConfigurations(BeanDefinitionRegistry registry,
                                       Environment environment);
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java` [MODIFIED]

```java
package com.iris.framework.context;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.ValueAnnotationBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;
import com.iris.framework.context.annotation.DeferredConfigurationLoader;
import com.iris.framework.context.event.ContextClosedEvent;
import com.iris.framework.context.event.ContextRefreshedEvent;
import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.PropertiesPropertySourceLoader;
import com.iris.framework.core.env.StandardEnvironment;

/**
 * Standalone application context that accepts {@link com.iris.framework.context.annotation.Configuration @Configuration}
 * classes as input — the primary context implementation for annotation-based configuration.
 *
 * <p>Usage:
 * <pre>{@code
 * var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
 * MyService svc = ctx.getBean(MyService.class);
 * // ... use beans ...
 * ctx.close();
 * }</pre>
 *
 * <p>The key method is {@link #refresh()}, which orchestrates the full container lifecycle:
 * <ol>
 *   <li>Process {@code @Configuration} classes (component scanning + {@code @Bean} methods)</li>
 *   <li>Register internal {@code BeanPostProcessor}s ({@code AutowiredBeanPostProcessor},
 *       {@code LifecycleBeanPostProcessor}, {@code ValueAnnotationBeanPostProcessor})</li>
 *   <li>Discover and register user-defined {@code BeanPostProcessor} beans</li>
 *   <li>Eagerly instantiate all singleton beans</li>
 *   <li>Discover {@code ApplicationListener} beans and publish {@code ContextRefreshedEvent}</li>
 * </ol>
 *
 * <p>Environment setup happens BEFORE {@code refresh()}: a {@link StandardEnvironment}
 * is created with system properties and system environment variables, then
 * {@code application.properties} is loaded from the classpath (if present)
 * and added as the lowest-priority property source.
 *
 * <p>In the real Spring Framework, this class hierarchy is:
 * <pre>
 * AnnotationConfigApplicationContext
 *   → GenericApplicationContext
 *     → AbstractApplicationContext (the refresh() template)
 *       → DefaultResourceLoader
 * </pre>
 * We collapse the three-level hierarchy into a single concrete class. The real
 * {@code AbstractApplicationContext.refresh()} has 12 steps — we implement the
 * 5 that matter most for understanding the pattern.
 *
 * @see ConfigurableApplicationContext
 * @see Environment
 * @see org.springframework.context.annotation.AnnotationConfigApplicationContext
 * @see org.springframework.context.support.AbstractApplicationContext#refresh()
 */
public class AnnotationConfigApplicationContext implements ConfigurableApplicationContext {

    /** Well-known name for the application.properties property source. */
    private static final String APPLICATION_PROPERTIES_SOURCE_NAME = "applicationProperties";

    /** Well-known classpath location for the application properties file. */
    private static final String APPLICATION_PROPERTIES_RESOURCE = "application.properties";

    /** The internal bean factory that stores definitions and singletons. */
    private final DefaultBeanFactory beanFactory;

    /** The environment holding property sources for this context. */
    private Environment environment;

    /** Application listeners registered programmatically or discovered from beans. */
    private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();

    /** Tracks whether refresh() has been called and close() has not. */
    private final AtomicBoolean active = new AtomicBoolean(false);

    /** Tracks whether close() has been called. */
    private final AtomicBoolean closed = new AtomicBoolean(false);

    /**
     * Optional loader for deferred configuration classes (e.g., auto-configuration).
     *
     * <p>When set, the context invokes this loader after the first pass of
     * configuration processing, enabling the "back-off" pattern: deferred
     * configs can check what user beans are already registered.
     *
     * @see DeferredConfigurationLoader
     */
    private DeferredConfigurationLoader deferredConfigurationLoader;

    /**
     * Create a new context WITHOUT registering any classes and WITHOUT calling
     * {@link #refresh()}.
     *
     * <p>Use this constructor when you need to configure the context before
     * refreshing — for example, when the bootstrap layer ({@code IrisApplication})
     * needs to set the environment, register sources, and add listeners before
     * starting the container lifecycle.
     *
     * <p>After construction, call {@link #register(Class[])} to add configuration
     * classes, then call {@link #refresh()} to start the container.
     *
     * <p>In the real Spring Framework, this no-arg constructor is the default,
     * and the convenience constructor below delegates to it.
     */
    public AnnotationConfigApplicationContext() {
        this.beanFactory = new DefaultBeanFactory();
        this.environment = createEnvironment();
    }

    /**
     * Create a new context that registers the given {@code @Configuration}
     * classes and immediately calls {@link #refresh()}.
     *
     * <p>This matches the real Spring's convenience constructor:
     * {@code AnnotationConfigApplicationContext(Class<?>...)}.
     *
     * @param componentClasses one or more {@code @Configuration} or {@code @Component} classes
     */
    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this();
        register(componentClasses);
        refresh();
    }

    /**
     * Register one or more {@code @Configuration} or {@code @Component} classes
     * with this context.
     *
     * <p>Must be called BEFORE {@link #refresh()}. Each class is registered as
     * a {@link BeanDefinition} in the internal bean factory.
     *
     * <p>In the real Spring Framework, this method is on
     * {@code AnnotationConfigApplicationContext} and delegates to
     * {@code AnnotatedBeanDefinitionReader.register()}.
     *
     * @param componentClasses one or more annotated classes
     */
    public void register(Class<?>... componentClasses) {
        for (Class<?> componentClass : componentClasses) {
            String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
            beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
        }
    }

    /**
     * Create and configure the Environment.
     *
     * <p>Creates a {@link StandardEnvironment} (system properties + system env),
     * then loads {@code application.properties} from the classpath and adds it
     * as the lowest-priority property source.
     *
     * <p>In the real Spring Framework, environment creation happens in
     * {@code AbstractApplicationContext.getOrCreateEnvironment()}, and
     * {@code application.properties} loading is handled by Spring Boot's
     * {@code ConfigDataEnvironmentPostProcessor} during the bootstrap lifecycle
     * (before {@code refresh()}). We combine both steps here for simplicity.
     */
    private StandardEnvironment createEnvironment() {
        StandardEnvironment env = new StandardEnvironment();

        // Load application.properties if present on the classpath
        PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
        MapPropertySource appProps = loader.load(
                APPLICATION_PROPERTIES_SOURCE_NAME, APPLICATION_PROPERTIES_RESOURCE);
        if (appProps != null) {
            env.getPropertySources().addLast(appProps);
        }

        return env;
    }

    // -----------------------------------------------------------------------
    // ConfigurableApplicationContext — Lifecycle
    // -----------------------------------------------------------------------

    /**
     * The main startup method — orchestrates the full container initialization.
     *
     * <p>This is the simplified equivalent of
     * {@code AbstractApplicationContext.refresh()}'s 12-step pipeline. We implement
     * 6 steps that capture the essential pattern:
     *
     * <pre>
     * refresh()
     *   ├── 1. invokeBeanFactoryPostProcessors()    ← @Configuration + scanning
     *   ├── 2. registerBeanPostProcessors()          ← internal + user-defined
     *   ├── 3. onRefresh()                           ← template method hook (e.g., start web server)
     *   ├── 4. finishBeanFactoryInitialization()     ← eager singleton creation
     *   ├── 5. registerListeners()                   ← discover ApplicationListener beans
     *   └── 6. finishRefresh()                       ← publish ContextRefreshedEvent
     * </pre>
     */
    @Override
    public void refresh() {
        try {
            // Step 1: Process @Configuration classes — component scanning + @Bean methods
            invokeBeanFactoryPostProcessors();

            // Step 2: Register BeanPostProcessors (internal + user-defined)
            registerBeanPostProcessors();

            // Step 3: Template method — subclasses can hook in here
            // (e.g., ServletWebServerApplicationContext creates the embedded server)
            onRefresh();

            // Step 4: Instantiate all remaining singleton beans
            finishBeanFactoryInitialization();

            // Step 5: Discover ApplicationListener beans
            registerListeners();

            // Step 6: Publish ContextRefreshedEvent
            finishRefresh();

            this.active.set(true);
            this.closed.set(false);
        } catch (RuntimeException ex) {
            // On failure: destroy any already-created singletons
            this.beanFactory.close();
            this.active.set(false);
            throw ex;
        }
    }

    /**
     * Template method which can be overridden to add context-specific refresh work.
     * Called during {@link #refresh()} after {@code registerBeanPostProcessors()}
     * and before {@code finishBeanFactoryInitialization()}.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.onRefresh()} (line 906). It's the hook
     * where {@code ServletWebServerApplicationContext} creates the embedded
     * web server — the server is initialized here but does not accept connections
     * until {@code finishRefresh()}.
     *
     * <p>The default implementation is empty.
     *
     * @see org.springframework.context.support.AbstractApplicationContext#onRefresh()
     */
    protected void onRefresh() {
        // For subclasses to override — do nothing by default.
    }

    /**
     * Step 1: Process {@code @Configuration} classes.
     *
     * <p>Processing happens in two passes:
     * <ol>
     *   <li><b>First pass:</b> Process all explicitly registered
     *       {@code @Configuration} classes — component scanning + {@code @Bean}
     *       methods. This registers all user-defined beans.</li>
     *   <li><b>Second pass (deferred):</b> If a {@link DeferredConfigurationLoader}
     *       is set, invoke it to load additional configuration classes (e.g.,
     *       auto-configuration candidates). Register them and re-run the
     *       processor. Condition evaluation in the second pass can see all
     *       user beans from the first pass.</li>
     * </ol>
     *
     * <p>The two-pass design is the simplified equivalent of Spring's
     * {@code DeferredImportSelector}: user configs are fully processed
     * before auto-configs, enabling the "back-off" pattern where
     * {@code @ConditionalOnMissingBean} detects user beans.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.invokeBeanFactoryPostProcessors()},
     * which delegates to {@code PostProcessorRegistrationDelegate} to invoke
     * all {@code BeanFactoryPostProcessor} beans. The most important one is
     * {@code ConfigurationClassPostProcessor}, which handles {@code @Configuration},
     * {@code @ComponentScan}, {@code @Import}, etc.
     */
    protected void invokeBeanFactoryPostProcessors() {
        ConfigurationClassProcessor processor = new ConfigurationClassProcessor();

        // First pass: process user @Configuration classes
        processor.processConfigurationClasses(this.beanFactory, this.environment);

        // Second pass: process deferred configurations (auto-configuration)
        if (this.deferredConfigurationLoader != null) {
            List<Class<?>> deferredConfigs =
                    this.deferredConfigurationLoader.loadConfigurations(
                            this.beanFactory, this.environment);
            if (!deferredConfigs.isEmpty()) {
                for (Class<?> configClass : deferredConfigs) {
                    String beanName = ConfigurationClassProcessor.deriveConfigBeanName(configClass);
                    if (!this.beanFactory.containsBeanDefinition(beanName)) {
                        this.beanFactory.registerBeanDefinition(
                                beanName, new BeanDefinition(configClass));
                    }
                }
                // Re-run: processes newly registered auto-config classes;
                // already-processed user classes are re-processed idempotently
                processor.processConfigurationClasses(this.beanFactory, this.environment);
            }
        }
    }

    /**
     * Step 2: Register {@code BeanPostProcessor}s.
     *
     * <p>First registers the three internal processors that the container always needs:
     * <ol>
     *   <li>{@code AutowiredBeanPostProcessor} — handles {@code @Autowired} field injection</li>
     *   <li>{@code ValueAnnotationBeanPostProcessor} — handles {@code @Value} field injection</li>
     *   <li>{@code LifecycleBeanPostProcessor} — handles {@code @PostConstruct} / {@code @PreDestroy}</li>
     * </ol>
     *
     * <p>Then discovers any user-defined {@code BeanPostProcessor} beans registered via
     * {@code @Configuration}/{@code @Bean} or component scanning, and adds them too.
     *
     * <p>In the real Spring Framework, this is
     * {@code PostProcessorRegistrationDelegate.registerBeanPostProcessors()},
     * which sorts processors by {@code PriorityOrdered}, {@code Ordered}, and
     * then unordered. We skip the ordering for simplicity.
     */
    private void registerBeanPostProcessors() {
        // Internal processors — always present
        this.beanFactory.addBeanPostProcessor(new AutowiredBeanPostProcessor(this.beanFactory));
        this.beanFactory.addBeanPostProcessor(new ValueAnnotationBeanPostProcessor(this.environment));
        this.beanFactory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

        // User-defined BeanPostProcessors registered as beans
        Map<String, BeanPostProcessor> postProcessorBeans =
                this.beanFactory.getBeansOfType(BeanPostProcessor.class);
        for (BeanPostProcessor bp : postProcessorBeans.values()) {
            this.beanFactory.addBeanPostProcessor(bp);
        }
    }

    /**
     * Step 3: Eagerly instantiate all singleton beans.
     *
     * <p>Up until this point, bean creation was lazy (on first {@code getBean()}).
     * This step forces creation of all singletons, which triggers the full
     * dependency injection and lifecycle pipeline for every bean.
     *
     * <p>In the real Spring Framework, this is
     * {@code AbstractApplicationContext.finishBeanFactoryInitialization()},
     * which calls {@code beanFactory.preInstantiateSingletons()}.
     */
    private void finishBeanFactoryInitialization() {
        this.beanFactory.preInstantiateSingletons();
    }

    /**
     * Step 4: Discover and register {@code ApplicationListener} beans.
     *
     * <p>Finds all beans implementing {@code ApplicationListener} and adds
     * them to the listener set. In the real Spring Framework, this is
     * {@code AbstractApplicationContext.registerListeners()}, which also
     * handles programmatically registered listeners and early events.
     */
    @SuppressWarnings({"rawtypes", "unchecked"})
    private void registerListeners() {
        Map<String, ApplicationListener> listenerBeans =
                this.beanFactory.getBeansOfType(ApplicationListener.class);
        for (ApplicationListener listener : listenerBeans.values()) {
            this.applicationListeners.add(listener);
        }
    }

    /**
     * Step 5: Publish the {@code ContextRefreshedEvent}.
     *
     * <p>This signals to all listeners that the context is fully initialized.
     * In the real Spring Framework, this is the last line of
     * {@code AbstractApplicationContext.finishRefresh()}.
     */
    private void finishRefresh() {
        publishEvent(new ContextRefreshedEvent(this));
    }

    @Override
    public void close() {
        if (this.closed.compareAndSet(false, true)) {
            // 1. Publish ContextClosedEvent while beans are still alive
            publishEvent(new ContextClosedEvent(this));

            // 2. Destroy all singletons
            this.beanFactory.close();

            // 3. Mark as inactive
            this.active.set(false);

            // 4. Clear listeners
            this.applicationListeners.clear();
        }
    }

    @Override
    public boolean isActive() {
        return this.active.get();
    }

    // -----------------------------------------------------------------------
    // ApplicationEventPublisher
    // -----------------------------------------------------------------------

    /**
     * Publish the given event to all listeners whose type parameter matches.
     *
     * <p>Uses {@link #resolveEventType(ApplicationListener)} to determine
     * the event type each listener accepts, then only invokes listeners
     * whose type is assignable from the actual event.
     *
     * <p>In the real Spring Framework, this dispatching is done by
     * {@code SimpleApplicationEventMulticaster.multicastEvent()}, which
     * caches listener resolution per event type. We do a simple linear
     * scan for clarity.
     */
    @Override
    public void publishEvent(ApplicationEvent event) {
        for (ApplicationListener<?> listener : this.applicationListeners) {
            Class<?> eventType = resolveEventType(listener);
            if (eventType == null || eventType.isInstance(event)) {
                invokeListener(listener, event);
            }
        }
    }

    @SuppressWarnings({"rawtypes", "unchecked"})
    private void invokeListener(ApplicationListener listener, ApplicationEvent event) {
        listener.onApplicationEvent(event);
    }

    /**
     * Resolve the event type parameter {@code E} from an {@code ApplicationListener<E>}.
     *
     * <p>Walks the class hierarchy and implemented interfaces to find the
     * concrete type argument. Returns {@code null} if the type cannot be
     * resolved (in which case the listener receives all events).
     *
     * <p>In the real Spring Framework, this is handled by
     * {@code GenericApplicationListenerAdapter} using {@code ResolvableType}.
     * We use direct reflection on generic interfaces.
     */
    static Class<?> resolveEventType(ApplicationListener<?> listener) {
        // Check all generic interfaces on the listener class and its superclasses
        Class<?> targetClass = listener.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Type genericInterface : targetClass.getGenericInterfaces()) {
                if (genericInterface instanceof ParameterizedType pt) {
                    if (ApplicationListener.class.isAssignableFrom((Class<?>) pt.getRawType())) {
                        Type typeArg = pt.getActualTypeArguments()[0];
                        if (typeArg instanceof Class<?> clazz) {
                            return clazz;
                        }
                    }
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return null;
    }

    // -----------------------------------------------------------------------
    // BeanFactory — delegation
    // -----------------------------------------------------------------------

    @Override
    public Object getBean(String name) {
        assertActive();
        return this.beanFactory.getBean(name);
    }

    @Override
    public <T> T getBean(String name, Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(name, requiredType);
    }

    @Override
    public <T> T getBean(Class<T> requiredType) {
        assertActive();
        return this.beanFactory.getBean(requiredType);
    }

    @Override
    public boolean containsBean(String name) {
        return this.beanFactory.containsBean(name);
    }

    @Override
    public boolean isSingleton(String name) {
        return this.beanFactory.isSingleton(name);
    }

    // -----------------------------------------------------------------------
    // ApplicationContext
    // -----------------------------------------------------------------------

    @Override
    public String getDisplayName() {
        return getClass().getSimpleName() + "@" + Integer.toHexString(hashCode());
    }

    @Override
    public Environment getEnvironment() {
        return this.environment;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    /**
     * Return the underlying {@code DefaultBeanFactory}.
     *
     * <p>In the real Spring Framework, {@code GenericApplicationContext.getBeanFactory()}
     * returns a {@code ConfigurableListableBeanFactory}. We return the concrete type
     * since we have no intermediate abstraction.
     */
    public DefaultBeanFactory getBeanFactory() {
        return this.beanFactory;
    }

    // -----------------------------------------------------------------------
    // Listener management
    // -----------------------------------------------------------------------

    /**
     * Add a programmatic listener. Must be called before {@code refresh()}.
     *
     * <p>In the real Spring Framework, this is
     * {@code ConfigurableApplicationContext.addApplicationListener()}.
     */
    @Override
    public void addApplicationListener(ApplicationListener<?> listener) {
        this.applicationListeners.add(listener);
    }

    /**
     * Set a {@link DeferredConfigurationLoader} for loading additional
     * configuration classes after user configs are processed.
     *
     * <p>This is the integration point for auto-configuration (Feature 13):
     * the boot layer sets a loader that reads auto-configuration candidates
     * from {@code META-INF/iris/AutoConfiguration.imports}.
     *
     * <p>Must be called before {@link #refresh()}.
     *
     * @param loader the deferred configuration loader
     * @see DeferredConfigurationLoader
     */
    public void setDeferredConfigurationLoader(DeferredConfigurationLoader loader) {
        this.deferredConfigurationLoader = loader;
    }

    // -----------------------------------------------------------------------
    // Internal
    // -----------------------------------------------------------------------

    private void assertActive() {
        if (!this.active.get()) {
            throw new IllegalStateException(
                    "ApplicationContext has not been refreshed yet or has already been closed");
        }
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java` [MODIFIED]

```java
package com.iris.framework.context.annotation;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.annotation.AnnotationUtils;
import com.iris.framework.core.env.Environment;

/**
 * Processes {@link Configuration @Configuration} classes registered in the
 * container — performing component scanning and discovering {@link Bean @Bean}
 * methods, then registering {@link BeanDefinition}s for each discovery.
 *
 * <p>Processing happens in two phases:
 * <ol>
 *   <li><b>Phase 1 — Component Scanning:</b> For each {@code @Configuration}
 *       class with {@link ComponentScan @ComponentScan}, delegate to a
 *       {@link ClassPathScanner} to discover and register
 *       {@link com.iris.framework.stereotype.Component @Component}-annotated
 *       classes on the classpath.</li>
 *   <li><b>Phase 2 — @Bean Method Processing:</b> For each
 *       {@code @Configuration} class (including those discovered by scanning),
 *       find {@code @Bean} methods and register factory-method bean definitions.</li>
 * </ol>
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code ConfigurationClassPostProcessor}, which in the real framework:
 * <ol>
 *   <li>Implements {@code BeanDefinitionRegistryPostProcessor}</li>
 *   <li>Uses {@code ConfigurationClassParser} to recursively discover
 *       {@code @Configuration}, {@code @ComponentScan}, {@code @Import}, etc.</li>
 *   <li>Uses {@code ConfigurationClassBeanDefinitionReader} to register beans</li>
 *   <li>Applies CGLIB enhancement for full {@code @Configuration} classes</li>
 * </ol>
 *
 * <p>Configuration detection uses {@link AnnotationUtils#hasAnnotation} to
 * support meta-annotated configuration classes — e.g., {@code @AutoConfiguration}
 * is meta-annotated with {@code @Configuration} and is recognized as a
 * configuration class without needing {@code @Configuration} directly.
 *
 * <p>Our simplification:
 * <ul>
 *   <li>Single-pass scanning (no iterative re-discovery of nested
 *       {@code @ComponentScan} on scanned {@code @Configuration} classes)</li>
 *   <li>No CGLIB enhancement, no {@code @Import}</li>
 * </ul>
 *
 * @see Configuration
 * @see Bean
 * @see ComponentScan
 * @see ClassPathScanner
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public class ConfigurationClassProcessor {

    /**
     * Process all {@code @Configuration} classes currently registered in the
     * given registry — without condition evaluation.
     *
     * <p>This overload preserves backward compatibility with code that does
     * not use conditional beans (e.g., unit tests from earlier features).
     * Delegates to the full method with a {@code null} environment.
     *
     * @param registry the bean definition registry to scan and populate
     */
    public void processConfigurationClasses(BeanDefinitionRegistry registry) {
        processConfigurationClasses(registry, null);
    }

    /**
     * Process all {@code @Configuration} classes currently registered in the
     * given registry. First performs component scanning (Phase 1), then
     * discovers and registers {@code @Bean} methods (Phase 2), evaluating
     * {@link Conditional @Conditional} annotations at both class and method
     * levels.
     *
     * <p>This maps to Spring's
     * {@code ConfigurationClassPostProcessor.processConfigBeanDefinitions()}:
     * iterate registered bean definitions, identify configuration candidates,
     * parse {@code @ComponentScan} and {@code @Bean}, and register new
     * bean definitions.
     *
     * <p>The two-phase design is deliberate: component scanning (Phase 1) may
     * discover new {@code @Configuration} classes whose {@code @Bean} methods
     * must also be processed. Phase 2 re-snapshots the bean definition names
     * to include everything discovered by scanning.
     *
     * <p><b>Condition evaluation (Feature 12):</b> Before processing a
     * {@code @Configuration} class's {@code @Bean} methods, class-level
     * conditions are checked — if they fail, the entire class is skipped.
     * Then for each {@code @Bean} method, method-level conditions are
     * checked — if they fail, that individual bean is skipped.
     *
     * <p>In the real Spring Framework, condition evaluation happens in two
     * phases: {@code PARSE_CONFIGURATION} (class-level, in
     * {@code ConfigurationClassParser}) and {@code REGISTER_BEAN}
     * (method-level, in {@code ConfigurationClassBeanDefinitionReader}).
     * We collapse both into this single method for simplicity.
     *
     * @param registry    the bean definition registry to scan and populate
     * @param environment the environment for property-based conditions
     *                    (may be {@code null} if conditions are not needed)
     */
    public void processConfigurationClasses(BeanDefinitionRegistry registry,
                                            Environment environment) {
        ConditionEvaluator evaluator = new ConditionEvaluator(registry, environment);

        // Phase 1: Component scanning — discover @Component classes on the classpath
        performComponentScanning(registry);

        // Phase 2: @Bean method processing — register factory-method beans
        // Re-snapshot names to include beans discovered by scanning
        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String candidateName : candidateNames) {
            BeanDefinition bd = registry.getBeanDefinition(candidateName);
            Class<?> beanClass = bd.getBeanClass();

            if (!AnnotationUtils.hasAnnotation(beanClass, Configuration.class)) {
                continue;
            }

            // Evaluate class-level conditions (e.g., @ConditionalOnClass on a @Configuration)
            // If conditions don't match, skip the entire class and all its @Bean methods
            if (evaluator.shouldSkip(beanClass)) {
                continue;
            }

            List<Method> beanMethods = findBeanMethods(beanClass);
            for (Method method : beanMethods) {
                // Evaluate method-level conditions (e.g., @ConditionalOnMissingBean on a @Bean)
                // If conditions don't match, skip this individual bean registration
                if (evaluator.shouldSkip(method)) {
                    continue;
                }

                String beanName = resolveBeanName(method);
                BeanDefinition beanDef = createBeanDefinition(method, candidateName);
                registry.registerBeanDefinition(beanName, beanDef);
            }
        }
    }

    /**
     * Phase 1: For each {@code @Configuration} class that has
     * {@link ComponentScan @ComponentScan}, delegate to a
     * {@link ClassPathScanner} to discover and register components.
     *
     * <p>If {@code @ComponentScan} specifies no base packages, the package
     * of the {@code @Configuration} class itself is used as the default —
     * matching the real Spring Framework's behavior.
     *
     * <p>In the real Spring Framework, this is handled by
     * {@code ComponentScanAnnotationParser.parse()} which is called from
     * {@code ConfigurationClassParser.doProcessConfigurationClass()}.
     *
     * @param registry the bean definition registry to scan and populate
     */
    private void performComponentScanning(BeanDefinitionRegistry registry) {
        ClassPathScanner scanner = new ClassPathScanner();
        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String candidateName : candidateNames) {
            BeanDefinition bd = registry.getBeanDefinition(candidateName);
            Class<?> beanClass = bd.getBeanClass();

            if (!AnnotationUtils.hasAnnotation(beanClass, Configuration.class)) {
                continue;
            }

            ComponentScan componentScan = beanClass.getAnnotation(ComponentScan.class);
            if (componentScan == null) {
                continue;
            }

            String[] basePackages = componentScan.value();
            if (basePackages.length == 0) {
                // Default: scan the package of the @Configuration class
                basePackages = new String[]{ beanClass.getPackageName() };
            }

            scanner.scan(registry, basePackages);
        }
    }

    /**
     * Find all methods annotated with {@code @Bean} in the given class,
     * including inherited methods.
     *
     * <p>In the real Spring Framework, {@code ConfigurationClassParser} uses
     * ASM-based metadata reading for deterministic ordering. We use JDK
     * reflection which is sufficient for our simplified version.
     */
    private List<Method> findBeanMethods(Class<?> configClass) {
        List<Method> beanMethods = new ArrayList<>();
        for (Method method : configClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Bean.class)) {
                beanMethods.add(method);
            }
        }
        return beanMethods;
    }

    /**
     * Determine the bean name: use the {@code @Bean(value)} if provided,
     * otherwise fall back to the method name.
     *
     * <p>In the real Spring Framework, {@code @Bean} supports an array of
     * names (name + aliases). We support a single name for simplicity.
     */
    private String resolveBeanName(Method method) {
        Bean beanAnnotation = method.getAnnotation(Bean.class);
        String explicitName = beanAnnotation.value();
        return explicitName.isEmpty() ? method.getName() : explicitName;
    }

    /**
     * Create a {@link BeanDefinition} that records this bean should be created
     * by invoking the given factory method on the named configuration class.
     *
     * <p>The bean's type is the method's return type. The factory method's
     * parameters will be resolved from the container at creation time — just
     * like constructor autowiring.
     *
     * <p>In the real Spring Framework, this is done in
     * {@code ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod()},
     * which sets {@code factoryBeanName}, {@code factoryMethodName}, and
     * {@code AUTOWIRE_CONSTRUCTOR} on a {@code RootBeanDefinition}.
     */
    private BeanDefinition createBeanDefinition(Method method, String configBeanName) {
        BeanDefinition bd = new BeanDefinition(method.getReturnType());
        bd.setFactoryBeanName(configBeanName);
        bd.setFactoryMethod(method);
        return bd;
    }

    /**
     * Derive a default bean name for a {@code @Configuration} class:
     * lowercase the first character of the simple class name.
     *
     * <p>For example, {@code AppConfig} → {@code "appConfig"}.
     *
     * <p>This mirrors Spring's {@code AnnotationBeanNameGenerator} which uses
     * the short class name with a lowercase first letter for {@code @Component}
     * and {@code @Configuration} classes.
     */
    public static String deriveConfigBeanName(Class<?> configClass) {
        String simpleName = configClass.getSimpleName();
        if (simpleName.isEmpty()) {
            return simpleName;
        }
        return Character.toLowerCase(simpleName.charAt(0)) + simpleName.substring(1);
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.context.annotation.Configuration;

/**
 * Indicates that a class is an auto-configuration class — a
 * {@link Configuration @Configuration} that is automatically discovered and
 * applied based on what's on the classpath and what beans are already
 * registered.
 *
 * <p>Auto-configuration classes are listed in
 * {@code META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports}
 * and loaded by {@link AutoConfigurationImportSelector}.
 *
 * <h3>Ordering</h3>
 *
 * <p>The {@link #before()} and {@link #after()} attributes declare ordering
 * constraints relative to other auto-configuration classes. The
 * {@link AutoConfigurationSorter} uses these to perform a topological sort.
 *
 * <p>Example:
 * <pre>{@code
 * @AutoConfiguration(after = DataSourceAutoConfiguration.class)
 * @ConditionalOnClass(name = "org.hibernate.Session")
 * public class HibernateAutoConfiguration {
 *     @Bean
 *     @ConditionalOnMissingBean
 *     public SessionFactory sessionFactory(DataSource ds) { ... }
 * }
 * }</pre>
 *
 * <p>In the real Spring Boot (since 2.7), {@code @AutoConfiguration} replaces
 * the older pattern of {@code @Configuration} +
 * {@code @AutoConfigureBefore/@AutoConfigureAfter}. The real annotation uses
 * {@code @AliasFor} to bridge its {@code before/after} attributes to the
 * separate ordering annotations. We read both directly since we don't have
 * {@code @AliasFor}.
 *
 * @see EnableAutoConfiguration
 * @see AutoConfigurationImportSelector
 * @see AutoConfigurationSorter
 * @see org.springframework.boot.autoconfigure.AutoConfiguration
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface AutoConfiguration {

    /**
     * Auto-configuration classes that should be applied <em>after</em> this
     * configuration — i.e., this configuration must run <em>before</em> them.
     */
    Class<?>[] before() default {};

    /**
     * Class names for {@link #before()} — use when the class may not be on
     * the classpath.
     */
    String[] beforeName() default {};

    /**
     * Auto-configuration classes that should be applied <em>before</em> this
     * configuration — i.e., this configuration must run <em>after</em> them.
     */
    Class<?>[] after() default {};

    /**
     * Class names for {@link #after()} — use when the class may not be on
     * the classpath.
     */
    String[] afterName() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigureBefore.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Hint that an auto-configuration class should be applied <em>before</em>
 * the specified auto-configuration classes.
 *
 * <p>Can be placed directly on an {@code @AutoConfiguration} class. The
 * {@link AutoConfigurationSorter} reads this annotation when performing
 * topological sorting.
 *
 * <p>In the real Spring Boot, this annotation is also bridged via
 * {@code @AliasFor} from {@code @AutoConfiguration(before=...)}.
 *
 * @see AutoConfigureAfter
 * @see AutoConfiguration
 * @see org.springframework.boot.autoconfigure.AutoConfigureBefore
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AutoConfigureBefore {

    /**
     * The auto-configuration classes that this class should be applied before.
     */
    Class<?>[] value() default {};

    /**
     * The fully-qualified names of auto-configuration classes that this class
     * should be applied before.
     */
    String[] name() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigureAfter.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Hint that an auto-configuration class should be applied <em>after</em>
 * the specified auto-configuration classes.
 *
 * <p>Can be placed directly on an {@code @AutoConfiguration} class. The
 * {@link AutoConfigurationSorter} reads this annotation when performing
 * topological sorting.
 *
 * <p>In the real Spring Boot, this annotation is also bridged via
 * {@code @AliasFor} from {@code @AutoConfiguration(after=...)}.
 *
 * @see AutoConfigureBefore
 * @see AutoConfiguration
 * @see org.springframework.boot.autoconfigure.AutoConfigureAfter
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AutoConfigureAfter {

    /**
     * The auto-configuration classes that this class should be applied after.
     */
    Class<?>[] value() default {};

    /**
     * The fully-qualified names of auto-configuration classes that this class
     * should be applied after.
     */
    String[] name() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/EnableAutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Enable auto-configuration of the Iris application context, attempting to
 * guess and configure beans that you are likely to need. Auto-configuration
 * classes are usually applied based on your classpath and what beans you
 * have already defined.
 *
 * <p>When present on the primary source class, the bootstrap layer
 * ({@link com.iris.boot.IrisApplication}) sets up a
 * {@link com.iris.framework.context.annotation.DeferredConfigurationLoader
 * DeferredConfigurationLoader} that discovers auto-configuration candidates
 * from {@code META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports}
 * files on the classpath.
 *
 * <p>Auto-configuration is always applied <em>after</em> user-defined beans
 * have been registered. This is the key invariant: if you define your own
 * {@code DataSource} bean, the auto-configured one will back off.
 *
 * <h3>Excluding auto-configurations</h3>
 *
 * <p>Use {@link #exclude()} or {@link #excludeName()} to prevent specific
 * auto-configuration classes from being applied:
 * <pre>{@code
 * @EnableAutoConfiguration(exclude = TomcatAutoConfiguration.class)
 * }</pre>
 *
 * <p>In the real Spring Boot, this annotation has {@code @Import(
 * AutoConfigurationImportSelector.class)} which triggers auto-config
 * loading via the {@code DeferredImportSelector} mechanism. We use a
 * direct detection approach: the bootstrap layer checks for this annotation
 * and sets up the deferred loader explicitly.
 *
 * @see AutoConfiguration
 * @see AutoConfigurationImportSelector
 * @see org.springframework.boot.autoconfigure.EnableAutoConfiguration
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableAutoConfiguration {

    /**
     * Exclude specific auto-configuration classes by class reference.
     */
    Class<?>[] exclude() default {};

    /**
     * Exclude specific auto-configuration classes by fully-qualified name.
     * Use this when the class may not be on the classpath.
     */
    String[] excludeName() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/context/annotation/ImportCandidates.java` [NEW]

```java
package com.iris.boot.context.annotation;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Enumeration;
import java.util.List;

/**
 * Reads candidate class names from {@code .imports} files on the classpath.
 *
 * <p>The file location follows the pattern:
 * <pre>META-INF/iris/{annotation-fully-qualified-name}.imports</pre>
 *
 * <p>For example, with
 * {@code com.iris.boot.autoconfigure.AutoConfiguration}:
 * <pre>META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports</pre>
 *
 * <h3>File format</h3>
 *
 * <p>Each non-empty line is a fully-qualified class name. Lines starting with
 * {@code #} are comments. Blank lines are ignored.
 *
 * <pre>
 * # Auto-configuration candidates
 * com.iris.boot.autoconfigure.web.TomcatAutoConfiguration
 * com.iris.boot.autoconfigure.jackson.JacksonAutoConfiguration
 * </pre>
 *
 * <p>Multiple files with the same name across different JARs are all loaded
 * and merged — this allows each module to contribute its own auto-configuration
 * candidates.
 *
 * <p>In the real Spring Boot, this is
 * {@code org.springframework.boot.context.annotation.ImportCandidates} which
 * reads from {@code META-INF/spring/{annotation}.imports}. The mechanism
 * replaced the older {@code spring.factories} SPI (removed in Spring Boot 3.x).
 *
 * @see org.springframework.boot.context.annotation.ImportCandidates
 */
public class ImportCandidates {

    /** Pattern for the classpath location of .imports files. */
    private static final String LOCATION_PATTERN = "META-INF/iris/%s.imports";

    private final List<String> candidates;

    private ImportCandidates(List<String> candidates) {
        this.candidates = Collections.unmodifiableList(candidates);
    }

    /**
     * Load import candidates for the given annotation from the classpath.
     *
     * <p>Reads all resources matching the location pattern using
     * {@code classLoader.getResources()}, which aggregates across all JARs.
     *
     * @param annotation  the annotation class whose candidates to load
     * @param classLoader the class loader to search
     * @return an {@code ImportCandidates} instance with all discovered names
     */
    public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
        String location = String.format(LOCATION_PATTERN, annotation.getName());
        List<String> candidates = new ArrayList<>();

        try {
            Enumeration<URL> urls = classLoader.getResources(location);
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                candidates.addAll(readCandidateConfigurations(url));
            }
        } catch (IOException ex) {
            throw new IllegalArgumentException(
                    "Unable to load auto-configuration candidates from '" + location + "'", ex);
        }

        return new ImportCandidates(candidates);
    }

    /**
     * Read candidate class names from a single resource URL.
     *
     * <p>Each non-empty, non-comment line becomes a candidate.
     * Comments start with {@code #} and are stripped. Lines are trimmed.
     */
    private static List<String> readCandidateConfigurations(URL url) throws IOException {
        List<String> candidates = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(url.openStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                // Strip comments
                int commentIdx = line.indexOf('#');
                if (commentIdx >= 0) {
                    line = line.substring(0, commentIdx);
                }
                line = line.trim();
                if (!line.isEmpty()) {
                    candidates.add(line);
                }
            }
        }
        return candidates;
    }

    /**
     * Return the list of candidate class names.
     *
     * @return unmodifiable list of fully-qualified class names
     */
    public List<String> getCandidates() {
        return this.candidates;
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigurationImportSelector.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

import com.iris.boot.context.annotation.ImportCandidates;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.context.annotation.DeferredConfigurationLoader;
import com.iris.framework.core.env.Environment;

/**
 * Loads, filters, and sorts auto-configuration candidates from the classpath.
 *
 * <p>Implements {@link DeferredConfigurationLoader} to integrate with the
 * application context's deferred processing mechanism. When invoked, it:
 *
 * <ol>
 *   <li><b>Loads candidates</b> — reads class names from
 *       {@code META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports}
 *       via {@link ImportCandidates}</li>
 *   <li><b>Removes duplicates</b> — multiple JARs may list the same class</li>
 *   <li><b>Applies exclusions</b> — respects {@code @EnableAutoConfiguration(exclude=...)}
 *       and the {@code iris.autoconfigure.exclude} property</li>
 *   <li><b>Loads classes</b> — uses {@code Class.forName()}; silently skips
 *       classes whose dependencies aren't on the classpath</li>
 *   <li><b>Sorts</b> — delegates to {@link AutoConfigurationSorter} for
 *       alphabetical + topological ordering</li>
 * </ol>
 *
 * <p>In the real Spring Boot, {@code AutoConfigurationImportSelector}:
 * <ul>
 *   <li>Implements {@code DeferredImportSelector} (not just {@code ImportSelector})</li>
 *   <li>Uses a {@code Group} mechanism ({@code AutoConfigurationGroup}) for
 *       batching and sorting across multiple selectors</li>
 *   <li>Applies {@code AutoConfigurationImportFilter} for fast-path filtering
 *       (e.g., {@code OnClassCondition}) before class loading</li>
 *   <li>Fires {@code AutoConfigurationImportEvent} for listeners</li>
 * </ul>
 *
 * <p>We simplify by skipping the Group mechanism, fast-path filtering, and
 * events. Full condition evaluation happens when the
 * {@code ConfigurationClassProcessor} processes the loaded classes.
 *
 * @see EnableAutoConfiguration
 * @see ImportCandidates
 * @see AutoConfigurationSorter
 * @see org.springframework.boot.autoconfigure.AutoConfigurationImportSelector
 */
public class AutoConfigurationImportSelector implements DeferredConfigurationLoader {

    /** Property to exclude auto-configuration classes. */
    private static final String EXCLUDE_PROPERTY = "iris.autoconfigure.exclude";

    private final Set<String> exclusions;

    /**
     * Create a selector with the given exclusions.
     *
     * @param exclusions class names to exclude from auto-configuration
     */
    public AutoConfigurationImportSelector(Set<String> exclusions) {
        this.exclusions = (exclusions != null)
                ? Collections.unmodifiableSet(exclusions)
                : Collections.emptySet();
    }

    /**
     * Create a selector with no exclusions.
     */
    public AutoConfigurationImportSelector() {
        this(Collections.emptySet());
    }

    @Override
    public List<Class<?>> loadConfigurations(BeanDefinitionRegistry registry,
                                              Environment environment) {
        // 1. Load candidate class names from .imports files
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (classLoader == null) {
            classLoader = getClass().getClassLoader();
        }

        ImportCandidates importCandidates =
                ImportCandidates.load(AutoConfiguration.class, classLoader);
        List<String> candidates = new ArrayList<>(importCandidates.getCandidates());

        if (candidates.isEmpty()) {
            return List.of();
        }

        // 2. Remove duplicates (preserve order)
        candidates = new ArrayList<>(new LinkedHashSet<>(candidates));

        // 3. Apply exclusions
        Set<String> allExclusions = collectExclusions(environment);
        candidates.removeAll(allExclusions);

        if (candidates.isEmpty()) {
            return List.of();
        }

        // 4. Load classes (skip those whose dependencies aren't available)
        Map<String, Class<?>> classMap = new LinkedHashMap<>();
        for (String candidate : candidates) {
            try {
                Class<?> clazz = Class.forName(candidate, false, classLoader);
                classMap.put(candidate, clazz);
            } catch (ClassNotFoundException ex) {
                // Class or its dependency not on classpath — skip silently.
                // This is expected: .imports files list all possible auto-configs,
                // but only those with their dependencies present are loaded.
            }
        }

        if (classMap.isEmpty()) {
            return List.of();
        }

        // 5. Sort by ordering constraints
        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classMap);
        List<String> sorted = sorter.getInPriorityOrder(new ArrayList<>(classMap.keySet()));

        // 6. Return loaded classes in sorted order
        List<Class<?>> result = new ArrayList<>(sorted.size());
        for (String name : sorted) {
            Class<?> clazz = classMap.get(name);
            if (clazz != null) {
                result.add(clazz);
            }
        }

        return result;
    }

    /**
     * Collect all exclusions from both the constructor-provided set and the
     * environment property.
     */
    private Set<String> collectExclusions(Environment environment) {
        Set<String> allExclusions = new LinkedHashSet<>(this.exclusions);

        // Check the iris.autoconfigure.exclude property
        if (environment != null) {
            String propertyExclusions = environment.getProperty(EXCLUDE_PROPERTY);
            if (propertyExclusions != null && !propertyExclusions.isBlank()) {
                for (String name : propertyExclusions.split(",")) {
                    String trimmed = name.trim();
                    if (!trimmed.isEmpty()) {
                        allExclusions.add(trimmed);
                    }
                }
            }
        }

        return allExclusions;
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/AutoConfigurationSorter.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * Sorts auto-configuration classes based on ordering constraints declared
 * via {@link AutoConfigureBefore @AutoConfigureBefore},
 * {@link AutoConfigureAfter @AutoConfigureAfter}, and the {@code before}/
 * {@code after} attributes on {@link AutoConfiguration @AutoConfiguration}.
 *
 * <h3>Sorting algorithm (two phases)</h3>
 *
 * <ol>
 *   <li><b>Alphabetical:</b> Sort class names alphabetically as a stable
 *       baseline. This ensures deterministic output when no ordering
 *       constraints exist.</li>
 *   <li><b>Topological sort:</b> Apply ordering constraints. For each class,
 *       determine which classes must come after it (from {@code @AutoConfigureBefore})
 *       and before it (from {@code @AutoConfigureAfter}). Perform a
 *       depth-first topological sort with cycle detection.</li>
 * </ol>
 *
 * <p>In the real Spring Boot, {@code AutoConfigurationSorter} has three phases
 * (alphabetical, {@code @AutoConfigureOrder} numeric priority, then topological).
 * We skip the numeric priority phase for simplicity — the annotation-based
 * before/after ordering is the most commonly used mechanism.
 *
 * @see AutoConfiguration
 * @see AutoConfigureBefore
 * @see AutoConfigureAfter
 * @see org.springframework.boot.autoconfigure.AutoConfigurationSorter
 */
public class AutoConfigurationSorter {

    private final Map<String, Class<?>> classMap;

    /**
     * Create a sorter for the given auto-configuration classes.
     *
     * @param classes the classes to sort (keyed by their fully-qualified name)
     */
    public AutoConfigurationSorter(Map<String, Class<?>> classes) {
        this.classMap = classes;
    }

    /**
     * Return the class names sorted by their ordering constraints.
     *
     * @param classNames the candidate class names (must be present in the
     *                   class map provided at construction)
     * @return a new list with the names in sorted order
     * @throws IllegalStateException if a cycle is detected
     */
    public List<String> getInPriorityOrder(List<String> classNames) {
        // Phase 1: Sort alphabetically (stable baseline)
        List<String> sorted = new ArrayList<>(classNames);
        Collections.sort(sorted);

        // Phase 2: Topological sort by @AutoConfigureBefore / @AutoConfigureAfter
        sorted = sortByAnnotation(sorted);

        return sorted;
    }

    // -----------------------------------------------------------------------
    // Topological sort
    // -----------------------------------------------------------------------

    /**
     * Perform a topological sort based on before/after annotations.
     *
     * <p>Uses a depth-first approach: for each class, recursively process all
     * classes that must come after it, then add the class to the sorted set.
     *
     * @param classNames alphabetically pre-sorted class names
     * @return topologically sorted list
     */
    private List<String> sortByAnnotation(List<String> classNames) {
        Set<String> candidates = new LinkedHashSet<>(classNames);

        // Build the "must come after" adjacency: for each class, which classes
        // should appear after it in the final order?
        Map<String, Set<String>> afterMap = buildAfterMap(candidates);

        // DFS-based topological sort
        LinkedHashSet<String> sorted = new LinkedHashSet<>();
        Set<String> processing = new LinkedHashSet<>();

        for (String className : classNames) {
            if (!sorted.contains(className)) {
                doSortByAfterAnnotation(className, candidates, afterMap,
                        sorted, processing);
            }
        }

        return new ArrayList<>(sorted);
    }

    /**
     * Recursive DFS: add all dependencies first, then the current class.
     *
     * @param current    the class being visited
     * @param candidates all candidate class names
     * @param afterMap   adjacency map: class → classes that must come after it
     * @param sorted     accumulator for the sorted result
     * @param processing cycle detection set (classes currently in the DFS stack)
     */
    private void doSortByAfterAnnotation(String current,
                                          Set<String> candidates,
                                          Map<String, Set<String>> afterMap,
                                          LinkedHashSet<String> sorted,
                                          Set<String> processing) {
        if (sorted.contains(current)) {
            return;
        }
        if (!processing.add(current)) {
            throw new IllegalStateException(
                    "Auto-configuration cycle detected involving: " + current
                            + ". Processing stack: " + processing);
        }

        // Process all classes that this class says it should come after
        Set<String> afterClasses = getClassesRequestedAfter(current, candidates, afterMap);
        for (String after : afterClasses) {
            doSortByAfterAnnotation(after, candidates, afterMap, sorted, processing);
        }

        processing.remove(current);
        sorted.add(current);
    }

    /**
     * For a given class, determine all classes that should come <em>before</em>
     * it in the sorted order.
     *
     * <p>This merges two sources:
     * <ol>
     *   <li>Classes listed in this class's {@code @AutoConfigureAfter} /
     *       {@code @AutoConfiguration(after=...)}: "I should run after X"
     *       → X must come before me</li>
     *   <li>Classes whose {@code @AutoConfigureBefore} /
     *       {@code @AutoConfiguration(before=...)} lists this class:
     *       "X should run before me" → X must come before me</li>
     * </ol>
     *
     * @param className  the current class
     * @param candidates the set of all candidate class names
     * @param afterMap   the pre-built afterMap
     * @return set of class names that must come before this class
     */
    private Set<String> getClassesRequestedAfter(String className,
                                                   Set<String> candidates,
                                                   Map<String, Set<String>> afterMap) {
        Set<String> result = new LinkedHashSet<>();

        // 1. Classes this class says it should run AFTER
        Class<?> clazz = this.classMap.get(className);
        if (clazz != null) {
            result.addAll(getAfterAnnotations(clazz));
        }

        // 2. Classes whose @AutoConfigureBefore lists this class
        //    (inverse: if X says "before className", then className must come after X)
        for (Map.Entry<String, Set<String>> entry : afterMap.entrySet()) {
            if (entry.getValue().contains(className)) {
                result.add(entry.getKey());
            }
        }

        // Only keep candidates that are actually in our set
        result.retainAll(candidates);
        return result;
    }

    /**
     * Build a map: className → set of classes that this class declares should
     * come AFTER it (via {@code @AutoConfigureBefore}).
     *
     * <p>This is the "before" perspective: if class A says "before B", then B
     * appears in A's afterMap entry.
     */
    private Map<String, Set<String>> buildAfterMap(Set<String> candidates) {
        Map<String, Set<String>> afterMap = new HashMap<>();
        for (String className : candidates) {
            Class<?> clazz = this.classMap.get(className);
            if (clazz != null) {
                afterMap.put(className, getBeforeAnnotations(clazz));
            }
        }
        return afterMap;
    }

    // -----------------------------------------------------------------------
    // Annotation reading
    // -----------------------------------------------------------------------

    /**
     * Get the set of class names that this class declares it should run
     * <em>before</em> (i.e., those classes should come AFTER this one).
     */
    private Set<String> getBeforeAnnotations(Class<?> clazz) {
        Set<String> result = new LinkedHashSet<>();

        // From @AutoConfiguration(before=..., beforeName=...)
        AutoConfiguration autoConfig = clazz.getAnnotation(AutoConfiguration.class);
        if (autoConfig != null) {
            for (Class<?> c : autoConfig.before()) {
                result.add(c.getName());
            }
            result.addAll(Arrays.asList(autoConfig.beforeName()));
        }

        // From @AutoConfigureBefore(value=..., name=...)
        AutoConfigureBefore before = clazz.getAnnotation(AutoConfigureBefore.class);
        if (before != null) {
            for (Class<?> c : before.value()) {
                result.add(c.getName());
            }
            result.addAll(Arrays.asList(before.name()));
        }

        return result;
    }

    /**
     * Get the set of class names that this class declares it should run
     * <em>after</em> (i.e., those classes should come BEFORE this one).
     */
    private Set<String> getAfterAnnotations(Class<?> clazz) {
        Set<String> result = new LinkedHashSet<>();

        // From @AutoConfiguration(after=..., afterName=...)
        AutoConfiguration autoConfig = clazz.getAnnotation(AutoConfiguration.class);
        if (autoConfig != null) {
            for (Class<?> c : autoConfig.after()) {
                result.add(c.getName());
            }
            result.addAll(Arrays.asList(autoConfig.afterName()));
        }

        // From @AutoConfigureAfter(value=..., name=...)
        AutoConfigureAfter after = clazz.getAnnotation(AutoConfigureAfter.class);
        if (after != null) {
            for (Class<?> c : after.value()) {
                result.add(c.getName());
            }
            result.addAll(Arrays.asList(after.name()));
        }

        return result;
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java` [MODIFIED]

Only the new/changed methods are shown. The `configureAutoConfiguration()` and `getAutoConfigExclusions()` methods were added, and `configureAutoConfiguration()` is called from the existing `prepareContext()` method after registering the primary source:

```java
/**
 * If the primary source class has {@link EnableAutoConfiguration},
 * set up the {@link AutoConfigurationImportSelector} as a deferred
 * configuration loader on the context.
 */
private void configureAutoConfiguration(AnnotationConfigApplicationContext context) {
    if (!AnnotationUtils.hasAnnotation(this.primarySource, EnableAutoConfiguration.class)) {
        return;
    }

    // Extract exclusions from the annotation
    Set<String> exclusions = getAutoConfigExclusions();

    // Create and set the deferred loader
    context.setDeferredConfigurationLoader(
            new AutoConfigurationImportSelector(exclusions));
}

/**
 * Collect exclusion class names from {@link EnableAutoConfiguration}
 * attributes.
 */
private Set<String> getAutoConfigExclusions() {
    Set<String> exclusions = new LinkedHashSet<>();

    EnableAutoConfiguration annotation =
            this.primarySource.getAnnotation(EnableAutoConfiguration.class);
    if (annotation != null) {
        for (Class<?> excluded : annotation.exclude()) {
            exclusions.add(excluded.getName());
        }
        for (String name : annotation.excludeName()) {
            if (!name.isBlank()) {
                exclusions.add(name.trim());
            }
        }
    }

    return exclusions;
}
```

### Test Code

#### File: `iris-boot-core/src/test/java/com/iris/boot/context/annotation/ImportCandidatesTest.java` [NEW]

```java
package com.iris.boot.context.annotation;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.AutoConfiguration;

/**
 * Tests for {@link ImportCandidates}.
 */
class ImportCandidatesTest {

    @Test
    @DisplayName("should load candidates from .imports file on classpath")
    void shouldLoadCandidates_WhenImportsFileExists() {
        // The test .imports file is at:
        // src/test/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports
        ImportCandidates candidates = ImportCandidates.load(
                AutoConfiguration.class, getClass().getClassLoader());

        List<String> names = candidates.getCandidates();
        assertThat(names).isNotEmpty();
        // Verify specific entries from the test .imports file
        assertThat(names).contains(
                "com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration",
                "com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration");
    }

    @Test
    @DisplayName("should return empty list when no .imports file found")
    void shouldReturnEmptyList_WhenNoImportsFileFound() {
        // Use an annotation class that has no corresponding .imports file
        ImportCandidates candidates = ImportCandidates.load(
                Override.class, getClass().getClassLoader());

        assertThat(candidates.getCandidates()).isEmpty();
    }

    @Test
    @DisplayName("should skip comment lines and blank lines")
    void shouldSkipCommentsAndBlankLines() {
        // The test .imports file contains comments (# ...) and blank lines
        ImportCandidates candidates = ImportCandidates.load(
                AutoConfiguration.class, getClass().getClassLoader());

        List<String> names = candidates.getCandidates();
        // No entry should be a comment or blank
        for (String name : names) {
            assertThat(name).doesNotStartWith("#");
            assertThat(name).isNotBlank();
        }
    }

    @Test
    @DisplayName("should return unmodifiable list")
    void shouldReturnUnmodifiableList() {
        ImportCandidates candidates = ImportCandidates.load(
                AutoConfiguration.class, getClass().getClassLoader());

        org.junit.jupiter.api.Assertions.assertThrows(
                UnsupportedOperationException.class,
                () -> candidates.getCandidates().add("com.example.Foo"));
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/AutoConfigurationSorterTest.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

/**
 * Tests for {@link AutoConfigurationSorter}.
 */
class AutoConfigurationSorterTest {

    // -----------------------------------------------------------------------
    // Test auto-configuration classes with ordering
    // -----------------------------------------------------------------------

    @AutoConfiguration
    static class AlphaConfig {}

    @AutoConfiguration
    static class BetaConfig {}

    @AutoConfiguration
    static class GammaConfig {}

    @AutoConfiguration(after = AlphaConfig.class)
    static class DependsOnAlphaConfig {}

    @AutoConfiguration(before = GammaConfig.class)
    static class BeforeGammaConfig {}

    @AutoConfiguration(after = AlphaConfig.class, before = GammaConfig.class)
    static class MiddleConfig {}

    @AutoConfigureAfter(AlphaConfig.class)
    @AutoConfiguration
    static class DirectAfterAlphaConfig {}

    @AutoConfigureBefore(GammaConfig.class)
    @AutoConfiguration
    static class DirectBeforeGammaConfig {}

    // Cycle: A after B, B after A
    @AutoConfiguration(after = CycleB.class)
    static class CycleA {}

    @AutoConfiguration(after = CycleA.class)
    static class CycleB {}

    // -----------------------------------------------------------------------
    // Tests
    // -----------------------------------------------------------------------

    @Test
    @DisplayName("should sort alphabetically when no ordering constraints")
    void shouldSortAlphabetically_WhenNoOrderingConstraints() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(GammaConfig.class.getName(), GammaConfig.class);
        classes.put(AlphaConfig.class.getName(), AlphaConfig.class);
        classes.put(BetaConfig.class.getName(), BetaConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(List.of(
                GammaConfig.class.getName(),
                AlphaConfig.class.getName(),
                BetaConfig.class.getName()));

        assertThat(sorted).containsExactly(
                AlphaConfig.class.getName(),
                BetaConfig.class.getName(),
                GammaConfig.class.getName());
    }

    @Test
    @DisplayName("should respect @AutoConfiguration(after=...) ordering")
    void shouldRespectAfterOrdering() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(DependsOnAlphaConfig.class.getName(), DependsOnAlphaConfig.class);
        classes.put(AlphaConfig.class.getName(), AlphaConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(List.of(
                DependsOnAlphaConfig.class.getName(),
                AlphaConfig.class.getName()));

        assertThat(sorted.indexOf(AlphaConfig.class.getName()))
                .isLessThan(sorted.indexOf(DependsOnAlphaConfig.class.getName()));
    }

    @Test
    @DisplayName("should respect @AutoConfiguration(before=...) ordering")
    void shouldRespectBeforeOrdering() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(GammaConfig.class.getName(), GammaConfig.class);
        classes.put(BeforeGammaConfig.class.getName(), BeforeGammaConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(List.of(
                GammaConfig.class.getName(),
                BeforeGammaConfig.class.getName()));

        assertThat(sorted.indexOf(BeforeGammaConfig.class.getName()))
                .isLessThan(sorted.indexOf(GammaConfig.class.getName()));
    }

    @Test
    @DisplayName("should respect combined before and after ordering")
    void shouldRespectCombinedOrdering() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(GammaConfig.class.getName(), GammaConfig.class);
        classes.put(AlphaConfig.class.getName(), AlphaConfig.class);
        classes.put(MiddleConfig.class.getName(), MiddleConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(List.of(
                GammaConfig.class.getName(),
                AlphaConfig.class.getName(),
                MiddleConfig.class.getName()));

        int alphaIdx = sorted.indexOf(AlphaConfig.class.getName());
        int middleIdx = sorted.indexOf(MiddleConfig.class.getName());
        int gammaIdx = sorted.indexOf(GammaConfig.class.getName());

        assertThat(alphaIdx).isLessThan(middleIdx);
        assertThat(middleIdx).isLessThan(gammaIdx);
    }

    @Test
    @DisplayName("should respect @AutoConfigureAfter annotation directly on class")
    void shouldRespectDirectAutoConfigureAfter() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(DirectAfterAlphaConfig.class.getName(), DirectAfterAlphaConfig.class);
        classes.put(AlphaConfig.class.getName(), AlphaConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(List.of(
                DirectAfterAlphaConfig.class.getName(),
                AlphaConfig.class.getName()));

        assertThat(sorted.indexOf(AlphaConfig.class.getName()))
                .isLessThan(sorted.indexOf(DirectAfterAlphaConfig.class.getName()));
    }

    @Test
    @DisplayName("should respect @AutoConfigureBefore annotation directly on class")
    void shouldRespectDirectAutoConfigureBefore() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(GammaConfig.class.getName(), GammaConfig.class);
        classes.put(DirectBeforeGammaConfig.class.getName(), DirectBeforeGammaConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(List.of(
                GammaConfig.class.getName(),
                DirectBeforeGammaConfig.class.getName()));

        assertThat(sorted.indexOf(DirectBeforeGammaConfig.class.getName()))
                .isLessThan(sorted.indexOf(GammaConfig.class.getName()));
    }

    @Test
    @DisplayName("should detect cycle and throw exception")
    void shouldDetectCycle_AndThrowException() {
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(CycleA.class.getName(), CycleA.class);
        classes.put(CycleB.class.getName(), CycleB.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        assertThatThrownBy(() -> sorter.getInPriorityOrder(List.of(
                CycleA.class.getName(), CycleB.class.getName())))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("cycle");
    }

    @Test
    @DisplayName("should handle empty input")
    void shouldHandleEmptyInput() {
        AutoConfigurationSorter sorter = new AutoConfigurationSorter(Map.of());
        List<String> sorted = sorter.getInPriorityOrder(List.of());
        assertThat(sorted).isEmpty();
    }

    @Test
    @DisplayName("should ignore ordering references to classes not in candidate set")
    void shouldIgnoreReferencesToNonCandidates() {
        // DependsOnAlphaConfig says "after Alpha", but Alpha is not in the set
        Map<String, Class<?>> classes = new LinkedHashMap<>();
        classes.put(DependsOnAlphaConfig.class.getName(), DependsOnAlphaConfig.class);

        AutoConfigurationSorter sorter = new AutoConfigurationSorter(classes);
        List<String> sorted = sorter.getInPriorityOrder(
                List.of(DependsOnAlphaConfig.class.getName()));

        assertThat(sorted).containsExactly(DependsOnAlphaConfig.class.getName());
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/AutoConfigurationImportSelectorTest.java` [NEW]

```java
package com.iris.boot.autoconfigure;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.Set;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration;
import com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration;
import com.iris.boot.autoconfigure.integration.TestNonExistentLibAutoConfiguration;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.StandardEnvironment;

/**
 * Tests for {@link AutoConfigurationImportSelector}.
 */
class AutoConfigurationImportSelectorTest {

    @Test
    @DisplayName("should load all auto-configuration candidates from .imports file")
    void shouldLoadAllCandidates() {
        AutoConfigurationImportSelector selector = new AutoConfigurationImportSelector();
        DefaultBeanFactory registry = new DefaultBeanFactory();

        List<Class<?>> configs = selector.loadConfigurations(registry, new StandardEnvironment());

        assertThat(configs).isNotEmpty();
        assertThat(configs).contains(
                TestJacksonAutoConfiguration.class,
                TestDataSourceAutoConfiguration.class,
                TestNonExistentLibAutoConfiguration.class);
    }

    @Test
    @DisplayName("should exclude classes specified in constructor")
    void shouldExcludeSpecifiedClasses() {
        Set<String> exclusions = Set.of(
                TestNonExistentLibAutoConfiguration.class.getName());
        AutoConfigurationImportSelector selector =
                new AutoConfigurationImportSelector(exclusions);
        DefaultBeanFactory registry = new DefaultBeanFactory();

        List<Class<?>> configs = selector.loadConfigurations(registry, new StandardEnvironment());

        assertThat(configs).doesNotContain(TestNonExistentLibAutoConfiguration.class);
        assertThat(configs).contains(TestJacksonAutoConfiguration.class);
    }

    @Test
    @DisplayName("should exclude classes specified via property")
    void shouldExcludeViaProperty() {
        AutoConfigurationImportSelector selector = new AutoConfigurationImportSelector();
        DefaultBeanFactory registry = new DefaultBeanFactory();

        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test", Map.of(
                "iris.autoconfigure.exclude",
                TestNonExistentLibAutoConfiguration.class.getName())));

        List<Class<?>> configs = selector.loadConfigurations(registry, env);

        assertThat(configs).doesNotContain(TestNonExistentLibAutoConfiguration.class);
    }

    @Test
    @DisplayName("should sort by ordering constraints - DataSource after Jackson")
    void shouldSortByOrderingConstraints() {
        AutoConfigurationImportSelector selector = new AutoConfigurationImportSelector();
        DefaultBeanFactory registry = new DefaultBeanFactory();

        List<Class<?>> configs = selector.loadConfigurations(registry, new StandardEnvironment());

        int jacksonIdx = configs.indexOf(TestJacksonAutoConfiguration.class);
        int dataSourceIdx = configs.indexOf(TestDataSourceAutoConfiguration.class);

        // TestDataSourceAutoConfiguration has @AutoConfigureAfter(TestJacksonAutoConfiguration)
        assertThat(jacksonIdx).isGreaterThanOrEqualTo(0);
        assertThat(dataSourceIdx).isGreaterThanOrEqualTo(0);
        assertThat(jacksonIdx).isLessThan(dataSourceIdx);
    }

    @Test
    @DisplayName("should return empty list when all candidates excluded")
    void shouldReturnEmptyList_WhenAllExcluded() {
        Set<String> exclusions = Set.of(
                TestJacksonAutoConfiguration.class.getName(),
                TestDataSourceAutoConfiguration.class.getName(),
                TestNonExistentLibAutoConfiguration.class.getName());
        AutoConfigurationImportSelector selector =
                new AutoConfigurationImportSelector(exclusions);
        DefaultBeanFactory registry = new DefaultBeanFactory();

        List<Class<?>> configs = selector.loadConfigurations(registry, new StandardEnvironment());

        assertThat(configs).isEmpty();
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/integration/AutoConfigurationIntegrationTest.java` [NEW]

```java
package com.iris.boot.autoconfigure.integration;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;
import java.util.Set;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.AutoConfigurationImportSelector;
import com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration.DataSourceHolder;
import com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration.ObjectMapperHolder;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.StandardEnvironment;

/**
 * Integration tests for the Auto-Configuration Engine (Feature 13).
 *
 * <p>Tests the full flow: auto-configuration candidates are read from the test
 * {@code .imports} file, sorted and filtered, and processed as deferred
 * {@code @Configuration} classes through {@link AnnotationConfigApplicationContext}.
 *
 * <p>The test .imports file references:
 * <ul>
 *   <li>{@link TestJacksonAutoConfiguration} — provides {@code ObjectMapperHolder} when Jackson is on classpath</li>
 *   <li>{@link TestDataSourceAutoConfiguration} — provides {@code DataSourceHolder}, ordered after Jackson</li>
 *   <li>{@link TestNonExistentLibAutoConfiguration} — guarded by a missing class, always skipped</li>
 * </ul>
 */
class AutoConfigurationIntegrationTest {

    // -----------------------------------------------------------------------
    // User override configurations
    // -----------------------------------------------------------------------

    /**
     * User configuration that provides a custom {@link ObjectMapperHolder},
     * overriding the auto-configured one.
     */
    @Configuration
    static class UserJacksonOverrideConfig {
        @Bean
        public ObjectMapperHolder objectMapperHolder() {
            return new ObjectMapperHolder("user-defined-jackson");
        }
    }

    /**
     * User configuration with a custom {@link DataSourceHolder}.
     */
    @Configuration
    static class UserDataSourceOverrideConfig {
        @Bean
        public DataSourceHolder dataSourceHolder() {
            return new DataSourceHolder("jdbc:mysql://user-defined");
        }
    }

    // -----------------------------------------------------------------------
    // Helper
    // -----------------------------------------------------------------------

    /**
     * Creates a context with the given user configs and enables auto-configuration
     * by setting a deferred loader (simulating what IrisApplication does).
     */
    private AnnotationConfigApplicationContext createAutoConfigContext(
            StandardEnvironment env, Class<?>... userConfigs) {
        var ctx = new AnnotationConfigApplicationContext();
        if (env != null) {
            ctx.setEnvironment(env);
        }
        for (Class<?> cfg : userConfigs) {
            ctx.register(cfg);
        }
        ctx.setDeferredConfigurationLoader(new AutoConfigurationImportSelector());
        ctx.refresh();
        return ctx;
    }

    private AnnotationConfigApplicationContext createAutoConfigContext(Class<?>... userConfigs) {
        return createAutoConfigContext(null, userConfigs);
    }

    // -----------------------------------------------------------------------
    // Integration tests
    // -----------------------------------------------------------------------

    @Test
    @DisplayName("should apply auto-configuration when no user beans defined")
    void shouldApplyAutoConfig_WhenNoUserBeansDefined() {
        var ctx = createAutoConfigContext();

        // TestJacksonAutoConfiguration provides ObjectMapperHolder
        assertThat(ctx.containsBean("objectMapperHolder")).isTrue();
        ObjectMapperHolder holder = ctx.getBean(ObjectMapperHolder.class);
        assertThat(holder.getName()).isEqualTo("auto-configured-jackson");

        // TestDataSourceAutoConfiguration provides DataSourceHolder
        assertThat(ctx.containsBean("dataSourceHolder")).isTrue();
        DataSourceHolder ds = ctx.getBean(DataSourceHolder.class);
        assertThat(ds.getUrl()).isEqualTo("jdbc:h2:mem:auto-configured");

        ctx.close();
    }

    @Test
    @DisplayName("should back off when user provides same bean type")
    void shouldBackOff_WhenUserProvidesSameBeanType() {
        var ctx = createAutoConfigContext(UserJacksonOverrideConfig.class);

        // User's bean should win (ConditionalOnMissingBean backs off)
        ObjectMapperHolder holder = ctx.getBean(ObjectMapperHolder.class);
        assertThat(holder.getName()).isEqualTo("user-defined-jackson");

        // DataSource auto-config should still apply (no user override)
        DataSourceHolder ds = ctx.getBean(DataSourceHolder.class);
        assertThat(ds.getUrl()).isEqualTo("jdbc:h2:mem:auto-configured");

        ctx.close();
    }

    @Test
    @DisplayName("should back off for multiple user overrides")
    void shouldBackOff_WhenMultipleUserOverrides() {
        var ctx = createAutoConfigContext(
                UserJacksonOverrideConfig.class,
                UserDataSourceOverrideConfig.class);

        assertThat(ctx.getBean(ObjectMapperHolder.class).getName())
                .isEqualTo("user-defined-jackson");
        assertThat(ctx.getBean(DataSourceHolder.class).getUrl())
                .isEqualTo("jdbc:mysql://user-defined");

        ctx.close();
    }

    @Test
    @DisplayName("should skip auto-config when class condition fails")
    void shouldSkipAutoConfig_WhenClassConditionFails() {
        var ctx = createAutoConfigContext();

        // TestNonExistentLibAutoConfiguration requires a non-existent class
        assertThat(ctx.containsBean("nonExistentBean")).isFalse();

        ctx.close();
    }

    @Test
    @DisplayName("should skip auto-config when property condition fails")
    void shouldSkipAutoConfig_WhenPropertyConditionFails() {
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test", Map.of(
                "datasource.enabled", "false")));

        var ctx = createAutoConfigContext(env);

        // DataSource auto-config should be skipped due to property
        assertThat(ctx.containsBean("dataSourceHolder")).isFalse();
        // Jackson auto-config should still apply
        assertThat(ctx.containsBean("objectMapperHolder")).isTrue();

        ctx.close();
    }

    @Test
    @DisplayName("should respect exclusions via constructor")
    void shouldRespectExclusions() {
        var ctx = new AnnotationConfigApplicationContext();
        ctx.setDeferredConfigurationLoader(
                new AutoConfigurationImportSelector(Set.of(
                        TestJacksonAutoConfiguration.class.getName())));
        ctx.refresh();

        // Excluded: should not be present
        assertThat(ctx.containsBean("objectMapperHolder")).isFalse();
        // Not excluded: should still be present
        assertThat(ctx.containsBean("dataSourceHolder")).isTrue();

        ctx.close();
    }

    @Test
    @DisplayName("should respect exclusions via property")
    void shouldRespectExclusions_ViaProperty() {
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test", Map.of(
                "iris.autoconfigure.exclude",
                TestJacksonAutoConfiguration.class.getName())));

        var ctx = new AnnotationConfigApplicationContext();
        ctx.setEnvironment(env);
        ctx.setDeferredConfigurationLoader(new AutoConfigurationImportSelector());
        ctx.refresh();

        assertThat(ctx.containsBean("objectMapperHolder")).isFalse();
        assertThat(ctx.containsBean("dataSourceHolder")).isTrue();

        ctx.close();
    }

    @Test
    @DisplayName("should process auto-configs AFTER user configs (deferred loading)")
    void shouldProcessAutoConfigsAfterUserConfigs() {
        // Core test: auto-config's @ConditionalOnMissingBean must see user beans
        // from the first processing pass. If deferred loading is broken, auto-config
        // would NOT see the user bean and would register its own default.

        var ctx = createAutoConfigContext(UserJacksonOverrideConfig.class);

        ObjectMapperHolder holder = ctx.getBean(ObjectMapperHolder.class);
        assertThat(holder.getName())
                .as("Auto-config should back off when user bean exists (deferred processing)")
                .isEqualTo("user-defined-jackson");

        ctx.close();
    }

    @Test
    @DisplayName("should work without deferred loader (no auto-configuration)")
    void shouldWorkWithoutDeferredLoader() {
        // Plain context without auto-configuration — backward compatibility
        var ctx = new AnnotationConfigApplicationContext(UserJacksonOverrideConfig.class);

        assertThat(ctx.getBean(ObjectMapperHolder.class).getName())
                .isEqualTo("user-defined-jackson");
        assertThat(ctx.containsBean("dataSourceHolder")).isFalse();

        ctx.close();
    }

    @Test
    @DisplayName("should respect ordering: DataSource runs after Jackson")
    void shouldRespectOrdering() {
        // TestDataSourceAutoConfiguration has @AutoConfigureAfter(TestJacksonAutoConfiguration)
        // Both should be present, indicating correct ordering
        var ctx = createAutoConfigContext();

        assertThat(ctx.containsBean("objectMapperHolder")).isTrue();
        assertThat(ctx.containsBean("dataSourceHolder")).isTrue();

        ctx.close();
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/integration/TestJacksonAutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure.integration;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnClass;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.framework.context.annotation.Bean;

/**
 * Test auto-configuration that provides a default {@link ObjectMapperHolder}
 * — only when Jackson is on the classpath and no user-defined
 * {@code ObjectMapperHolder} bean exists.
 *
 * <p>This simulates a real auto-configuration pattern:
 * <ol>
 *   <li>Class-level condition: only applies if Jackson is available</li>
 *   <li>Bean-level condition: backs off if the user defines their own bean</li>
 * </ol>
 */
@AutoConfiguration
@ConditionalOnClass(name = "com.fasterxml.jackson.databind.ObjectMapper")
public class TestJacksonAutoConfiguration {

    /**
     * Simple holder type for testing — avoids using String which is too
     * broad for {@code @ConditionalOnMissingBean} type deduction.
     */
    public static class ObjectMapperHolder {
        private final String name;

        public ObjectMapperHolder(String name) {
            this.name = name;
        }

        public String getName() {
            return name;
        }
    }

    @Bean
    @ConditionalOnMissingBean
    public ObjectMapperHolder objectMapperHolder() {
        return new ObjectMapperHolder("auto-configured-jackson");
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/integration/TestDataSourceAutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure.integration;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.AutoConfigureAfter;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.condition.ConditionalOnProperty;
import com.iris.framework.context.annotation.Bean;

/**
 * Test auto-configuration for a simulated DataSource — ordered to run
 * AFTER {@link TestJacksonAutoConfiguration} and gated by a property.
 *
 * <p>Demonstrates:
 * <ul>
 *   <li>Ordering via {@code @AutoConfigureAfter}</li>
 *   <li>Property-based gating</li>
 *   <li>{@code @ConditionalOnMissingBean} backing off</li>
 * </ul>
 */
@AutoConfiguration
@AutoConfigureAfter(TestJacksonAutoConfiguration.class)
@ConditionalOnProperty(name = "datasource.enabled", matchIfMissing = true)
public class TestDataSourceAutoConfiguration {

    /**
     * Simple holder type for testing — avoids using String which is too
     * broad for {@code @ConditionalOnMissingBean} type deduction.
     */
    public static class DataSourceHolder {
        private final String url;

        public DataSourceHolder(String url) {
            this.url = url;
        }

        public String getUrl() {
            return url;
        }
    }

    @Bean
    @ConditionalOnMissingBean
    public DataSourceHolder dataSourceHolder() {
        return new DataSourceHolder("jdbc:h2:mem:auto-configured");
    }
}
```

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/integration/TestNonExistentLibAutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure.integration;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnClass;
import com.iris.framework.context.annotation.Bean;

/**
 * Test auto-configuration guarded by a missing class.
 * Should always be skipped since the class doesn't exist.
 */
@AutoConfiguration
@ConditionalOnClass(name = "com.example.NonExistentLibrary")
public class TestNonExistentLibAutoConfiguration {

    @Bean
    public String nonExistentBean() {
        return "should-never-exist";
    }
}
```

#### File: `iris-boot-core/src/test/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports` [NEW]

```
# Test auto-configuration candidates
# Used by ImportCandidatesTest and AutoConfigurationIntegrationTest

com.iris.boot.autoconfigure.integration.TestJacksonAutoConfiguration
com.iris.boot.autoconfigure.integration.TestDataSourceAutoConfiguration
com.iris.boot.autoconfigure.integration.TestNonExistentLibAutoConfiguration
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`DeferredConfigurationLoader`** | A `@FunctionalInterface` callback invoked after user `@Configuration` classes are processed -- returns additional config classes for a second processing pass |
| **Two-pass processing** | `invokeBeanFactoryPostProcessors()` runs `ConfigurationClassProcessor` twice: first for user configs, then for deferred auto-configs -- guaranteeing that `@ConditionalOnMissingBean` can see user beans |
| **`@AutoConfiguration`** | Meta-annotated with `@Configuration` -- marks a class as an auto-configuration candidate with optional `before`/`after` ordering attributes |
| **`@EnableAutoConfiguration`** | Trigger annotation -- when detected on the primary source, the bootstrap layer wires `AutoConfigurationImportSelector` as the deferred loader |
| **`ImportCandidates`** | Reads `META-INF/iris/{annotation}.imports` files from the classpath -- the SPI mechanism for discovering auto-configuration candidates across JARs |
| **`AutoConfigurationImportSelector`** | The six-step pipeline: load candidates from `.imports` files, deduplicate, apply exclusions, load classes, sort, and return |
| **`AutoConfigurationSorter`** | Two-phase sort: alphabetical baseline + DFS topological sort using `@AutoConfigureBefore`/`@AutoConfigureAfter` with cycle detection |
| **`AnnotationUtils.hasAnnotation()`** in `ConfigurationClassProcessor` | Enables meta-annotated `@AutoConfiguration` classes to be recognized as `@Configuration` -- without this, auto-config classes are silently ignored |

**Next: Chapter 14 -- Starter Dependencies** -- Package auto-configuration classes, conditional beans, and transitive library dependencies into a single "starter" artifact that users add as a single Gradle/Maven dependency, enabling the "just add the starter" experience
