# Chapter 14: Configuration Properties

> Line references based on commit `5922311a95a` of the Spring Boot repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The framework supports `@Value("${key}")` for injecting individual properties into fields | Configuring a component with many related properties requires a `@Value` annotation on every single field — no type safety, no grouping, no IDE completion | Build `@ConfigurationProperties(prefix="...")` that binds an entire properties prefix to a Java POJO, with relaxed binding, nested object support, and correct BPP ordering |

---

## 14.1 The Integration Point

The integration spans two seams — one in each module:

**Seam 1: Bean registration** — `ConfigurationClassProcessor` must detect `@EnableConfigurationProperties(Foo.class)` on `@Configuration` classes and register `Foo` as a bean definition.

**Seam 2: BPP ordering** — `AnnotationConfigApplicationContext.registerBeanPostProcessors()` must position the new `ConfigurationPropertiesBindingPostProcessor` after `@Autowired`/`@Value` injection but before `@PostConstruct` callbacks.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`
**Change:** After processing `@Bean` methods for each `@Configuration` class, call `processEnableConfigurationProperties()` to register listed property classes as beans.

```java
            // Process @EnableConfigurationProperties — register listed classes as beans
            processEnableConfigurationProperties(beanClass, registry);
```

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java`
**Change:** Add `additionalInternalBeanPostProcessors` list and `addInternalBeanPostProcessor()` method. Insert these BPPs between `ValueAnnotationBeanPostProcessor` and `LifecycleBeanPostProcessor` during `registerBeanPostProcessors()`.

```java
private void registerBeanPostProcessors() {
    // Group 1-2: Internal injection processors
    this.beanFactory.addBeanPostProcessor(new AutowiredBeanPostProcessor(this.beanFactory));
    this.beanFactory.addBeanPostProcessor(new ValueAnnotationBeanPostProcessor(this.environment));

    // Group 3: Additional internal BPPs from the boot layer
    for (BeanPostProcessor bp : this.additionalInternalBeanPostProcessors) {
        this.beanFactory.addBeanPostProcessor(bp);
    }

    // Group 4: Lifecycle processor — runs @PostConstruct/@PreDestroy
    this.beanFactory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

    // User-defined BeanPostProcessors registered as beans
    ...
}
```

**Modifying:** `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java`
**Change:** In `prepareContext()`, register the `ConfigurationPropertiesBindingPostProcessor` as an internal BPP.

```java
annotationContext.addInternalBeanPostProcessor(
        new ConfigurationPropertiesBindingPostProcessor(environment));
```

Two key decisions here:

1. **Detection by annotation name, not type** — `ConfigurationClassProcessor` is in `iris-framework` but `@EnableConfigurationProperties` is in `iris-boot-core`. We use `annotation.annotationType().getSimpleName()` to detect it at runtime, avoiding a compile-time dependency. In real Spring Boot, `@Import(EnableConfigurationPropertiesRegistrar.class)` achieves the same decoupling via the `@Import` mechanism.

2. **Explicit BPP insertion point** — Instead of `PriorityOrdered`/`Ordered` interfaces (which Spring uses), we provide `addInternalBeanPostProcessor()` as an explicit slot between value injection and lifecycle callbacks. This guarantees bound values are available in `@PostConstruct`.

This connects **the bootstrap layer** (IrisApplication) to **the bean creation pipeline** (BeanPostProcessors). To make it work, we need to build:
- `@ConfigurationProperties` annotation — marks a POJO for binding
- `@EnableConfigurationProperties` annotation — triggers bean registration
- `ConfigurationPropertiesBinder` — the binding engine (prefix traversal + type conversion)
- `ConfigurationPropertiesBindingPostProcessor` — the BPP that invokes the binder

---

## 14.2 The Annotations

**New file:** `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationProperties.java`

```java
package com.iris.boot.context.properties;

import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {
    String prefix() default "";
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/context/properties/EnableConfigurationProperties.java`

```java
package com.iris.boot.context.properties;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableConfigurationProperties {
    Class<?>[] value() default {};
}
```

---

## 14.3 The Binding Engine

The binder is where the real work happens. Given a prefix like `"server"` and a target POJO, it:

1. Iterates the target's fields
2. For each field, constructs property keys (`server.port`, `server.max-connections`)
3. Resolves from the Environment with relaxed binding (camelCase ↔ kebab-case)
4. Converts the string value to the field's type
5. Recurses for nested objects

**New file:** `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBinder.java`

```java
package com.iris.boot.context.properties;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.List;

import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.PropertySource;
import com.iris.framework.core.env.PropertySourcesPropertyResolver;

public class ConfigurationPropertiesBinder {

    private final Environment environment;

    public ConfigurationPropertiesBinder(Environment environment) {
        this.environment = environment;
    }

    public void bind(Object target, String prefix) {
        bindFields(target, target.getClass(), prefix);
    }

    private void bindFields(Object target, Class<?> clazz, String prefix) {
        if (clazz == null || clazz == Object.class) return;
        for (Field field : clazz.getDeclaredFields()) {
            if (Modifier.isStatic(field.getModifiers())
                    || Modifier.isFinal(field.getModifiers())) continue;
            bindField(target, field, prefix);
        }
        bindFields(target, clazz.getSuperclass(), prefix);
    }

    private void bindField(Object target, Field field, String prefix) {
        String fieldName = field.getName();
        Class<?> fieldType = field.getType();

        if (isSimpleType(fieldType)) {
            String value = resolveProperty(prefix, fieldName);
            if (value != null) {
                Object converted = PropertySourcesPropertyResolver.convertValue(value, fieldType);
                setFieldValue(target, field, converted);
            }
        } else if (fieldType == List.class) {
            String value = resolveProperty(prefix, fieldName);
            if (value != null) {
                List<String> list = Arrays.stream(value.split(","))
                        .map(String::trim).filter(s -> !s.isEmpty()).toList();
                setFieldValue(target, field, list);
            }
        } else {
            // Nested object — recurse with extended prefix
            Object nested = getFieldValue(target, field);
            if (nested == null && hasPropertiesWithPrefix(prefix + "." + fieldName)) {
                try {
                    var ctor = field.getType().getDeclaredConstructor();
                    ctor.setAccessible(true);
                    nested = ctor.newInstance();
                    setFieldValue(target, field, nested);
                } catch (ReflectiveOperationException ex) { return; }
            }
            if (nested != null) {
                bind(nested, prefix + "." + fieldName);
            }
        }
    }

    private String resolveProperty(String prefix, String fieldName) {
        String value = environment.getProperty(prefix + "." + fieldName);
        if (value != null) return value;
        String kebabName = toKebabCase(fieldName);
        if (!kebabName.equals(fieldName)) {
            value = environment.getProperty(prefix + "." + kebabName);
        }
        return value;
    }

    static String toKebabCase(String camelCase) {
        if (camelCase == null || camelCase.isEmpty()) return camelCase;
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < camelCase.length(); i++) {
            char c = camelCase.charAt(i);
            if (Character.isUpperCase(c)) {
                if (i > 0) result.append('-');
                result.append(Character.toLowerCase(c));
            } else {
                result.append(c);
            }
        }
        return result.toString();
    }

    // ... helper methods for field access, type checking, prefix scanning
}
```

---

## 14.4 The BeanPostProcessor

The BPP is the glue — it detects `@ConfigurationProperties` on each bean and triggers binding.

**New file:** `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBindingPostProcessor.java`

```java
package com.iris.boot.context.properties;

import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.core.env.Environment;

public class ConfigurationPropertiesBindingPostProcessor implements BeanPostProcessor {

    private final ConfigurationPropertiesBinder binder;

    public ConfigurationPropertiesBindingPostProcessor(Environment environment) {
        this.binder = new ConfigurationPropertiesBinder(environment);
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        ConfigurationProperties annotation =
                bean.getClass().getAnnotation(ConfigurationProperties.class);
        if (annotation == null || annotation.prefix().isEmpty()) {
            return bean;
        }
        binder.bind(bean, annotation.prefix());
        return bean;
    }
}
```

---

## 14.5 @EnableConfigurationProperties Detection

The `ConfigurationClassProcessor` gains a new method that uses runtime reflection to discover `@EnableConfigurationProperties` by annotation name — keeping `iris-framework` free of compile-time dependencies on `iris-boot-core`.

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`
**Change:** Add `processEnableConfigurationProperties()` method

```java
private void processEnableConfigurationProperties(Class<?> configClass,
                                                   BeanDefinitionRegistry registry) {
    for (Annotation annotation : configClass.getAnnotations()) {
        if (!"EnableConfigurationProperties".equals(
                annotation.annotationType().getSimpleName())) {
            continue;
        }
        try {
            Method valueMethod = annotation.annotationType().getMethod("value");
            Class<?>[] classes = (Class<?>[]) valueMethod.invoke(annotation);
            for (Class<?> propsClass : classes) {
                String beanName = deriveConfigBeanName(propsClass);
                if (!registry.containsBeanDefinition(beanName)) {
                    registry.registerBeanDefinition(beanName,
                            new BeanDefinition(propsClass));
                }
            }
        } catch (ReflectiveOperationException ex) {
            // Annotation doesn't have a value() method — skip
        }
    }
}
```

---

## 14.6 Try It Yourself

<details>
<summary>Challenge: Implement the kebab-case converter</summary>

Given the field name `maxPoolConnections`, write a method that returns `max-pool-connections`. Insert a dash before each uppercase letter and lowercase it.

```java
static String toKebabCase(String camelCase) {
    if (camelCase == null || camelCase.isEmpty()) return camelCase;
    StringBuilder result = new StringBuilder();
    for (int i = 0; i < camelCase.length(); i++) {
        char c = camelCase.charAt(i);
        if (Character.isUpperCase(c)) {
            if (i > 0) result.append('-');
            result.append(Character.toLowerCase(c));
        } else {
            result.append(c);
        }
    }
    return result.toString();
}
```

</details>

<details>
<summary>Challenge: Add nested object auto-creation to the binder</summary>

When binding `app.database.url=jdbc:h2:mem`, the `database` field might be `null`. The binder should check if any properties exist with the prefix `app.database.` and, if so, instantiate the nested object via its no-arg constructor.

```java
if (nested == null && hasPropertiesWithPrefix(prefix + "." + fieldName)) {
    try {
        var ctor = field.getType().getDeclaredConstructor();
        ctor.setAccessible(true);
        nested = ctor.newInstance();
        setFieldValue(target, field, nested);
    } catch (ReflectiveOperationException ex) { return; }
}
```

The `hasPropertiesWithPrefix` method iterates all property sources and checks their backing maps for keys starting with the given prefix.

</details>

---

## 14.7 Tests

### Unit Tests

**New file:** `iris-boot-core/src/test/java/com/iris/boot/context/properties/ConfigurationPropertiesBinderTest.java`

```java
@Test
void shouldBindStringProperty_WhenExactKeyExists() {
    addProperties("test", Map.of("app.host", "myhost"));
    var target = new SimpleProperties();
    new ConfigurationPropertiesBinder(environment).bind(target, "app");
    assertThat(target.host).isEqualTo("myhost");
}

@Test
void shouldBindKebabCaseProperty_WhenFieldIsCamelCase() {
    addProperties("test", Map.of("app.max-connections", "200"));
    var target = new RelaxedProperties();
    new ConfigurationPropertiesBinder(environment).bind(target, "app");
    assertThat(target.maxConnections).isEqualTo(200);
}

@Test
void shouldAutoCreateNestedObject_WhenPropertiesExistAndFieldIsNull() {
    addProperties("test", Map.of("app.ssl.enabled", "true", "app.ssl.protocol", "TLSv1.2"));
    var target = new NestedProperties();
    new ConfigurationPropertiesBinder(environment).bind(target, "app");
    assertThat(target.ssl).isNotNull();
    assertThat(target.ssl.enabled).isTrue();
}

@Test
void shouldPreserveDefaultValue_WhenPropertyNotDefined() {
    var target = new SimpleProperties();
    new ConfigurationPropertiesBinder(environment).bind(target, "app");
    assertThat(target.port).isEqualTo(8080); // default
}
```

**New file:** `iris-boot-core/src/test/java/com/iris/boot/context/properties/ConfigurationPropertiesBindingPostProcessorTest.java`

```java
@Test
void shouldBindProperties_WhenBeanHasConfigurationPropertiesAnnotation() {
    addProperties(Map.of("server.port", "9090", "server.host", "myhost"));
    var bean = new ServerProperties();
    processor.postProcessBeforeInitialization(bean, "serverProperties");
    assertThat(bean.getPort()).isEqualTo(9090);
    assertThat(bean.getHost()).isEqualTo("myhost");
}

@Test
void shouldNotModifyBean_WhenBeanHasNoAnnotation() {
    addProperties(Map.of("server.port", "9090"));
    var bean = new PlainBean();
    bean.port = 8080;
    processor.postProcessBeforeInitialization(bean, "plainBean");
    assertThat(bean.port).isEqualTo(8080); // unchanged
}
```

### Integration Tests

**New file:** `iris-boot-core/src/test/java/com/iris/boot/context/properties/integration/ConfigurationPropertiesIntegrationTest.java`

```java
@Test
void shouldRegisterAndBindPropertiesBean_WhenEnabledOnConfiguration() {
    var env = createEnvironment(Map.of("server.port", "9090", "server.host", "myhost"));
    var ctx = new AnnotationConfigApplicationContext();
    ctx.setEnvironment(env);
    ctx.addInternalBeanPostProcessor(new ConfigurationPropertiesBindingPostProcessor(env));
    ctx.register(ServerAutoConfiguration.class);
    ctx.refresh();

    ServerProperties props = ctx.getBean(ServerProperties.class);
    assertThat(props.getPort()).isEqualTo(9090);
    assertThat(props.getHost()).isEqualTo("myhost");
    ctx.close();
}

@Test
void shouldBindPropertiesBeforePostConstruct_WhenBothArePresent() {
    var env = createEnvironment(Map.of("lifecycle.name", "test"));
    var ctx = new AnnotationConfigApplicationContext();
    ctx.setEnvironment(env);
    ctx.addInternalBeanPostProcessor(new ConfigurationPropertiesBindingPostProcessor(env));
    ctx.register(LifecycleConfiguration.class);
    ctx.refresh();

    LifecycleProperties props = ctx.getBean(LifecycleProperties.class);
    assertThat(props.getNameInPostConstruct()).isEqualTo("test"); // bound before @PostConstruct
    ctx.close();
}
```

**Run:** `./gradlew test` — expected: all tests pass (173+ tests including prior features)

---

## 14.8 Why This Works

> ★ **Insight** -------------------------------------------
> **The BPP ordering chain is the backbone of Spring Boot's property binding.** The real Spring Boot uses `PriorityOrdered` with `HIGHEST_PRECEDENCE + 1` to ensure `ConfigurationPropertiesBindingPostProcessor` runs before almost everything else — especially before `@PostConstruct`. Our simplified version achieves the same through an explicit insertion slot in `registerBeanPostProcessors()`. The key invariant: **bound properties must be visible in `@PostConstruct`**, because initialization logic often reads configuration.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The `Binder` in real Spring Boot is a 600-line recursive descent engine.** It supports four forms of relaxed binding (camelCase, kebab-case, underscore_case, SCREAMING_SNAKE), indexed list binding (`servers[0].host`), `Map` binding, constructor binding for immutable value objects, and a `BindHandler` chain (like a filter pipeline) for validation and error handling. Our 100-line binder covers the essential algorithm — prefix + field traversal with type conversion — which is enough to understand the pattern. The sophistication comes from edge cases, not from a different algorithm.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Cross-module annotation detection by name is a common Spring pattern.** Our `ConfigurationClassProcessor` detects `@EnableConfigurationProperties` by its simple name string, avoiding a compile-time dependency from `iris-framework` on `iris-boot-core`. Real Spring Boot solves this differently with `@Import(EnableConfigurationPropertiesRegistrar.class)`, where the framework's `@Import` mechanism discovers and invokes the registrar without the processor knowing about the annotation. Both approaches achieve the same goal: **the framework layer processes boot-layer annotations without coupling to them**.
> -----------------------------------------------------------

---

## 14.9 What We Enhanced

| Aspect | Before (ch13) | Current (ch14) | Real Framework |
|--------|---------------|----------------|----------------|
| **Property injection** | `@Value("${key}")` per field — no grouping | `@ConfigurationProperties(prefix)` binds entire prefix to a POJO | Full `Binder` with `JavaBeanBinder`, `ValueObjectBinder`, relaxed binding, `BindHandler` chains |
| **BPP ordering** | Three internal BPPs, no extension point | `addInternalBeanPostProcessor()` allows boot layer to insert BPPs at specific positions | `PriorityOrdered`/`Ordered` interfaces with `PostProcessorRegistrationDelegate` sorting |
| **Bean registration from annotations** | Only `@Bean` methods and `@Component` scanning | `@EnableConfigurationProperties` registers listed classes as beans | `@Import(EnableConfigurationPropertiesRegistrar.class)` with full `@Import` mechanism |
| **Relaxed binding** | None — exact key match only | camelCase ↔ kebab-case (`maxConnections` ↔ `max-connections`) | Four forms + `ConfigurationPropertyName` abstraction |
| **Nested objects** | Not supported | Recursive binding with auto-creation when properties exist | `JavaBeanBinder` with `BeanSupplier` lazy creation and `DataObjectPropertyBinder` |

---

## 14.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ConfigurationProperties` | `ConfigurationProperties` | `ConfigurationProperties.java:52` | Real version has `value`/`prefix` aliases via `@AliasFor`, `ignoreInvalidFields`, `ignoreUnknownFields` |
| `EnableConfigurationProperties` | `EnableConfigurationProperties` | `EnableConfigurationProperties.java:41` | Real version uses `@Import(EnableConfigurationPropertiesRegistrar.class)` for the registration trigger |
| `ConfigurationPropertiesBindingPostProcessor` | `ConfigurationPropertiesBindingPostProcessor` | `ConfigurationPropertiesBindingPostProcessor.java:46` | Real version implements `PriorityOrdered`, skips value objects (bound at instantiation) |
| `postProcessBeforeInitialization()` | `postProcessBeforeInitialization()` | `ConfigurationPropertiesBindingPostProcessor.java:82` | Real version delegates to `ConfigurationPropertiesBean.get()` which checks class + factory method annotations |
| `ConfigurationPropertiesBinder.bind()` | `ConfigurationPropertiesBinder.bind()` | `ConfigurationPropertiesBinder.java:92` | Real version builds a `BindHandler` chain and delegates to `Binder` |
| `ConfigurationPropertiesBinder` (ours) | `Binder` | `Binder.java:61` | Real `Binder` is a generic recursive engine with `ConfigurationPropertyName`, `BindResult`, `Bindable` abstractions |
| `bindField()` for nested objects | `bindDataObject()` | `Binder.java:495` | Real version delegates to `JavaBeanBinder` or `ValueObjectBinder` via `DataObjectBinder` SPI |
| `toKebabCase()` | `DataObjectPropertyName.toDashedForm()` | `DataObjectPropertyName.java:37` | Real version also handles underscores and adjacent uppercase letters |
| `processEnableConfigurationProperties()` | `EnableConfigurationPropertiesRegistrar.registerBeanDefinitions()` | `EnableConfigurationPropertiesRegistrar.java:45` | Real version also registers the BPP and `BoundConfigurationProperties` tracking bean |

---

## 14.11 Complete Code

### Production Code

#### File: `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationProperties.java` [NEW]

```java
package com.iris.boot.context.properties;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {

    String prefix() default "";
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/context/properties/EnableConfigurationProperties.java` [NEW]

```java
package com.iris.boot.context.properties;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableConfigurationProperties {

    Class<?>[] value() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBinder.java` [NEW]

```java
package com.iris.boot.context.properties;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.List;

import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.PropertySource;
import com.iris.framework.core.env.PropertySourcesPropertyResolver;

public class ConfigurationPropertiesBinder {

    private final Environment environment;

    public ConfigurationPropertiesBinder(Environment environment) {
        this.environment = environment;
    }

    public void bind(Object target, String prefix) {
        Class<?> targetClass = target.getClass();
        bindFields(target, targetClass, prefix);
    }

    private void bindFields(Object target, Class<?> clazz, String prefix) {
        if (clazz == null || clazz == Object.class) {
            return;
        }

        for (Field field : clazz.getDeclaredFields()) {
            if (Modifier.isStatic(field.getModifiers())
                    || Modifier.isFinal(field.getModifiers())) {
                continue;
            }
            bindField(target, field, prefix);
        }

        bindFields(target, clazz.getSuperclass(), prefix);
    }

    private void bindField(Object target, Field field, String prefix) {
        String fieldName = field.getName();
        Class<?> fieldType = field.getType();

        if (isSimpleType(fieldType)) {
            bindSimpleField(target, field, prefix, fieldName, fieldType);
        } else if (fieldType == List.class) {
            bindListField(target, field, prefix, fieldName);
        } else {
            bindNestedObject(target, field, prefix, fieldName);
        }
    }

    private void bindSimpleField(Object target, Field field, String prefix,
                                  String fieldName, Class<?> fieldType) {
        String value = resolveProperty(prefix, fieldName);
        if (value != null) {
            Object converted = PropertySourcesPropertyResolver.convertValue(value, fieldType);
            setFieldValue(target, field, converted);
        }
    }

    private void bindListField(Object target, Field field, String prefix,
                                String fieldName) {
        String value = resolveProperty(prefix, fieldName);
        if (value != null) {
            List<String> list = Arrays.stream(value.split(","))
                    .map(String::trim)
                    .filter(s -> !s.isEmpty())
                    .toList();
            setFieldValue(target, field, list);
        }
    }

    private void bindNestedObject(Object target, Field field, String prefix,
                                   String fieldName) {
        Object nested = getFieldValue(target, field);

        if (nested == null) {
            String nestedPrefix = prefix + "." + fieldName;
            String nestedKebabPrefix = prefix + "." + toKebabCase(fieldName);
            if (hasPropertiesWithPrefix(nestedPrefix)
                    || hasPropertiesWithPrefix(nestedKebabPrefix)) {
                try {
                    var ctor = field.getType().getDeclaredConstructor();
                    ctor.setAccessible(true);
                    nested = ctor.newInstance();
                    setFieldValue(target, field, nested);
                } catch (ReflectiveOperationException ex) {
                    return;
                }
            }
        }

        if (nested != null) {
            bind(nested, prefix + "." + fieldName);
        }
    }

    private String resolveProperty(String prefix, String fieldName) {
        String value = environment.getProperty(prefix + "." + fieldName);
        if (value != null) {
            return value;
        }

        String kebabName = toKebabCase(fieldName);
        if (!kebabName.equals(fieldName)) {
            value = environment.getProperty(prefix + "." + kebabName);
        }
        return value;
    }

    private boolean hasPropertiesWithPrefix(String prefix) {
        String dotPrefix = prefix + ".";
        for (PropertySource<?> ps : environment.getPropertySources()) {
            Object source = ps.getSource();
            if (source instanceof java.util.Map<?, ?> map) {
                for (Object key : map.keySet()) {
                    if (key instanceof String strKey && strKey.startsWith(dotPrefix)) {
                        return true;
                    }
                }
            }
        }
        return false;
    }

    private static Object getFieldValue(Object target, Field field) {
        try {
            field.setAccessible(true);
            return field.get(target);
        } catch (IllegalAccessException ex) {
            return null;
        }
    }

    private static void setFieldValue(Object target, Field field, Object value) {
        try {
            field.setAccessible(true);
            field.set(target, value);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    "Failed to set field '" + field.getName() + "' on "
                    + target.getClass().getSimpleName(), ex);
        }
    }

    static boolean isSimpleType(Class<?> type) {
        return type == String.class
                || type == int.class || type == Integer.class
                || type == long.class || type == Long.class
                || type == boolean.class || type == Boolean.class
                || type == double.class || type == Double.class;
    }

    static String toKebabCase(String camelCase) {
        if (camelCase == null || camelCase.isEmpty()) {
            return camelCase;
        }
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < camelCase.length(); i++) {
            char c = camelCase.charAt(i);
            if (Character.isUpperCase(c)) {
                if (i > 0) {
                    result.append('-');
                }
                result.append(Character.toLowerCase(c));
            } else {
                result.append(c);
            }
        }
        return result.toString();
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBindingPostProcessor.java` [NEW]

```java
package com.iris.boot.context.properties;

import com.iris.framework.beans.factory.config.BeanPostProcessor;
import com.iris.framework.core.env.Environment;

public class ConfigurationPropertiesBindingPostProcessor implements BeanPostProcessor {

    private final ConfigurationPropertiesBinder binder;

    public ConfigurationPropertiesBindingPostProcessor(Environment environment) {
        this.binder = new ConfigurationPropertiesBinder(environment);
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        Class<?> beanClass = bean.getClass();
        ConfigurationProperties annotation = beanClass.getAnnotation(ConfigurationProperties.class);
        if (annotation == null) {
            return bean;
        }

        String prefix = annotation.prefix();
        if (prefix.isEmpty()) {
            return bean;
        }

        binder.bind(bean, prefix);
        return bean;
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java` [MODIFIED]

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.annotation.AnnotationUtils;
import com.iris.framework.core.env.Environment;

public class ConfigurationClassProcessor {

    public void processConfigurationClasses(BeanDefinitionRegistry registry) {
        processConfigurationClasses(registry, null);
    }

    public void processConfigurationClasses(BeanDefinitionRegistry registry,
                                            Environment environment) {
        ConditionEvaluator evaluator = new ConditionEvaluator(registry, environment);

        performComponentScanning(registry);

        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String candidateName : candidateNames) {
            BeanDefinition bd = registry.getBeanDefinition(candidateName);
            Class<?> beanClass = bd.getBeanClass();

            if (!AnnotationUtils.hasAnnotation(beanClass, Configuration.class)) {
                continue;
            }

            if (evaluator.shouldSkip(beanClass)) {
                continue;
            }

            List<Method> beanMethods = findBeanMethods(beanClass);
            for (Method method : beanMethods) {
                if (evaluator.shouldSkip(method)) {
                    continue;
                }

                String beanName = resolveBeanName(method);
                BeanDefinition beanDef = createBeanDefinition(method, candidateName);
                registry.registerBeanDefinition(beanName, beanDef);
            }

            // Process @EnableConfigurationProperties — register listed classes as beans
            processEnableConfigurationProperties(beanClass, registry);
        }
    }

    private void processEnableConfigurationProperties(Class<?> configClass,
                                                       BeanDefinitionRegistry registry) {
        for (Annotation annotation : configClass.getAnnotations()) {
            String simpleName = annotation.annotationType().getSimpleName();
            if (!"EnableConfigurationProperties".equals(simpleName)) {
                continue;
            }

            try {
                Method valueMethod = annotation.annotationType().getMethod("value");
                Class<?>[] classes = (Class<?>[]) valueMethod.invoke(annotation);
                for (Class<?> propsClass : classes) {
                    String beanName = deriveConfigBeanName(propsClass);
                    if (!registry.containsBeanDefinition(beanName)) {
                        registry.registerBeanDefinition(beanName,
                                new BeanDefinition(propsClass));
                    }
                }
            } catch (ReflectiveOperationException ex) {
                // Annotation doesn't have a value() method — skip
            }
        }
    }

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
                basePackages = new String[]{ beanClass.getPackageName() };
            }

            scanner.scan(registry, basePackages);
        }
    }

    private List<Method> findBeanMethods(Class<?> configClass) {
        List<Method> beanMethods = new ArrayList<>();
        for (Method method : configClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Bean.class)) {
                beanMethods.add(method);
            }
        }
        return beanMethods;
    }

    private String resolveBeanName(Method method) {
        Bean beanAnnotation = method.getAnnotation(Bean.class);
        String explicitName = beanAnnotation.value();
        return explicitName.isEmpty() ? method.getName() : explicitName;
    }

    private BeanDefinition createBeanDefinition(Method method, String configBeanName) {
        BeanDefinition bd = new BeanDefinition(method.getReturnType());
        bd.setFactoryBeanName(configBeanName);
        bd.setFactoryMethod(method);
        return bd;
    }

    public static String deriveConfigBeanName(Class<?> configClass) {
        String simpleName = configClass.getSimpleName();
        if (simpleName.isEmpty()) {
            return simpleName;
        }
        return Character.toLowerCase(simpleName.charAt(0)) + simpleName.substring(1);
    }
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java` [MODIFIED]

Changes: Added `additionalInternalBeanPostProcessors` field, `addInternalBeanPostProcessor()` method, and updated `registerBeanPostProcessors()` to insert additional BPPs between value injection and lifecycle callbacks.

*(Full file is 630 lines — the complete source is in the repository. Key additions are at lines 94-111 (field), 580-600 (method), and 355-375 (updated `registerBeanPostProcessors`))*

#### File: `iris-boot-core/src/main/java/com/iris/boot/IrisApplication.java` [MODIFIED]

Change in `prepareContext()` method — added at line 331:

```java
// Register ConfigurationPropertiesBindingPostProcessor as an internal BPP.
annotationContext.addInternalBeanPostProcessor(
        new ConfigurationPropertiesBindingPostProcessor(environment));
```

### Test Code

#### File: `iris-boot-core/src/test/java/com/iris/boot/context/properties/ConfigurationPropertiesBinderTest.java` [NEW]

*(16 tests covering: simple type binding, relaxed binding, list binding, nested objects, kebab-case conversion — see repository for full source)*

#### File: `iris-boot-core/src/test/java/com/iris/boot/context/properties/ConfigurationPropertiesBindingPostProcessorTest.java` [NEW]

*(6 tests covering: annotation detection, no-op for plain beans, empty prefix handling, default preservation — see repository for full source)*

#### File: `iris-boot-core/src/test/java/com/iris/boot/context/properties/integration/ConfigurationPropertiesIntegrationTest.java` [NEW]

*(6 tests covering: full context lifecycle, @Bean parameter injection, nested binding, @PostConstruct ordering, multiple property classes — see repository for full source)*

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@ConfigurationProperties`** | Marks a POJO for type-safe binding — all properties under a prefix are mapped to its fields |
| **`@EnableConfigurationProperties`** | Registers property classes as beans and triggers binding — the bridge between `@Configuration` and `@ConfigurationProperties` |
| **`ConfigurationPropertiesBinder`** | The binding engine — iterates fields, resolves properties with relaxed binding, converts types, recurses for nested objects |
| **BPP ordering** | The binding post-processor runs after `@Autowired`/`@Value` but before `@PostConstruct` — bound values are available in initialization callbacks |
| **Relaxed binding** | `maxConnections` and `max-connections` both resolve — properties don't need to match Java naming conventions exactly |

**Next: Chapter 15 — @IrisBootApplication** — A single meta-annotation that combines `@Configuration`, `@ComponentScan`, and `@EnableAutoConfiguration`, plus built-in auto-configuration classes for Tomcat, MVC, and Jackson — bringing the full "just run `main()`" experience.
