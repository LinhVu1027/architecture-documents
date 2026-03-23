# Chapter 7: Environment & Properties

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The `ApplicationContext` orchestrates bean lifecycle via `refresh()`, but has no concept of external configuration — all bean wiring is hard-coded in `@Configuration` classes and `@Autowired` injection | Cannot externalize values like server ports, feature flags, or connection strings; changing any configuration requires recompiling code | Build an `Environment` abstraction that loads properties from multiple sources (system properties, environment variables, `application.properties`), resolves `${...}` placeholders, and injects values into beans via `@Value` |

---

## 7.1 The Integration Point

The integration point is `AnnotationConfigApplicationContext` — specifically, two changes: (1) it gains an `Environment` field that is created before `refresh()`, and (2) it registers a `ValueAnnotationBeanPostProcessor` alongside the existing `AutowiredBeanPostProcessor`.

**Modifying:** `src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java`
**Change:** Create a `StandardEnvironment` in the constructor (loading `application.properties`), register `ValueAnnotationBeanPostProcessor` in `registerBeanPostProcessors()`, and add `getEnvironment()`.

```java
// New fields
private final StandardEnvironment environment;

// Constructor — environment created BEFORE refresh()
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this.beanFactory = new DefaultBeanFactory();
    this.environment = createEnvironment();  // ← NEW

    for (Class<?> componentClass : componentClasses) {
        String beanName = ConfigurationClassProcessor.deriveConfigBeanName(componentClass);
        beanFactory.registerBeanDefinition(beanName, new BeanDefinition(componentClass));
    }

    refresh();
}

private StandardEnvironment createEnvironment() {
    StandardEnvironment env = new StandardEnvironment();

    // Load application.properties if present on the classpath
    PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
    MapPropertySource appProps = loader.load("applicationProperties", "application.properties");
    if (appProps != null) {
        env.getPropertySources().addLast(appProps);
    }
    return env;
}

// In registerBeanPostProcessors() — add ValueAnnotationBeanPostProcessor
private void registerBeanPostProcessors() {
    this.beanFactory.addBeanPostProcessor(new AutowiredBeanPostProcessor(this.beanFactory));
    this.beanFactory.addBeanPostProcessor(new ValueAnnotationBeanPostProcessor(this.environment)); // ← NEW
    this.beanFactory.addBeanPostProcessor(new LifecycleBeanPostProcessor());
    // ... user-defined processors
}

// New method — exposes the Environment
@Override
public Environment getEnvironment() {
    return this.environment;
}
```

**Modifying:** `src/main/java/com/iris/framework/context/ApplicationContext.java`
**Change:** Add `getEnvironment()` method to expose the `Environment`.

```java
public interface ApplicationContext extends BeanFactory, ApplicationEventPublisher {
    String getDisplayName();
    Environment getEnvironment();  // ← NEW
}
```

Two key decisions here:

1. **Environment is created BEFORE `refresh()`** — properties must be available when beans are instantiated (during Step 3 of `refresh()`). If we created it later, `@Value` fields would have no property sources to resolve against.

2. **`ValueAnnotationBeanPostProcessor` is registered between `AutowiredBeanPostProcessor` and `LifecycleBeanPostProcessor`** — this gives us the ordering: `@Autowired` injection → `@Value` injection → `@PostConstruct` callbacks. A bean's dependencies are wired first, then its configuration values are set, then it can initialize.

This connects **external configuration** to **the bean lifecycle pipeline**. To make it work, we need to build:
- `PropertySource` / `MapPropertySource` — the data model for named key-value sources
- `MutablePropertySources` — an ordered collection with precedence semantics
- `PropertyResolver` / `Environment` — the resolution interface hierarchy
- `StandardEnvironment` — the default implementation with system properties and system environment
- `PropertySourcesPropertyResolver` — iterates sources and resolves `${...}` placeholders
- `PropertiesPropertySourceLoader` — loads `.properties` files from the classpath
- `@Value` / `ValueAnnotationBeanPostProcessor` — annotation-driven value injection

## 7.2 The Property Source Data Model

A `PropertySource` is a named source of key-value pairs. The name serves as its identity — not the content.

**New file:** `src/main/java/com/iris/framework/core/env/PropertySource.java`

```java
package com.iris.framework.core.env;

public abstract class PropertySource<T> {

    private final String name;
    private final T source;

    public PropertySource(String name, T source) {
        if (name == null) {
            throw new IllegalArgumentException("Property source name must not be null");
        }
        this.name = name;
        this.source = source;
    }

    public String getName() { return this.name; }
    public T getSource() { return this.source; }
    public abstract Object getProperty(String name);

    public boolean containsProperty(String name) {
        return getProperty(name) != null;
    }

    // Identity is based solely on name, not content
    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof PropertySource<?> that)) return false;
        return this.name.equals(that.name);
    }

    @Override
    public int hashCode() { return this.name.hashCode(); }
}
```

**New file:** `src/main/java/com/iris/framework/core/env/MapPropertySource.java`

```java
package com.iris.framework.core.env;

import java.util.Map;

public class MapPropertySource extends PropertySource<Map<String, Object>> {

    public MapPropertySource(String name, Map<String, Object> source) {
        super(name, source);
    }

    @Override
    public Object getProperty(String name) {
        return getSource().get(name);
    }

    @Override
    public boolean containsProperty(String name) {
        return getSource().containsKey(name);
    }
}
```

**New file:** `src/main/java/com/iris/framework/core/env/MutablePropertySources.java`

```java
package com.iris.framework.core.env;

import java.util.Iterator;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class MutablePropertySources implements Iterable<PropertySource<?>> {

    private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();

    public void addFirst(PropertySource<?> propertySource) {
        removeIfPresent(propertySource);
        this.propertySourceList.add(0, propertySource);
    }

    public void addLast(PropertySource<?> propertySource) {
        removeIfPresent(propertySource);
        this.propertySourceList.add(propertySource);
    }

    public PropertySource<?> get(String name) { /* ... */ }
    public boolean contains(String name) { return get(name) != null; }
    public PropertySource<?> remove(String name) { /* ... */ }
    public int size() { return this.propertySourceList.size(); }

    @Override
    public Iterator<PropertySource<?>> iterator() {
        return this.propertySourceList.iterator();
    }

    private void removeIfPresent(PropertySource<?> propertySource) {
        this.propertySourceList.remove(propertySource);
    }
}
```

The property source ordering determines **precedence**: the first source in the list has the highest priority. If the same key exists in multiple sources, the first non-null value wins. `CopyOnWriteArrayList` provides thread safety — reads (during property lookups) are lock-free, writes (adding/removing sources) synchronize on the list.

## 7.3 The Environment and Property Resolution

**New file:** `src/main/java/com/iris/framework/core/env/PropertyResolver.java`

```java
package com.iris.framework.core.env;

public interface PropertyResolver {
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    <T> T getProperty(String key, Class<T> targetType);
    String getRequiredProperty(String key);
    boolean containsProperty(String key);
    String resolvePlaceholders(String text);
    String resolveRequiredPlaceholders(String text);
}
```

**New file:** `src/main/java/com/iris/framework/core/env/Environment.java`

```java
package com.iris.framework.core.env;

public interface Environment extends PropertyResolver {
    MutablePropertySources getPropertySources();
}
```

**New file:** `src/main/java/com/iris/framework/core/env/PropertySourcesPropertyResolver.java`

The heart of the feature — iterates property sources and resolves `${...}` placeholders:

```java
package com.iris.framework.core.env;

public class PropertySourcesPropertyResolver implements PropertyResolver {

    private static final String PLACEHOLDER_PREFIX = "${";
    private static final String PLACEHOLDER_SUFFIX = "}";
    private static final String VALUE_SEPARATOR = ":";

    private final MutablePropertySources propertySources;

    public PropertySourcesPropertyResolver(MutablePropertySources propertySources) {
        this.propertySources = propertySources;
    }

    @Override
    public String getProperty(String key) {
        for (PropertySource<?> propertySource : this.propertySources) {
            Object value = propertySource.getProperty(key);
            if (value != null) {
                return String.valueOf(value);
            }
        }
        return null;
    }

    // Placeholder resolution engine
    private String doResolvePlaceholders(String text, boolean failOnUnresolvable) {
        if (text == null || !text.contains(PLACEHOLDER_PREFIX)) {
            return text;
        }

        StringBuilder result = new StringBuilder();
        int i = 0;

        while (i < text.length()) {
            int start = text.indexOf(PLACEHOLDER_PREFIX, i);
            if (start == -1) {
                result.append(text, i, text.length());
                break;
            }
            result.append(text, i, start);
            int end = text.indexOf(PLACEHOLDER_SUFFIX, start + 2);
            if (end == -1) {
                result.append(text, start, text.length());
                break;
            }

            String placeholder = text.substring(start + 2, end);
            String key;
            String defaultValue = null;
            int separatorIdx = placeholder.indexOf(VALUE_SEPARATOR);
            if (separatorIdx != -1) {
                key = placeholder.substring(0, separatorIdx);
                defaultValue = placeholder.substring(separatorIdx + 1);
            } else {
                key = placeholder;
            }

            String resolved = getProperty(key);
            if (resolved != null) {
                result.append(resolved);
            } else if (defaultValue != null) {
                result.append(defaultValue);
            } else if (failOnUnresolvable) {
                throw new IllegalArgumentException(
                    "Could not resolve placeholder '" + key + "' in value \"" + text + "\"");
            } else {
                result.append("${").append(placeholder).append('}');
            }
            i = end + 1;
        }
        return result.toString();
    }
}
```

**New file:** `src/main/java/com/iris/framework/core/env/StandardEnvironment.java`

```java
package com.iris.framework.core.env;

import java.util.Map;

public class StandardEnvironment implements Environment {

    public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
    public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

    private final MutablePropertySources propertySources;
    private final PropertySourcesPropertyResolver propertyResolver;

    public StandardEnvironment() {
        this.propertySources = new MutablePropertySources();
        customizePropertySources(this.propertySources);
        this.propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);
    }

    @SuppressWarnings({"unchecked", "rawtypes"})
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new MapPropertySource(
                SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, (Map) System.getProperties()));
        propertySources.addLast(new MapPropertySource(
                SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, (Map) System.getenv()));
    }

    // All PropertyResolver methods delegate to propertyResolver
    @Override
    public String getProperty(String key) {
        return this.propertyResolver.getProperty(key);
    }
    // ... (remaining delegations)
}
```

**New file:** `src/main/java/com/iris/framework/core/env/PropertiesPropertySourceLoader.java`

```java
package com.iris.framework.core.env;

import java.io.*;
import java.util.*;

public class PropertiesPropertySourceLoader {

    public MapPropertySource load(String name, String resource) {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (classLoader == null) {
            classLoader = getClass().getClassLoader();
        }

        try (InputStream is = classLoader.getResourceAsStream(resource)) {
            if (is == null) {
                return null;  // Resource not found — silently skip
            }
            Properties properties = new Properties();
            properties.load(is);

            Map<String, Object> map = new LinkedHashMap<>();
            for (String key : properties.stringPropertyNames()) {
                map.put(key, properties.getProperty(key));
            }
            return new MapPropertySource(name, map);
        } catch (IOException ex) {
            throw new RuntimeException("Failed to load properties from '" + resource + "'", ex);
        }
    }
}
```

## 7.4 The @Value Annotation and Injection

**New file:** `src/main/java/com/iris/framework/beans/factory/annotation/Value.java`

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {
    String value();
}
```

**New file:** `src/main/java/com/iris/framework/beans/factory/annotation/ValueAnnotationBeanPostProcessor.java`

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.reflect.Field;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.PropertySourcesPropertyResolver;

public class ValueAnnotationBeanPostProcessor implements BeanPostProcessor {

    private final Environment environment;

    public ValueAnnotationBeanPostProcessor(Environment environment) {
        this.environment = environment;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        Class<?> targetClass = bean.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Field field : targetClass.getDeclaredFields()) {
                if (field.isAnnotationPresent(Value.class)) {
                    injectValue(bean, beanName, field);
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return bean;
    }

    private void injectValue(Object bean, String beanName, Field field) {
        Value valueAnnotation = field.getAnnotation(Value.class);
        String expression = valueAnnotation.value();

        try {
            String resolved = environment.resolveRequiredPlaceholders(expression);
            Object convertedValue = PropertySourcesPropertyResolver.convertValue(
                    resolved, field.getType());
            field.setAccessible(true);
            field.set(bean, convertedValue);
        } catch (IllegalArgumentException ex) {
            throw new BeanCreationException(beanName,
                    "Failed to resolve @Value(\"" + expression + "\") on field '"
                            + field.getName() + "': " + ex.getMessage(), ex);
        } catch (IllegalAccessException ex) {
            throw new BeanCreationException(beanName,
                    "Failed to inject @Value field '" + field.getName() + "'", ex);
        }
    }
}
```

## 7.5 Try It Yourself

<details>
<summary>Challenge: Implement the placeholder resolution algorithm</summary>

Given a string like `"App ${app.name} on port ${server.port:8080}"`, write a method that:
1. Finds each `${...}` placeholder
2. Splits on `:` to separate the key from the default value
3. Looks up the key in the property sources (first non-null wins)
4. Uses the default value if the key is not found
5. Replaces the placeholder with the resolved value

Think about edge cases: What if the `}` is missing? What if there's no default and the key doesn't exist?

```java
private String doResolvePlaceholders(String text, boolean failOnUnresolvable) {
    if (text == null || !text.contains("${")) {
        return text;
    }

    StringBuilder result = new StringBuilder();
    int i = 0;

    while (i < text.length()) {
        int start = text.indexOf("${", i);
        if (start == -1) {
            result.append(text, i, text.length());
            break;
        }
        result.append(text, i, start);
        int end = text.indexOf("}", start + 2);
        if (end == -1) {
            result.append(text, start, text.length());
            break;
        }

        String placeholder = text.substring(start + 2, end);
        String key;
        String defaultValue = null;
        int sep = placeholder.indexOf(':');
        if (sep != -1) {
            key = placeholder.substring(0, sep);
            defaultValue = placeholder.substring(sep + 1);
        } else {
            key = placeholder;
        }

        String resolved = getProperty(key);
        if (resolved != null) {
            result.append(resolved);
        } else if (defaultValue != null) {
            result.append(defaultValue);
        } else if (failOnUnresolvable) {
            throw new IllegalArgumentException(
                "Could not resolve placeholder '" + key + "'");
        } else {
            result.append("${").append(placeholder).append('}');
        }
        i = end + 1;
    }
    return result.toString();
}
```

</details>

<details>
<summary>Challenge: Implement the ValueAnnotationBeanPostProcessor</summary>

Given access to an `Environment`, write a `BeanPostProcessor` that:
1. Scans bean fields for `@Value` annotations
2. Resolves the annotation's expression via `environment.resolveRequiredPlaceholders()`
3. Converts the string result to the field's type (int, boolean, double, etc.)
4. Injects the value via reflection

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    Class<?> targetClass = bean.getClass();
    while (targetClass != null && targetClass != Object.class) {
        for (Field field : targetClass.getDeclaredFields()) {
            if (field.isAnnotationPresent(Value.class)) {
                String expression = field.getAnnotation(Value.class).value();
                String resolved = environment.resolveRequiredPlaceholders(expression);
                Object converted = PropertySourcesPropertyResolver.convertValue(
                        resolved, field.getType());
                field.setAccessible(true);
                field.set(bean, converted);
            }
        }
        targetClass = targetClass.getSuperclass();
    }
    return bean;
}
```

</details>

## 7.6 Tests

### Unit Tests

**New file:** `src/test/java/com/iris/framework/core/env/PropertySourceTest.java`

```java
@Test
void shouldReturnPropertyValue_WhenKeyExists() {
    MapPropertySource source = new MapPropertySource("test", Map.of("key1", "value1"));
    assertThat(source.getProperty("key1")).isEqualTo("value1");
}

@Test
void shouldBeEqualByName_WhenNameMatches() {
    MapPropertySource source1 = new MapPropertySource("same", Map.of("a", "1"));
    MapPropertySource source2 = new MapPropertySource("same", Map.of("b", "2"));
    assertThat(source1).isEqualTo(source2);
}
```

**New file:** `src/test/java/com/iris/framework/core/env/PropertySourcesPropertyResolverTest.java`

```java
@Test
void shouldRespectPrecedence_WhenSameKeyInMultipleSources() {
    // "app.name" exists in both sources — first source wins
    assertThat(resolver.getProperty("app.name")).isEqualTo("Iris");
}

@Test
void shouldResolveMultiplePlaceholders_WhenTextContainsMultiple() {
    String result = resolver.resolvePlaceholders("${app.name} on port ${server.port}");
    assertThat(result).isEqualTo("Iris on port 8080");
}

@Test
void shouldUseDefaultValue_WhenPlaceholderNotResolvable() {
    String result = resolver.resolvePlaceholders("${missing.key:defaultVal}");
    assertThat(result).isEqualTo("defaultVal");
}
```

**New file:** `src/test/java/com/iris/framework/beans/factory/annotation/ValueAnnotationBeanPostProcessorTest.java`

```java
@Test
void shouldInjectIntValue_WhenPropertyExists() {
    IntValueBean bean = new IntValueBean();
    processor.postProcessBeforeInitialization(bean, "intBean");
    assertThat(bean.getPort()).isEqualTo(8080);
}

@Test
void shouldUseDefaultValue_WhenPropertyMissing() {
    DefaultValueBean bean = new DefaultValueBean();
    processor.postProcessBeforeInitialization(bean, "defaultBean");
    assertThat(bean.getValue()).isEqualTo("defaultValue");
}

@Test
void shouldThrowException_WhenRequiredPropertyMissing() {
    MissingPropertyBean bean = new MissingPropertyBean();
    assertThatThrownBy(() ->
            processor.postProcessBeforeInitialization(bean, "missingBean"))
            .isInstanceOf(BeanCreationException.class)
            .hasMessageContaining("nonexistent.property");
}
```

### Integration Tests

**New file:** `src/test/java/com/iris/framework/core/env/integration/EnvironmentIntegrationTest.java`

```java
@Test
void shouldInjectPropertiesFromApplicationProperties_WhenContextCreated() {
    var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    ServerSettings settings = ctx.getBean(ServerSettings.class);
    assertThat(settings.getPort()).isEqualTo(9090);
    assertThat(settings.getGreeting()).isEqualTo("Hello from properties!");
    ctx.close();
}

@Test
void shouldGiveSystemPropertiesHigherPrecedence_ThanApplicationProperties() {
    System.setProperty("app.name", "SystemOverride");
    try {
        var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        AppInfo info = ctx.getBean(AppInfo.class);
        assertThat(info.getName()).isEqualTo("SystemOverride");
        ctx.close();
    } finally {
        System.clearProperty("app.name");
    }
}

@Test
void shouldStillSupportAutowiredInjection_WhenValueAnnotationAlsoUsed() {
    var ctx = new AnnotationConfigApplicationContext(MixedConfig.class);
    MixedService service = ctx.getBean(MixedService.class);
    assertThat(service.getMessage()).isEqualTo("Iris Test App: Hello from repo");
    ctx.close();
}
```

**Run:** `./gradlew :iris-framework:test` — expected: all 220 tests pass (including prior features' tests)

---

## 7.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Precedence via iteration order is the core mechanism.** The entire property resolution system reduces to one algorithm: iterate an ordered list, return the first non-null match. System properties override environment variables override application.properties — not because of complex priority logic, but because of their position in a list. This makes the behavior easy to reason about: to change precedence, just reorder the list. To add a new source, insert it at the right position.
> - **Trade-off:** This linear scan means every property lookup iterates all sources until a match is found. For applications with many property sources (Spring Boot can have 17+), this adds overhead per lookup. The real Spring Framework mitigates this with caching in `PropertySourcesPropertyResolver` and by resolving most properties at startup, not at runtime.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Composition over inheritance is why `StandardEnvironment` delegates.** In the real Spring Framework, `AbstractEnvironment` does NOT extend `AbstractPropertyResolver`. It holds a `PropertySourcesPropertyResolver` field and delegates all `PropertyResolver` methods to it. This is because Java's single inheritance would force a choice between the environment hierarchy (`AbstractEnvironment → StandardEnvironment → StandardServletEnvironment`) and the resolver hierarchy (`AbstractPropertyResolver → PropertySourcesPropertyResolver`). Delegation lets you have both hierarchies independently.
> - **When to prefer inheritance:** If the two hierarchies will always evolve together and there's no need for multiple resolver implementations per environment type, inheritance would be simpler. The delegation pattern pays off when you need flexibility — e.g., swapping in a mock resolver for testing.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **`@Value` injection happens between `@Autowired` and `@PostConstruct`.** The BeanPostProcessor ordering is: `AutowiredBeanPostProcessor` → `ValueAnnotationBeanPostProcessor` → `LifecycleBeanPostProcessor`. This means by the time `@PostConstruct` runs, both dependencies AND configuration values are available. A bean's init method can safely use both `@Autowired` collaborators and `@Value` properties.
> - **Real-world implication:** In the real Spring Framework, `@Value` is processed by the SAME `AutowiredAnnotationBeanPostProcessor` that handles `@Autowired`. This is more efficient (one reflection pass per bean instead of two) but conflates two concerns. Our separation makes the concepts clearer for learning.
> -----------------------------------------------------------

## 7.8 What We Enhanced

| Aspect | Before (ch01-ch06) | Current (ch07) | Real Framework |
|--------|---------------------|----------------|----------------|
| External configuration | None — all values hard-coded in Java | `application.properties` loaded from classpath, `@Value("${key}")` injection | 17+ property sources including command-line args, YAML, config server (`AbstractEnvironment.java:152`) |
| Property sources | None | `MutablePropertySources` with system properties, system env, application.properties | `MutablePropertySources` with `addBefore/addAfter/replace/precedenceOf` (`MutablePropertySources.java:48`) |
| Placeholder resolution | None | `${key}` and `${key:default}` via `PropertySourcesPropertyResolver` | Full `PropertyPlaceholderHelper` with nested placeholders, escape characters, configurable syntax (`PropertyPlaceholderHelper.java:84`) |
| Type conversion | None | Basic: String, int, long, boolean, double | Full `ConversionService` with 200+ converters, custom converter registration (`DefaultConversionService.java:52`) |
| ApplicationContext | `BeanFactory` + event publishing | Adds `getEnvironment()` for property access | Extends `EnvironmentCapable` via separate interface (`EnvironmentCapable.java:30`) |
| BeanPostProcessor count | 2 (Autowired + Lifecycle) | 3 (+ ValueAnnotation) | `@Value` handled within `AutowiredAnnotationBeanPostProcessor` (`AutowiredAnnotationBeanPostProcessor.java:285`) |

## 7.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `PropertySource<T>` | `PropertySource<T>` | `PropertySource.java:48` | Real has inner `StubPropertySource` and `ComparisonPropertySource` for collection lookups |
| `MapPropertySource` | `MapPropertySource` + `SystemEnvironmentPropertySource` | `MapPropertySource.java:28` | Real has `SystemEnvironmentPropertySource` that maps `SPRING_PROFILES_ACTIVE` → `spring.profiles.active` |
| `MutablePropertySources` | `MutablePropertySources` | `MutablePropertySources.java:46` | Real adds `addBefore()`, `addAfter()`, `replace()`, `precedenceOf()` |
| `PropertyResolver` | `PropertyResolver` + `ConfigurablePropertyResolver` | `PropertyResolver.java:36` | Real splits into read-only and configurable interfaces (placeholder syntax, required props) |
| `Environment` | `Environment` + `ConfigurableEnvironment` | `Environment.java:42` | Real adds profile management (`getActiveProfiles()`, `acceptsProfiles()`) |
| `StandardEnvironment` | `AbstractEnvironment` → `StandardEnvironment` | `StandardEnvironment.java:56` | Real uses template method `customizePropertySources()` for subclass extension |
| `PropertySourcesPropertyResolver` | `AbstractPropertyResolver` → `PropertySourcesPropertyResolver` | `PropertySourcesPropertyResolver.java:38` | Real delegates parsing to `PropertyPlaceholderHelper`, supports nested placeholders and escape chars |
| `PropertiesPropertySourceLoader` | `PropertiesPropertySourceLoader` | Spring Boot `PropertiesPropertySourceLoader.java:38` | Real supports origin tracking (file:line per property) and multi-document files (`#---`) |
| `@Value` | `@Value` | `Value.java:44` | Real also supports SpEL expressions (`#{...}`) |
| `ValueAnnotationBeanPostProcessor` | `AutowiredAnnotationBeanPostProcessor` | `AutowiredAnnotationBeanPostProcessor.java:285` | Real handles `@Value` within the same processor as `@Autowired`, using embedded value resolvers |

## 7.10 Complete Code

### Production Code

#### File: `src/main/java/com/iris/framework/core/env/PropertySource.java` [NEW]

```java
package com.iris.framework.core.env;

/**
 * Abstract base representing a source of name/value property pairs.
 *
 * <p>Each {@code PropertySource} has a unique {@link #getName() name} and can
 * look up a property value by key. The name serves as the identity — two
 * property sources are considered equal if they have the same name, regardless
 * of content. This enables collection-based manipulation in
 * {@link MutablePropertySources} (find/replace/remove by name).
 *
 * <p>The generic type {@code T} represents the underlying source object
 * (e.g., {@code Map<String, Object>}, {@code java.util.Properties}, or a
 * system-level configuration object).
 *
 * <p>In the real Spring Framework, {@code PropertySource} also provides inner
 * classes {@code StubPropertySource} (placeholder) and {@code ComparisonPropertySource}
 * (for collection lookups). We omit these for simplicity.
 *
 * @param <T> the type of the underlying source object
 * @see MutablePropertySources
 * @see org.springframework.core.env.PropertySource
 */
public abstract class PropertySource<T> {

    private final String name;
    private final T source;

    public PropertySource(String name, T source) {
        if (name == null) {
            throw new IllegalArgumentException("Property source name must not be null");
        }
        this.name = name;
        this.source = source;
    }

    /**
     * Return the name of this property source.
     */
    public String getName() {
        return this.name;
    }

    /**
     * Return the underlying source object.
     */
    public T getSource() {
        return this.source;
    }

    /**
     * Return the value associated with the given property name, or {@code null}
     * if not found.
     *
     * @param name the property name to find
     * @return the property value, or {@code null}
     */
    public abstract Object getProperty(String name);

    /**
     * Return whether this property source contains the given name.
     * Default delegates to {@code getProperty(name) != null}.
     */
    public boolean containsProperty(String name) {
        return getProperty(name) != null;
    }

    /**
     * Identity is based solely on {@code name}, not content.
     * This enables set/list-based manipulation in {@link MutablePropertySources}.
     */
    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof PropertySource<?> that)) return false;
        return this.name.equals(that.name);
    }

    @Override
    public int hashCode() {
        return this.name.hashCode();
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + " {name='" + this.name + "'}";
    }
}
```

#### File: `src/main/java/com/iris/framework/core/env/MapPropertySource.java` [NEW]

```java
package com.iris.framework.core.env;

import java.util.Map;

/**
 * {@link PropertySource} backed by a {@code Map<String, Object>}.
 *
 * <p>This is the most common concrete property source, used for system properties,
 * loaded {@code application.properties} files, and programmatic property maps.
 *
 * <p>In the real Spring Framework, there are several Map-based property source
 * subtypes: {@code MapPropertySource}, {@code PropertiesPropertySource} (wraps
 * {@code java.util.Properties}), and {@code SystemEnvironmentPropertySource}
 * (handles environment variable naming conventions like {@code SPRING_PROFILES_ACTIVE}
 * mapping to {@code spring.profiles.active}). We use a single class for all cases.
 *
 * @see PropertySource
 * @see org.springframework.core.env.MapPropertySource
 */
public class MapPropertySource extends PropertySource<Map<String, Object>> {

    public MapPropertySource(String name, Map<String, Object> source) {
        super(name, source);
    }

    @Override
    public Object getProperty(String name) {
        return getSource().get(name);
    }

    @Override
    public boolean containsProperty(String name) {
        return getSource().containsKey(name);
    }
}
```

#### File: `src/main/java/com/iris/framework/core/env/MutablePropertySources.java` [NEW]

```java
package com.iris.framework.core.env;

import java.util.Iterator;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Mutable, ordered collection of {@link PropertySource} objects.
 *
 * <p>Property source ordering determines <strong>precedence</strong>: sources
 * added first (via {@link #addFirst}) have the highest priority — if the same
 * key exists in multiple sources, the first one wins.
 *
 * <p>Uses a {@link CopyOnWriteArrayList} for thread safety: iterations
 * (property lookups) are lock-free, while mutations (add/remove) synchronize
 * on the list. This matches the real Spring implementation.
 *
 * <p>In the real Spring Framework, {@code MutablePropertySources} also provides
 * {@code addBefore()}, {@code addAfter()}, {@code replace()}, and
 * {@code precedenceOf()} methods. We simplify to {@code addFirst()},
 * {@code addLast()}, {@code get()}, {@code contains()}, and {@code remove()}.
 *
 * @see PropertySource
 * @see org.springframework.core.env.MutablePropertySources
 */
public class MutablePropertySources implements Iterable<PropertySource<?>> {

    private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();

    /**
     * Add a property source with the highest precedence (first in the list).
     * If a source with the same name already exists, it is removed first.
     */
    public void addFirst(PropertySource<?> propertySource) {
        removeIfPresent(propertySource);
        this.propertySourceList.add(0, propertySource);
    }

    /**
     * Add a property source with the lowest precedence (last in the list).
     * If a source with the same name already exists, it is removed first.
     */
    public void addLast(PropertySource<?> propertySource) {
        removeIfPresent(propertySource);
        this.propertySourceList.add(propertySource);
    }

    /**
     * Return the property source with the given name, or {@code null} if not found.
     */
    public PropertySource<?> get(String name) {
        for (PropertySource<?> propertySource : this.propertySourceList) {
            if (propertySource.getName().equals(name)) {
                return propertySource;
            }
        }
        return null;
    }

    /**
     * Return whether a property source with the given name is contained.
     */
    public boolean contains(String name) {
        return get(name) != null;
    }

    /**
     * Remove and return the property source with the given name,
     * or {@code null} if not found.
     */
    public PropertySource<?> remove(String name) {
        for (int i = 0; i < this.propertySourceList.size(); i++) {
            if (this.propertySourceList.get(i).getName().equals(name)) {
                return this.propertySourceList.remove(i);
            }
        }
        return null;
    }

    /**
     * Return the number of property sources.
     */
    public int size() {
        return this.propertySourceList.size();
    }

    /**
     * Iterate property sources in precedence order (highest first).
     */
    @Override
    public Iterator<PropertySource<?>> iterator() {
        return this.propertySourceList.iterator();
    }

    private void removeIfPresent(PropertySource<?> propertySource) {
        this.propertySourceList.remove(propertySource);
    }

    @Override
    public String toString() {
        return this.propertySourceList.toString();
    }
}
```

#### File: `src/main/java/com/iris/framework/core/env/PropertyResolver.java` [NEW]

```java
package com.iris.framework.core.env;

/**
 * Interface for resolving properties against any underlying source.
 *
 * <p>This is the read-only contract for property access and placeholder
 * resolution. Consumers that only need to read properties depend on this
 * minimal interface, not the full {@link Environment}.
 *
 * <p>In the real Spring Framework, this interface is the root of the
 * environment hierarchy: {@code PropertyResolver} → {@code Environment}
 * → {@code ConfigurableEnvironment}. There's also a
 * {@code ConfigurablePropertyResolver} sub-interface for changing
 * placeholder syntax, conversion services, and required properties.
 * We skip that layer for simplicity.
 *
 * @see Environment
 * @see org.springframework.core.env.PropertyResolver
 */
public interface PropertyResolver {

    /**
     * Return the property value for the given key, or {@code null} if not found.
     */
    String getProperty(String key);

    /**
     * Return the property value for the given key, or the default if not found.
     */
    String getProperty(String key, String defaultValue);

    /**
     * Return the property value converted to the target type, or {@code null}.
     */
    <T> T getProperty(String key, Class<T> targetType);

    /**
     * Return the property value for the given key, throwing an exception if not found.
     *
     * @throws IllegalStateException if the key cannot be resolved
     */
    String getRequiredProperty(String key);

    /**
     * Return whether the given property key is available.
     */
    boolean containsProperty(String key);

    /**
     * Resolve {@code ${...}} placeholders in the given text, replacing them
     * with property values. Unresolvable placeholders are left unchanged.
     *
     * <p>Supports default values via the {@code :} separator:
     * {@code ${key:default}} resolves to {@code "default"} if {@code key}
     * is not found.
     */
    String resolvePlaceholders(String text);

    /**
     * Resolve {@code ${...}} placeholders in the given text, throwing an
     * exception for any unresolvable placeholder without a default value.
     *
     * @throws IllegalArgumentException if a placeholder cannot be resolved
     */
    String resolveRequiredPlaceholders(String text);
}
```

#### File: `src/main/java/com/iris/framework/core/env/Environment.java` [NEW]

```java
package com.iris.framework.core.env;

/**
 * Interface representing the environment in which the application is running.
 *
 * <p>Extends {@link PropertyResolver} to add access to the underlying
 * {@link MutablePropertySources}. In the real Spring Framework, this
 * interface also provides profile management ({@code getActiveProfiles()},
 * {@code acceptsProfiles()}). We omit profiles — they come in Feature 17
 * (Profiles & ConfigData).
 *
 * <p>The real hierarchy is:
 * <pre>
 * PropertyResolver
 *   → Environment           (adds profiles)
 *     → ConfigurableEnvironment  (adds getPropertySources(), setActiveProfiles())
 * </pre>
 * We collapse to {@code PropertyResolver → Environment} and put
 * {@code getPropertySources()} directly here.
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
     * We promote it here because without profiles, this is the
     * main capability beyond basic property resolution.
     */
    MutablePropertySources getPropertySources();
}
```

#### File: `src/main/java/com/iris/framework/core/env/PropertySourcesPropertyResolver.java` [NEW]

```java
package com.iris.framework.core.env;

/**
 * {@link PropertyResolver} implementation that resolves property values
 * by iterating over a set of {@link PropertySource}s.
 *
 * <p>The core resolution algorithm: iterate the property sources in order
 * and return the first non-null value found. This is the "precedence"
 * mechanism — sources earlier in the list override later ones.
 *
 * <p>Also handles {@code ${...}} placeholder resolution with support for
 * default values ({@code ${key:default}}) and multiple placeholders in a
 * single string ({@code ${host}:${port}}).
 *
 * <p>In the real Spring Framework, placeholder resolution is split across
 * {@code AbstractPropertyResolver} (the placeholder parsing engine) and
 * {@code PropertySourcesPropertyResolver} (the property source iteration).
 * The parsing is delegated to {@code PropertyPlaceholderHelper}. We inline
 * both for simplicity.
 *
 * @see MutablePropertySources
 * @see org.springframework.core.env.PropertySourcesPropertyResolver
 */
public class PropertySourcesPropertyResolver implements PropertyResolver {

    private static final String PLACEHOLDER_PREFIX = "${";
    private static final String PLACEHOLDER_SUFFIX = "}";
    private static final String VALUE_SEPARATOR = ":";

    private final MutablePropertySources propertySources;

    public PropertySourcesPropertyResolver(MutablePropertySources propertySources) {
        this.propertySources = propertySources;
    }

    @Override
    public String getProperty(String key) {
        for (PropertySource<?> propertySource : this.propertySources) {
            Object value = propertySource.getProperty(key);
            if (value != null) {
                return String.valueOf(value);
            }
        }
        return null;
    }

    @Override
    public String getProperty(String key, String defaultValue) {
        String value = getProperty(key);
        return (value != null) ? value : defaultValue;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProperty(String key, Class<T> targetType) {
        String value = getProperty(key);
        if (value == null) {
            return null;
        }
        return (T) convertValue(value, targetType);
    }

    @Override
    public String getRequiredProperty(String key) {
        String value = getProperty(key);
        if (value == null) {
            throw new IllegalStateException("Required property '" + key + "' not found");
        }
        return value;
    }

    @Override
    public boolean containsProperty(String key) {
        for (PropertySource<?> propertySource : this.propertySources) {
            if (propertySource.containsProperty(key)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public String resolvePlaceholders(String text) {
        return doResolvePlaceholders(text, false);
    }

    @Override
    public String resolveRequiredPlaceholders(String text) {
        return doResolvePlaceholders(text, true);
    }

    // -----------------------------------------------------------------------
    // Placeholder resolution engine
    // -----------------------------------------------------------------------

    /**
     * Parse and resolve {@code ${...}} placeholders in the given text.
     *
     * <p>Supports:
     * <ul>
     *   <li>Simple placeholders: {@code ${key}}</li>
     *   <li>Default values: {@code ${key:default}}</li>
     *   <li>Multiple placeholders: {@code ${host}:${port}}</li>
     * </ul>
     *
     * <p>In the real Spring Framework, this parsing is handled by
     * {@code PropertyPlaceholderHelper}, a standalone utility with support
     * for nested placeholders ({@code ${${env}.url}}), escape characters,
     * and configurable prefix/suffix. We implement the essential algorithm
     * inline.
     *
     * @param text the text containing placeholders
     * @param failOnUnresolvable if true, throw on unresolvable placeholders
     * @return the resolved text
     */
    private String doResolvePlaceholders(String text, boolean failOnUnresolvable) {
        if (text == null || !text.contains(PLACEHOLDER_PREFIX)) {
            return text;
        }

        StringBuilder result = new StringBuilder();
        int i = 0;

        while (i < text.length()) {
            int start = text.indexOf(PLACEHOLDER_PREFIX, i);
            if (start == -1) {
                // No more placeholders — append remainder
                result.append(text, i, text.length());
                break;
            }

            // Append text before the placeholder
            result.append(text, i, start);

            // Find the matching closing brace
            int end = text.indexOf(PLACEHOLDER_SUFFIX, start + PLACEHOLDER_PREFIX.length());
            if (end == -1) {
                // Unclosed placeholder — append as-is
                result.append(text, start, text.length());
                break;
            }

            // Extract the placeholder content: "key" or "key:default"
            String placeholder = text.substring(start + PLACEHOLDER_PREFIX.length(), end);

            // Split on the FIRST ':' for default value support
            String key;
            String defaultValue = null;
            int separatorIdx = placeholder.indexOf(VALUE_SEPARATOR);
            if (separatorIdx != -1) {
                key = placeholder.substring(0, separatorIdx);
                defaultValue = placeholder.substring(separatorIdx + VALUE_SEPARATOR.length());
            } else {
                key = placeholder;
            }

            // Resolve the property value
            String resolved = getProperty(key);
            if (resolved != null) {
                result.append(resolved);
            } else if (defaultValue != null) {
                result.append(defaultValue);
            } else if (failOnUnresolvable) {
                throw new IllegalArgumentException(
                        "Could not resolve placeholder '" + key + "' in value \"" + text + "\"");
            } else {
                // Leave unresolvable placeholder as-is
                result.append(PLACEHOLDER_PREFIX).append(placeholder).append(PLACEHOLDER_SUFFIX);
            }

            i = end + PLACEHOLDER_SUFFIX.length();
        }

        return result.toString();
    }

    // -----------------------------------------------------------------------
    // Type conversion
    // -----------------------------------------------------------------------

    /**
     * Convert a string value to the target type.
     *
     * <p>Supports common types: {@code String}, {@code int}/{@code Integer},
     * {@code long}/{@code Long}, {@code boolean}/{@code Boolean},
     * {@code double}/{@code Double}.
     *
     * <p>In the real Spring Framework, type conversion is handled by a full
     * {@code ConversionService} framework with hundreds of converters. We
     * support only the types needed for {@code @Value} injection.
     */
    public static Object convertValue(String value, Class<?> targetType) {
        if (targetType == String.class) {
            return value;
        }
        if (targetType == int.class || targetType == Integer.class) {
            return Integer.parseInt(value);
        }
        if (targetType == long.class || targetType == Long.class) {
            return Long.parseLong(value);
        }
        if (targetType == boolean.class || targetType == Boolean.class) {
            return Boolean.parseBoolean(value);
        }
        if (targetType == double.class || targetType == Double.class) {
            return Double.parseDouble(value);
        }
        throw new IllegalArgumentException(
                "Cannot convert value '" + value + "' to type " + targetType.getName());
    }
}
```

#### File: `src/main/java/com/iris/framework/core/env/StandardEnvironment.java` [NEW]

```java
package com.iris.framework.core.env;

import java.util.Map;

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
 * <p>System properties have higher precedence because they are per-JVM and can
 * be set on a per-process basis, while environment variables are often shared
 * across processes.
 *
 * <p>Delegates all {@link PropertyResolver} methods to a
 * {@link PropertySourcesPropertyResolver} that iterates the property sources.
 * This is the <strong>composition over inheritance</strong> pattern from the
 * real Spring Framework — {@code AbstractEnvironment} holds a
 * {@code PropertySourcesPropertyResolver} field rather than extending it.
 *
 * <p>In the real Spring Framework, the hierarchy is:
 * <pre>
 * AbstractEnvironment (delegates to PropertySourcesPropertyResolver)
 *   → StandardEnvironment (adds systemProperties + systemEnvironment)
 *     → StandardServletEnvironment (adds servletConfig, servletContext, JNDI)
 * </pre>
 * We collapse to a single class.
 *
 * @see Environment
 * @see PropertySourcesPropertyResolver
 * @see org.springframework.core.env.StandardEnvironment
 */
public class StandardEnvironment implements Environment {

    /** Well-known property source name for JVM system properties. */
    public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";

    /** Well-known property source name for OS environment variables. */
    public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

    private final MutablePropertySources propertySources;
    private final PropertySourcesPropertyResolver propertyResolver;

    public StandardEnvironment() {
        this.propertySources = new MutablePropertySources();
        customizePropertySources(this.propertySources);
        this.propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);
    }

    /**
     * Populate the default property sources.
     *
     * <p>In the real Spring Framework, this is a protected template method
     * on {@code AbstractEnvironment} — subclasses override it to add their
     * own sources (e.g., {@code StandardServletEnvironment} adds servlet
     * config/context). We call it directly from the constructor.
     */
    @SuppressWarnings({"unchecked", "rawtypes"})
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new MapPropertySource(
                SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME,
                (Map) System.getProperties()));
        propertySources.addLast(new MapPropertySource(
                SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                (Map) System.getenv()));
    }

    // -----------------------------------------------------------------------
    // Environment
    // -----------------------------------------------------------------------

    @Override
    public MutablePropertySources getPropertySources() {
        return this.propertySources;
    }

    // -----------------------------------------------------------------------
    // PropertyResolver — delegation to PropertySourcesPropertyResolver
    // -----------------------------------------------------------------------

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

    @Override
    public String toString() {
        return getClass().getSimpleName() + " {propertySources=" + this.propertySources + "}";
    }
}
```

#### File: `src/main/java/com/iris/framework/core/env/PropertiesPropertySourceLoader.java` [NEW]

```java
package com.iris.framework.core.env;

import java.io.IOException;
import java.io.InputStream;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Properties;

/**
 * Loads a {@code .properties} file from the classpath and converts it
 * into a {@link MapPropertySource}.
 *
 * <p>Usage:
 * <pre>{@code
 * PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
 * MapPropertySource source = loader.load("applicationProperties", "application.properties");
 * if (source != null) {
 *     environment.getPropertySources().addLast(source);
 * }
 * }</pre>
 *
 * <p>Returns {@code null} if the resource is not found on the classpath,
 * allowing callers to silently skip missing files.
 *
 * <p>In the real Spring Boot, {@code PropertiesPropertySourceLoader} implements
 * the {@code PropertySourceLoader} SPI and returns
 * {@code OriginTrackedMapPropertySource} objects that track which file and line
 * each property came from (useful for error reporting). It also supports
 * multi-document properties files (separated by {@code #---}). We keep it simple:
 * one file → one property source, no origin tracking.
 *
 * @see MapPropertySource
 * @see org.springframework.boot.env.PropertiesPropertySourceLoader
 */
public class PropertiesPropertySourceLoader {

    /**
     * Load a {@code .properties} file from the classpath.
     *
     * @param name the name for the resulting {@code PropertySource}
     * @param resource the classpath resource path (e.g., {@code "application.properties"})
     * @return a {@code MapPropertySource} containing the loaded properties,
     *         or {@code null} if the resource was not found
     */
    public MapPropertySource load(String name, String resource) {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (classLoader == null) {
            classLoader = getClass().getClassLoader();
        }

        try (InputStream is = classLoader.getResourceAsStream(resource)) {
            if (is == null) {
                return null; // Resource not found — silently skip
            }

            Properties properties = new Properties();
            properties.load(is);

            // Convert Properties to Map<String, Object>
            Map<String, Object> map = new LinkedHashMap<>();
            for (String key : properties.stringPropertyNames()) {
                map.put(key, properties.getProperty(key));
            }

            return new MapPropertySource(name, map);
        } catch (IOException ex) {
            throw new RuntimeException("Failed to load properties from '" + resource + "'", ex);
        }
    }
}
```

#### File: `src/main/java/com/iris/framework/beans/factory/annotation/Value.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation for injecting externalized property values into bean fields.
 *
 * <p>The value attribute supports {@code ${...}} placeholder syntax:
 * <pre>{@code
 * @Value("${server.port}")
 * private int port;
 *
 * @Value("${app.name:My Application}")
 * private String appName;
 * }</pre>
 *
 * <p>Placeholders are resolved against the {@code Environment}'s property
 * sources. If the property is not found and no default is provided, the
 * container throws a {@code BeanCreationException} at startup.
 *
 * <p>In the real Spring Framework, {@code @Value} supports two expression
 * syntaxes: property placeholders ({@code ${...}}) and SpEL expressions
 * ({@code #{...}}). We support only property placeholders. The real
 * {@code @Value} is processed by {@code AutowiredAnnotationBeanPostProcessor},
 * which delegates placeholder resolution to the {@code BeanFactory}'s
 * embedded value resolvers. We use a dedicated
 * {@link ValueAnnotationBeanPostProcessor} that resolves directly against
 * the {@code Environment}.
 *
 * @see ValueAnnotationBeanPostProcessor
 * @see org.springframework.beans.factory.annotation.Value
 */
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {

    /**
     * The expression to resolve — typically a property placeholder
     * like {@code "${my.property}"} or {@code "${my.property:default}"}.
     */
    String value();
}
```

#### File: `src/main/java/com/iris/framework/beans/factory/annotation/ValueAnnotationBeanPostProcessor.java` [NEW]

```java
package com.iris.framework.beans.factory.annotation;

import java.lang.reflect.Field;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.PropertySourcesPropertyResolver;

/**
 * {@link BeanPostProcessor} that resolves {@link Value @Value} annotations
 * on bean fields and injects the resolved values.
 *
 * <p>For each bean, this processor scans the class hierarchy for fields
 * annotated with {@code @Value}, resolves the placeholder expression
 * against the {@link Environment}, converts the result to the field's
 * type, and injects it via reflection.
 *
 * <p>Processing happens in {@code postProcessBeforeInitialization()},
 * which runs AFTER {@code @Autowired} injection but BEFORE
 * {@code @PostConstruct} callbacks. This ordering matches the real
 * Spring Framework.
 *
 * <p>In the real Spring Framework, {@code @Value} is processed by
 * {@code AutowiredAnnotationBeanPostProcessor}, which uses the
 * {@code BeanFactory}'s embedded value resolvers (registered by
 * {@code PropertySourcesPlaceholderConfigurer}) for placeholder resolution.
 * We simplify by having a dedicated processor that resolves directly
 * against the {@code Environment}.
 *
 * @see Value
 * @see Environment
 * @see org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor
 */
public class ValueAnnotationBeanPostProcessor implements BeanPostProcessor {

    private final Environment environment;

    public ValueAnnotationBeanPostProcessor(Environment environment) {
        this.environment = environment;
    }

    /**
     * Scan the bean for {@code @Value} fields and inject resolved values.
     * Walks the class hierarchy from the concrete class up to (but not
     * including) {@code Object}.
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        Class<?> targetClass = bean.getClass();
        while (targetClass != null && targetClass != Object.class) {
            for (Field field : targetClass.getDeclaredFields()) {
                if (field.isAnnotationPresent(Value.class)) {
                    injectValue(bean, beanName, field);
                }
            }
            targetClass = targetClass.getSuperclass();
        }
        return bean;
    }

    private void injectValue(Object bean, String beanName, Field field) {
        Value valueAnnotation = field.getAnnotation(Value.class);
        String expression = valueAnnotation.value();

        try {
            // Resolve ${...} placeholders against the Environment
            String resolved = environment.resolveRequiredPlaceholders(expression);

            // Convert to the field's type
            Object convertedValue = PropertySourcesPropertyResolver.convertValue(
                    resolved, field.getType());

            field.setAccessible(true);
            field.set(bean, convertedValue);
        } catch (IllegalArgumentException ex) {
            throw new BeanCreationException(beanName,
                    "Failed to resolve @Value(\"" + expression + "\") on field '"
                            + field.getName() + "': " + ex.getMessage(),
                    ex);
        } catch (IllegalAccessException ex) {
            throw new BeanCreationException(beanName,
                    "Failed to inject @Value field '" + field.getName() + "'",
                    ex);
        }
    }
}
```

#### File: `src/main/java/com/iris/framework/context/ApplicationContext.java` [MODIFIED]

```java
package com.iris.framework.context;

import com.iris.framework.beans.factory.BeanFactory;
import com.iris.framework.core.env.Environment;

/**
 * Central interface for providing configuration to an application.
 *
 * <p>Extends {@link BeanFactory} (bean access), {@link ApplicationEventPublisher}
 * (event dispatching), and provides access to the {@link Environment}.
 * An {@code ApplicationContext} is a read-only, fully initialized container —
 * unlike a bare {@code BeanFactory}, it has already been {@code refresh()}-ed
 * and is ready to serve beans.
 *
 * <p>In the real Spring Framework, {@code ApplicationContext} extends six
 * interfaces:
 * <ul>
 *   <li>{@code ListableBeanFactory} — iterate over all beans</li>
 *   <li>{@code HierarchicalBeanFactory} — parent-child context chains</li>
 *   <li>{@code EnvironmentCapable} — access the {@code Environment}</li>
 *   <li>{@code MessageSource} — internationalized messages</li>
 *   <li>{@code ApplicationEventPublisher} — event dispatching</li>
 *   <li>{@code ResourcePatternResolver} — classpath resource loading</li>
 * </ul>
 *
 * <p>We simplify to {@code BeanFactory} + {@code ApplicationEventPublisher} +
 * {@code getEnvironment()}, which captures the three essential capabilities:
 * bean access, event publishing, and externalized configuration.
 *
 * <p>In the real Spring Framework, environment access is defined on a
 * separate {@code EnvironmentCapable} interface so that framework methods
 * accepting {@code BeanFactory} can do an {@code instanceof EnvironmentCapable}
 * check to access the environment when available, without requiring
 * {@code ApplicationContext} as a parameter type. We put
 * {@link #getEnvironment()} directly on this interface for simplicity.
 *
 * @see ConfigurableApplicationContext
 * @see Environment
 * @see org.springframework.context.ApplicationContext
 * @see org.springframework.core.env.EnvironmentCapable
 */
public interface ApplicationContext extends BeanFactory, ApplicationEventPublisher {

    /**
     * Return a display name for this context.
     */
    String getDisplayName();

    /**
     * Return the {@link Environment} associated with this context.
     *
     * <p>In the real Spring Framework, this method is defined on
     * {@code EnvironmentCapable}, which {@code ApplicationContext} extends.
     * The environment holds property sources and supports placeholder resolution.
     */
    Environment getEnvironment();
}
```

#### File: `src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java` [MODIFIED]

(See full file in the repository -- too long to duplicate here. Key changes: added `environment` field, `createEnvironment()` method, `ValueAnnotationBeanPostProcessor` registration in `registerBeanPostProcessors()`, and `getEnvironment()` implementation.)

### Test Code

#### File: `src/test/resources/application.properties` [NEW]

```properties
app.name=Iris Test App
app.version=1.0.0
server.port=9090
greeting.message=Hello from properties!
feature.enabled=true
max.connections=100
timeout.seconds=30.5
```

#### File: `src/test/java/com/iris/framework/core/env/PropertySourceTest.java` [NEW]

(See full file in the repository -- 10 tests covering property lookup, equality-by-name, null handling, and mutable map behavior.)

#### File: `src/test/java/com/iris/framework/core/env/MutablePropertySourcesTest.java` [NEW]

(See full file in the repository -- 8 tests covering addFirst/addLast ordering, move semantics, get/contains/remove by name.)

#### File: `src/test/java/com/iris/framework/core/env/PropertySourcesPropertyResolverTest.java` [NEW]

(See full file in the repository -- 16 tests covering basic lookup, precedence, type conversion, placeholder resolution with defaults, multiple placeholders, and error handling.)

#### File: `src/test/java/com/iris/framework/core/env/StandardEnvironmentTest.java` [NEW]

(See full file in the repository -- 11 tests covering default property sources, system property resolution, environment variable access, precedence ordering, and custom source addition.)

#### File: `src/test/java/com/iris/framework/core/env/PropertiesPropertySourceLoaderTest.java` [NEW]

(See full file in the repository -- 4 tests covering successful loading, missing file handling, all-properties loading, and missing key within loaded source.)

#### File: `src/test/java/com/iris/framework/beans/factory/annotation/ValueAnnotationBeanPostProcessorTest.java` [NEW]

(See full file in the repository -- 13 tests covering String/int/boolean/double injection, default values, multiple annotated fields, missing property error, and non-annotated field passthrough.)

#### File: `src/test/java/com/iris/framework/core/env/integration/EnvironmentIntegrationTest.java` [NEW]

(See full file in the repository -- 10 tests covering end-to-end application.properties loading, @Value injection via ApplicationContext, system property precedence, mixed @Autowired + @Value usage, and typed property access.)

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`PropertySource`** | A named source of key-value pairs. Identity is by name, not content. Multiple sources are ordered by precedence. |
| **`MutablePropertySources`** | An ordered collection where position = priority. First source with a non-null value wins. |
| **`PropertyResolver`** | The read-only contract: `getProperty()`, `containsProperty()`, `resolvePlaceholders()`. |
| **`Environment`** | Extends `PropertyResolver` with access to the underlying property sources. The bridge between external configuration and the bean lifecycle. |
| **`StandardEnvironment`** | The default implementation: system properties (highest) → system environment → application.properties (lowest). Uses composition -- delegates to `PropertySourcesPropertyResolver`. |
| **`@Value`** | Annotation-driven property injection: `@Value("${key:default}")` on a field → resolved at startup, converted to the field's type. |
| **Precedence** | The core mechanism: iterate an ordered list, return the first non-null match. To change behavior, reorder the list. |

**Next: Chapter 8 -- IrisApplication Bootstrap** -- The `IrisApplication.run()` entry point that detects the web application type, creates the right context, fires lifecycle events, and prints a startup banner. This is where the "just run main()" experience comes from.
