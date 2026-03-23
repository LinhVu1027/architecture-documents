# Chapter 12: Conditional Beans

> Line references based on commits `11ab0b4351` (Spring Framework) and `5922311a95a` (Spring Boot).

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| `ConfigurationClassProcessor` registers every `@Bean` method unconditionally — all `@Configuration` classes and all `@Bean` methods always produce bean definitions (Features 4–11) | No way to conditionally register beans based on the classpath, existing beans, or property values — every auto-configuration class would blindly overwrite user-defined beans, and library-specific beans would fail when the library isn't present | Build a `Condition` SPI and `@Conditional` meta-annotation, then implement `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `@ConditionalOnProperty` so that bean registration can be gated at both the class and method level |

---

## 12.1 The Integration Point: ConfigurationClassProcessor Gains Condition Evaluation

The integration point is `ConfigurationClassProcessor.processConfigurationClasses()` — the method that iterates `@Configuration` classes and registers their `@Bean` methods. Two `evaluator.shouldSkip()` calls are inserted: one before processing a class (skip the entire `@Configuration` if class-level conditions fail) and one before registering each `@Bean` (skip individual beans if method-level conditions fail).

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`
**Change:** Add `Environment` parameter, create `ConditionEvaluator`, and insert two condition checks into the existing processing loop

```java
public void processConfigurationClasses(BeanDefinitionRegistry registry,
                                        Environment environment) {
    ConditionEvaluator evaluator = new ConditionEvaluator(registry, environment);

    // Phase 1: Component scanning (unchanged)
    performComponentScanning(registry);

    // Phase 2: @Bean method processing with condition evaluation
    String[] candidateNames = registry.getBeanDefinitionNames();

    for (String candidateName : candidateNames) {
        BeanDefinition bd = registry.getBeanDefinition(candidateName);
        Class<?> beanClass = bd.getBeanClass();

        if (!beanClass.isAnnotationPresent(Configuration.class)) {
            continue;
        }

        // NEW: Evaluate class-level conditions
        if (evaluator.shouldSkip(beanClass)) {
            continue;
        }

        List<Method> beanMethods = findBeanMethods(beanClass);
        for (Method method : beanMethods) {
            // NEW: Evaluate method-level conditions
            if (evaluator.shouldSkip(method)) {
                continue;
            }

            String beanName = resolveBeanName(method);
            BeanDefinition beanDef = createBeanDefinition(method, candidateName);
            registry.registerBeanDefinition(beanName, beanDef);
        }
    }
}
```

The diff against the previous version:

```diff
-public void processConfigurationClasses(BeanDefinitionRegistry registry) {
+public void processConfigurationClasses(BeanDefinitionRegistry registry,
+                                        Environment environment) {
+    ConditionEvaluator evaluator = new ConditionEvaluator(registry, environment);
+
     performComponentScanning(registry);
     String[] candidateNames = registry.getBeanDefinitionNames();

     for (String candidateName : candidateNames) {
         // ... get beanClass, check @Configuration ...

+        if (evaluator.shouldSkip(beanClass)) {
+            continue;
+        }
+
         List<Method> beanMethods = findBeanMethods(beanClass);
         for (Method method : beanMethods) {
+            if (evaluator.shouldSkip(method)) {
+                continue;
+            }
+
             String beanName = resolveBeanName(method);
             BeanDefinition beanDef = createBeanDefinition(method, candidateName);
             registry.registerBeanDefinition(beanName, beanDef);
         }
     }
 }
```

A backward-compatible overload delegates to the new method with `null` environment:

```java
public void processConfigurationClasses(BeanDefinitionRegistry registry) {
    processConfigurationClasses(registry, null);
}
```

Two key decisions here:

1. **Why evaluate in `ConfigurationClassProcessor` rather than in `DefaultBeanFactory`?** Conditions are about *registration*, not *creation*. A bean that fails its condition should never have a `BeanDefinition` in the registry — downstream code that queries by type should not see it. Putting the check at registration time is the earliest meaningful point.

2. **Why pass `Environment` explicitly rather than deriving it from the registry?** Our `DefaultBeanFactory` doesn't hold an `Environment`. The real Spring derives it from the registry (if it's also a `BeanFactory` that implements `EnvironmentCapable`). Passing it explicitly keeps the code simple and avoids adding interface proliferation to the factory.

This connects the **condition evaluation system** to the **bean registration pipeline**. To make it work, we need to build:
- `Condition` interface — the SPI that condition implementations fulfill
- `@Conditional` annotation — the meta-annotation linking conditions to classes/methods
- `ConditionContext` interface — the context object passed to conditions
- `ConditionEvaluator` — the engine that collects and evaluates conditions
- Three concrete conditions in `iris-boot-core`: `OnClassCondition`, `OnBeanCondition`, `OnPropertyCondition`

**Modifying:** `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java`
**Change:** Pass `this.environment` to `processConfigurationClasses()` in `invokeBeanFactoryPostProcessors()`

```java
private void invokeBeanFactoryPostProcessors() {
    ConfigurationClassProcessor processor = new ConfigurationClassProcessor();
    processor.processConfigurationClasses(this.beanFactory, this.environment);
}
```

---

## 12.2 The Condition SPI (iris-framework)

Four new files form the condition evaluation infrastructure in `iris-framework`.

### Condition Interface

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/Condition.java`

```java
package com.iris.framework.context.annotation;

import java.lang.reflect.AnnotatedElement;

@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext context, AnnotatedElement element);
}
```

The `AnnotatedElement` parameter is key — it gives the condition access to *all* annotations on the target class or method. This is how `OnClassCondition` reads `@ConditionalOnClass` attributes and `OnPropertyCondition` reads `@ConditionalOnProperty` attributes, even though the framework only knows about the generic `@Conditional`.

### @Conditional Annotation

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/Conditional.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
    Class<? extends Condition>[] value();
}
```

Can target both types (classes) and methods (`@Bean` methods). The array allows multiple conditions — all must match.

### ConditionContext Interface

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConditionContext.java`

```java
package com.iris.framework.context.annotation;

import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.env.Environment;

public interface ConditionContext {
    BeanDefinitionRegistry getRegistry();
    Environment getEnvironment();
    ClassLoader getClassLoader();
}
```

Three resources for three condition types: registry for `@ConditionalOnMissingBean`, environment for `@ConditionalOnProperty`, class loader for `@ConditionalOnClass`.

### ConditionEvaluator

**New file:** `iris-framework/src/main/java/com/iris/framework/context/annotation/ConditionEvaluator.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.env.Environment;

public class ConditionEvaluator {

    private final ConditionContext context;

    public ConditionEvaluator(BeanDefinitionRegistry registry, Environment environment) {
        this.context = new DefaultConditionContext(registry, environment);
    }

    public boolean shouldSkip(AnnotatedElement element) {
        List<Condition> conditions = collectConditions(element);
        if (conditions.isEmpty()) {
            return false;
        }
        for (Condition condition : conditions) {
            if (!condition.matches(context, element)) {
                return true;
            }
        }
        return false;
    }

    private List<Condition> collectConditions(AnnotatedElement element) {
        List<Condition> conditions = new ArrayList<>();

        // 1. Direct @Conditional on the element
        Conditional direct = element.getAnnotation(Conditional.class);
        if (direct != null) {
            for (Class<? extends Condition> condClass : direct.value()) {
                conditions.add(instantiate(condClass));
            }
        }

        // 2. @Conditional as meta-annotation on the element's annotations
        for (Annotation ann : element.getAnnotations()) {
            Class<? extends Annotation> annType = ann.annotationType();
            if (annType.getName().startsWith("java.lang.annotation")) continue;
            if (annType == Conditional.class) continue;

            Conditional meta = annType.getAnnotation(Conditional.class);
            if (meta != null) {
                for (Class<? extends Condition> condClass : meta.value()) {
                    conditions.add(instantiate(condClass));
                }
            }
        }

        return conditions;
    }

    private Condition instantiate(Class<? extends Condition> conditionClass) {
        try {
            var ctor = conditionClass.getDeclaredConstructor();
            ctor.setAccessible(true);
            return ctor.newInstance();
        } catch (Exception ex) {
            throw new IllegalStateException(
                    "Failed to instantiate condition class '" + conditionClass.getName() + "'", ex);
        }
    }

    private static class DefaultConditionContext implements ConditionContext {
        private final BeanDefinitionRegistry registry;
        private final Environment environment;
        private final ClassLoader classLoader;

        DefaultConditionContext(BeanDefinitionRegistry registry, Environment environment) {
            this.registry = registry;
            this.environment = environment;
            ClassLoader cl = Thread.currentThread().getContextClassLoader();
            this.classLoader = (cl != null) ? cl : getClass().getClassLoader();
        }

        @Override public BeanDefinitionRegistry getRegistry() { return registry; }
        @Override public Environment getEnvironment() { return environment; }
        @Override public ClassLoader getClassLoader() { return classLoader; }
    }
}
```

The `collectConditions()` method handles two levels:
1. **Direct** — `@Conditional(FooCondition.class)` on the element
2. **Meta-annotation** — `@ConditionalOnClass` on the element, where `@ConditionalOnClass` itself carries `@Conditional(OnClassCondition.class)`

This is the pattern that decouples `iris-framework` from `iris-boot-core`: the framework only knows about `@Conditional` and `Condition`, but discovers Boot's condition implementations through the meta-annotation at runtime.

---

## 12.3 Three Concrete Conditions (iris-boot-core)

### @ConditionalOnClass + OnClassCondition

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/ConditionalOnClass.java`

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.annotation.*;
import com.iris.framework.context.annotation.Conditional;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
    Class<?>[] value() default {};
    String[] name() default {};
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/OnClassCondition.java`

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.reflect.AnnotatedElement;
import com.iris.framework.context.annotation.Condition;
import com.iris.framework.context.annotation.ConditionContext;

public class OnClassCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnClass annotation = element.getAnnotation(ConditionalOnClass.class);
        if (annotation == null) return true;

        ClassLoader classLoader = context.getClassLoader();

        for (Class<?> clazz : annotation.value()) {
            try {
                Class.forName(clazz.getName(), false, classLoader);
            } catch (ClassNotFoundException e) {
                return false;
            }
        }

        for (String className : annotation.name()) {
            try {
                Class.forName(className, false, classLoader);
            } catch (ClassNotFoundException e) {
                return false;
            }
        }

        return true;
    }
}
```

Uses `Class.forName(name, false, classLoader)` — the `false` parameter skips class initialization (no static initializers run). The `name` attribute is the safe choice when the class might be absent; the `value` attribute (class references) requires the class at compile time.

### @ConditionalOnMissingBean + OnBeanCondition

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/ConditionalOnMissingBean.java`

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.annotation.*;
import com.iris.framework.context.annotation.Conditional;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnMissingBean {
    Class<?>[] value() default {};
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/OnBeanCondition.java`

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.context.annotation.Condition;
import com.iris.framework.context.annotation.ConditionContext;

public class OnBeanCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnMissingBean annotation = element.getAnnotation(ConditionalOnMissingBean.class);
        if (annotation == null) return true;

        Class<?>[] types = annotation.value();

        // Deduce from @Bean method return type if no explicit types
        if (types.length == 0 && element instanceof Method method) {
            types = new Class<?>[]{ method.getReturnType() };
        }
        if (types.length == 0) return true;

        BeanDefinitionRegistry registry = context.getRegistry();
        for (Class<?> type : types) {
            if (hasBeanOfType(registry, type)) {
                return false; // Bean exists → condition NOT met → skip
            }
        }
        return true; // No beans → condition met → register
    }

    private boolean hasBeanOfType(BeanDefinitionRegistry registry, Class<?> type) {
        for (String name : registry.getBeanDefinitionNames()) {
            BeanDefinition bd = registry.getBeanDefinition(name);
            if (type.isAssignableFrom(bd.getBeanClass())) {
                return true;
            }
        }
        return false;
    }
}
```

The return-type deduction (`element instanceof Method method`) is the pattern that makes `@ConditionalOnMissingBean` on a `@Bean` method so concise — you just write `@ConditionalOnMissingBean` with no arguments, and it deduces the type to check from the method's return type. The `hasBeanOfType()` method inspects bean *definitions* (metadata), not bean *instances* — no beans are instantiated during condition evaluation.

### @ConditionalOnProperty + OnPropertyCondition

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/ConditionalOnProperty.java`

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.annotation.*;
import com.iris.framework.context.annotation.Conditional;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {
    String[] value() default {};
    String prefix() default "";
    String[] name() default {};
    String havingValue() default "";
    boolean matchIfMissing() default false;
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/OnPropertyCondition.java`

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.reflect.AnnotatedElement;
import com.iris.framework.context.annotation.Condition;
import com.iris.framework.context.annotation.ConditionContext;
import com.iris.framework.core.env.Environment;

public class OnPropertyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnProperty annotation = element.getAnnotation(ConditionalOnProperty.class);
        if (annotation == null) return true;

        Environment environment = context.getEnvironment();
        if (environment == null) return false;

        String prefix = annotation.prefix();
        String havingValue = annotation.havingValue();
        boolean matchIfMissing = annotation.matchIfMissing();
        String[] names = annotation.name().length > 0 ? annotation.name() : annotation.value();

        for (String name : names) {
            String key = prefix.isEmpty() ? name : prefix + "." + name;
            String value = environment.getProperty(key);

            if (value == null) {
                if (!matchIfMissing) return false;
            } else {
                if (!havingValue.isEmpty()) {
                    if (!havingValue.equalsIgnoreCase(value)) return false;
                } else {
                    if ("false".equalsIgnoreCase(value)) return false;
                }
            }
        }
        return true;
    }
}
```

The matching rules mirror the real Spring Boot:
- **No `havingValue`**: property must exist and not be `"false"`
- **With `havingValue`**: property must equal that value (case-insensitive)
- **`matchIfMissing = true`**: absent properties are treated as matching

---

## 12.3 Try It Yourself

<details>
<summary>Challenge 1: Implement a @ConditionalOnMissingBean that deduces the type from the @Bean method's return type</summary>

Given this auto-configuration:

```java
@Configuration
public class DataSourceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean  // no explicit type!
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

How does `OnBeanCondition` know to check for `DataSource.class`?

**Answer:** The `element` parameter is a `java.lang.reflect.Method`. When `annotation.value()` is empty, cast the element to `Method` and call `method.getReturnType()`:

```java
if (types.length == 0 && element instanceof Method method) {
    types = new Class<?>[]{ method.getReturnType() };
}
```

This is the Java 17 pattern matching for `instanceof` — it both checks the type and binds the variable in one expression.

</details>

<details>
<summary>Challenge 2: Write a custom @ConditionalOnJdk annotation that matches a minimum Java version</summary>

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnJdkCondition.class)
public @interface ConditionalOnJdk {
    int value(); // minimum version, e.g., 21
}

public class OnJdkCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnJdk annotation = element.getAnnotation(ConditionalOnJdk.class);
        if (annotation == null) return true;
        int required = annotation.value();
        int current = Runtime.version().feature();
        return current >= required;
    }
}
```

This demonstrates how the `@Conditional` meta-annotation pattern makes the system extensible — you create a new annotation, a new `Condition` implementation, and link them with `@Conditional(YourCondition.class)`. No changes to the framework are needed.

</details>

---

## 12.4 Tests

### Unit Tests — ConditionEvaluator

**New file:** `iris-framework/src/test/java/com/iris/framework/context/annotation/ConditionEvaluatorTest.java`

```java
@Test
@DisplayName("should not skip when no conditions are present")
void shouldNotSkip_WhenNoConditions() {
    assertThat(evaluator.shouldSkip(NoConditions.class)).isFalse();
}

@Test
@DisplayName("should skip when direct @Conditional does not match")
void shouldSkip_WhenDirectConditionDoesNotMatch() {
    assertThat(evaluator.shouldSkip(DirectFalseCondition.class)).isTrue();
}

@Test
@DisplayName("should skip when meta-annotated condition does not match")
void shouldSkip_WhenMetaConditionDoesNotMatch() {
    assertThat(evaluator.shouldSkip(MetaFalseCondition.class)).isTrue();
}

@Test
@DisplayName("should skip when one of multiple direct conditions fails")
void shouldSkip_WhenOneOfMultipleDirectConditionsFails() {
    assertThat(evaluator.shouldSkip(MultipleConditionsOneFailsClass.class)).isTrue();
}

@Test
@DisplayName("should skip entire @Configuration class when class-level condition fails")
void shouldSkipEntireConfig_WhenClassLevelConditionFails() {
    factory.registerBeanDefinition("skippedConfig",
            new BeanDefinition(SkippedConfig.class));
    processor.processConfigurationClasses(factory, null);
    assertThat(factory.containsBeanDefinition("shouldNotBeRegistered")).isFalse();
}
```

### Unit Tests — OnClassCondition

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnClassConditionTest.java`

```java
@Test
@DisplayName("should register beans when class is present on classpath")
void shouldRegisterBeans_WhenClassPresent() {
    factory.registerBeanDefinition("config", new BeanDefinition(PresentClassConfig.class));
    processor.processConfigurationClasses(factory, null);
    assertThat(factory.containsBeanDefinition("presentBean")).isTrue();
}

@Test
@DisplayName("should skip beans when class is missing from classpath")
void shouldSkipBeans_WhenClassMissing() {
    factory.registerBeanDefinition("config", new BeanDefinition(MissingClassConfig.class));
    processor.processConfigurationClasses(factory, null);
    assertThat(factory.containsBeanDefinition("missingBean")).isFalse();
}
```

### Unit Tests — OnBeanCondition

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnBeanConditionTest.java`

```java
@Test
@DisplayName("should register default bean when no user bean exists")
void shouldRegisterDefaultBean_WhenNoUserBeanExists() {
    factory.registerBeanDefinition("autoConfig", new BeanDefinition(AutoConfig.class));
    processor.processConfigurationClasses(factory, null);
    assertThat(factory.containsBeanDefinition("greetingService")).isTrue();
}

@Test
@DisplayName("should skip bean with deduced type when same type already exists")
void shouldSkipBean_WhenDeducedTypeAlreadyExists() {
    factory.registerBeanDefinition("existingString",
            new BeanDefinition(String.class, () -> "existing"));
    factory.registerBeanDefinition("deducedTypeConfig",
            new BeanDefinition(DeducedTypeConfig.class));
    processor.processConfigurationClasses(factory, null);
    assertThat(factory.containsBeanDefinition("defaultMessage")).isFalse();
}
```

### Unit Tests — OnPropertyCondition

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnPropertyConditionTest.java`

```java
@Test
@DisplayName("should register bean when property exists and is not false")
void shouldRegisterBean_WhenPropertyExistsAndIsNotFalse() {
    environment.getPropertySources().addLast(
            new MapPropertySource("test", Map.of("feature.enabled", "true")));
    factory.registerBeanDefinition("config", new BeanDefinition(FeatureEnabledConfig.class));
    processor.processConfigurationClasses(factory, environment);
    assertThat(factory.containsBeanDefinition("featureBean")).isTrue();
}

@Test
@DisplayName("should register bean when havingValue matches case-insensitively")
void shouldMatchHavingValue_CaseInsensitively() {
    environment.getPropertySources().addLast(
            new MapPropertySource("test", Map.of("feature.type", "ADVANCED")));
    factory.registerBeanDefinition("config", new BeanDefinition(FeatureTypeConfig.class));
    processor.processConfigurationClasses(factory, environment);
    assertThat(factory.containsBeanDefinition("featureTypeBean")).isTrue();
}
```

### Integration Tests — Full Context Lifecycle

**New file:** `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/integration/ConditionalBeansIntegrationTest.java`

```java
@Test
@DisplayName("should back off auto-config when user provides same bean type")
void shouldBackOffAutoConfig_WhenUserProvidesSameBeanType() {
    var ctx = new AnnotationConfigApplicationContext();
    ctx.register(UserMessageConfiguration.class, MessageAutoConfiguration.class);
    ctx.refresh();
    MessageService service = ctx.getBean(MessageService.class);
    assertThat(service.getMessage()).isEqualTo("Custom message");
    ctx.close();
}

@Test
@DisplayName("should enable/disable beans based on property values")
void shouldEnableDisableBeans_BasedOnPropertyValues() {
    var ctx = new AnnotationConfigApplicationContext();
    StandardEnvironment env = new StandardEnvironment();
    env.getPropertySources().addLast(
            new MapPropertySource("test", Map.of("app.feature.x", "on")));
    ctx.setEnvironment(env);
    ctx.register(PropertyGatedConfig.class);
    ctx.refresh();
    assertThat(ctx.containsBean("featureX")).isTrue();
    assertThat(ctx.containsBean("featureY")).isFalse();
    ctx.close();
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 12.5 Why This Works

> ★ **Insight** -------------------------------------------
> **The meta-annotation pattern achieves open/closed extensibility.** `iris-framework` defines only `@Conditional` and `Condition` — it never imports `@ConditionalOnClass` or `OnClassCondition`. Boot-level conditions are discovered at runtime through the `@Conditional` meta-annotation on the custom annotation type. This means anyone can create new condition types (e.g., `@ConditionalOnJdk`, `@ConditionalOnBean`) without modifying the framework. This is the same pattern the real Spring uses: `spring-context` defines the SPI, and `spring-boot-autoconfigure` provides the implementations.
>
> **Trade-off:** Single-level meta-annotation traversal means you can't compose conditions like `@ConditionalOnFoo` → `@ConditionalOnClass`. The real Spring uses `MergedAnnotations` for deep traversal, which adds complexity but supports arbitrary composition depth.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Conditions inspect definitions, not instances.** `OnBeanCondition.hasBeanOfType()` iterates `BeanDefinition` metadata — it never calls `getBean()`. This is crucial because conditions run during `ConfigurationClassProcessor.processConfigurationClasses()`, which happens in Step 1 of `refresh()` — *before* any singletons are created (Step 4). Calling `getBean()` from a condition would trigger premature instantiation, potentially creating beans that should have been conditionally excluded.
>
> In the real Spring Boot, `OnBeanCondition` calls `beanFactory.getBeanNamesForType(type, true, false)` where the second `false` means "don't eagerly initialize FactoryBeans". Same principle: check metadata, don't create instances.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Evaluation order matters for `@ConditionalOnMissingBean`.** Our processor iterates `@Configuration` classes in registration order and processes `@Bean` methods within each class. This means that if `UserConfig` is registered before `AutoConfig`, the user's beans are in the registry *before* the auto-config's conditions are evaluated. The "back off if user provides their own" pattern depends on this ordering. In the real Spring Boot, `@AutoConfiguration(after = ...)` explicitly controls evaluation order, and `OnBeanCondition` implements `ConfigurationCondition` with `REGISTER_BEAN` phase to ensure all classes are parsed before any bean-presence conditions fire.
> -----------------------------------------------------------

---

## 12.6 What We Enhanced

| Aspect | Before (ch11) | Current (ch12) | Real Framework |
|--------|---------------|-----------------|----------------|
| Bean registration | Unconditional — every `@Bean` method always registers | Conditional — `@Conditional` on classes and methods gates registration | `ConfigurationClassParser` + `ConfigurationClassBeanDefinitionReader` with `ConditionEvaluator` and phase-aware `ConfigurationCondition` (`ConditionEvaluator.java`) |
| `ConfigurationClassProcessor` | Takes only `BeanDefinitionRegistry` | Takes `BeanDefinitionRegistry` + `Environment`; creates `ConditionEvaluator` | `ConfigurationClassPostProcessor` receives full `BeanDefinitionRegistry`, deduces environment via `ConditionContextImpl` |
| `AnnotationConfigApplicationContext.invokeBeanFactoryPostProcessors()` | Passes only `beanFactory` | Passes `beanFactory` + `environment` | Environment deduced from registry in `ConditionContextImpl.deduceEnvironment()` |
| Classpath awareness | None — all beans registered regardless of classpath | `@ConditionalOnClass` checks `Class.forName()` | `OnClassCondition` with fast-path `AutoConfigurationImportFilter` and `ClassNameFilter` (`OnClassCondition.java:90`) |
| Bean conflict resolution | Last registration wins (overwrites) | `@ConditionalOnMissingBean` checks existing definitions by type | `OnBeanCondition` with `getBeanNamesForType()`, annotation matching, `SearchStrategy`, `REGISTER_BEAN` phase (`OnBeanCondition.java:135`) |
| Property gating | None | `@ConditionalOnProperty` with prefix, havingValue, matchIfMissing | `OnPropertyCondition` with `@Repeatable`, `ConditionOutcome` messages (`OnPropertyCondition.java:54`) |

---

## 12.7 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Condition` | `Condition` | `spring-context/…/Condition.java:32` | Real receives `AnnotatedTypeMetadata` instead of `AnnotatedElement`; we use JDK reflection directly |
| `@Conditional` | `@Conditional` | `spring-context/…/Conditional.java:42` | Identical structure — both target TYPE and METHOD |
| `ConditionContext` | `ConditionContext` | `spring-context/…/ConditionContext.java:30` | Real adds `getBeanFactory()` and `getResourceLoader()` |
| `ConditionEvaluator` | `ConditionEvaluator` | `spring-context/…/ConditionEvaluator.java:55` | Real supports phase-aware evaluation (`ConfigurationCondition.ConfigurationPhase`), `@Order` sorting, and `MergedAnnotations` traversal |
| `OnClassCondition` | `OnClassCondition` | `spring-boot-autoconfigure/…/OnClassCondition.java:42` | Real extends `FilteringSpringBootCondition` for fast-path filtering, supports `@ConditionalOnMissingClass`, multi-threaded evaluation |
| `OnBeanCondition` | `OnBeanCondition` | `spring-boot-autoconfigure/…/OnBeanCondition.java:55` | Real implements `ConfigurationCondition` (REGISTER_BEAN phase), handles `@ConditionalOnBean`, `@ConditionalOnSingleCandidate`, annotation matching, `SearchStrategy` |
| `OnPropertyCondition` | `OnPropertyCondition` | `spring-boot-autoconfigure/…/OnPropertyCondition.java:38` | Real extends `SpringBootCondition` with `ConditionOutcome` reporting, supports `@Repeatable`, `@ConditionalOnBooleanProperty` |
| Two `shouldSkip()` calls in `ConfigurationClassProcessor` | Split across `ConfigurationClassParser` + `ConfigurationClassBeanDefinitionReader` | `ConfigurationClassParser.java:249`, `ConfigurationClassBeanDefinitionReader.java:196` | Real separates into PARSE_CONFIGURATION phase (class-level, skips `@Import`/`@ComponentScan`) and REGISTER_BEAN phase (method-level, handles `TrackedConditionEvaluator`) |

---

## 12.8 Complete Code

### Production Code

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/Condition.java` [NEW]

```java
package com.iris.framework.context.annotation;

import java.lang.reflect.AnnotatedElement;

/**
 * A single condition that must be {@linkplain #matches matched} in order for
 * a bean to be registered.
 *
 * <p>Conditions are checked just before a bean definition is registered in the
 * container — both at the class level ({@code @Configuration} classes) and at
 * the method level ({@code @Bean} methods). The {@code element} parameter gives
 * the condition access to all annotations on the class or method, so it can
 * inspect domain-specific attributes (e.g., which class to check for in
 * {@code @ConditionalOnClass}).
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code org.springframework.context.annotation.Condition}. The real interface
 * receives {@code AnnotatedTypeMetadata} (Spring's merged annotation model)
 * instead of a raw {@code AnnotatedElement}. We use the JDK reflection API
 * directly, which is sufficient for our single-level meta-annotation traversal.
 *
 * @see Conditional
 * @see ConditionContext
 * @see org.springframework.context.annotation.Condition
 */
@FunctionalInterface
public interface Condition {

    /**
     * Determine if the condition matches.
     *
     * @param context  access to the registry, environment, and class loader
     * @param element  the annotated class or method being evaluated
     * @return {@code true} if the condition matches and the bean should be registered
     */
    boolean matches(ConditionContext context, AnnotatedElement element);
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/Conditional.java` [NEW]

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that a component is only eligible for registration when all
 * {@linkplain #value specified conditions} match.
 *
 * <p>Can be used in two ways:
 * <ol>
 *   <li><b>Directly</b> — on any {@code @Configuration} class or {@code @Bean}
 *       method, referencing one or more {@link Condition} implementations</li>
 *   <li><b>As a meta-annotation</b> — on custom conditional annotations
 *       (e.g., {@code @ConditionalOnClass}, {@code @ConditionalOnProperty}).
 *       This is how Spring Boot defines its conditions: each annotation carries
 *       {@code @Conditional(SomeCondition.class)} and adds domain-specific
 *       attributes</li>
 * </ol>
 *
 * <p>All specified conditions must match for the bean to be registered.
 * If any condition returns {@code false}, the bean (or entire configuration
 * class) is skipped.
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code org.springframework.context.annotation.Conditional}. The real
 * annotation is identical in structure — the difference is in the evaluation
 * engine ({@code ConditionEvaluator}) which in the real framework supports
 * phase-aware evaluation ({@code ConfigurationCondition.ConfigurationPhase}).
 *
 * @see Condition
 * @see ConditionContext
 * @see ConditionEvaluator
 * @see org.springframework.context.annotation.Conditional
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

    /**
     * All {@link Condition} classes that must match for the component
     * to be registered.
     */
    Class<? extends Condition>[] value();
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/ConditionContext.java` [NEW]

```java
package com.iris.framework.context.annotation;

import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.env.Environment;

/**
 * Context information for use by {@link Condition} implementations.
 *
 * <p>Provides access to the three resources conditions typically need:
 * <ul>
 *   <li>{@link BeanDefinitionRegistry} — inspect which beans are already
 *       registered (for {@code @ConditionalOnMissingBean})</li>
 *   <li>{@link Environment} — resolve property values
 *       (for {@code @ConditionalOnProperty})</li>
 *   <li>{@link ClassLoader} — check classpath presence
 *       (for {@code @ConditionalOnClass})</li>
 * </ul>
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code org.springframework.context.annotation.ConditionContext}, which
 * also provides a {@code ConfigurableListableBeanFactory} and
 * {@code ResourceLoader}. We omit those since our simplified conditions
 * don't need them.
 *
 * @see Condition
 * @see org.springframework.context.annotation.ConditionContext
 */
public interface ConditionContext {

    /**
     * Return the {@link BeanDefinitionRegistry} that will hold the bean
     * definition if the condition matches.
     */
    BeanDefinitionRegistry getRegistry();

    /**
     * Return the {@link Environment}, or {@code null} if not available.
     */
    Environment getEnvironment();

    /**
     * Return the {@link ClassLoader} to use for evaluating classpath conditions.
     */
    ClassLoader getClassLoader();
}
```

#### File: `iris-framework/src/main/java/com/iris/framework/context/annotation/ConditionEvaluator.java` [NEW]

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.core.env.Environment;

/**
 * Internal class used to evaluate {@link Conditional @Conditional} annotations.
 *
 * <p>The evaluator collects all {@link Condition} classes referenced by
 * {@code @Conditional} annotations — both direct annotations and
 * meta-annotations — then asks each condition if it matches. All conditions
 * must match for the bean to be registered.
 *
 * <p>Condition collection works at two levels:
 * <ol>
 *   <li><b>Direct:</b> If the element has {@code @Conditional(FooCondition.class)},
 *       {@code FooCondition} is collected</li>
 *   <li><b>Meta-annotation:</b> If the element has {@code @ConditionalOnClass}
 *       which is itself annotated with {@code @Conditional(OnClassCondition.class)},
 *       {@code OnClassCondition} is collected. This enables the Spring Boot
 *       pattern where custom annotations carry their condition class as a
 *       meta-annotation</li>
 * </ol>
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code org.springframework.context.annotation.ConditionEvaluator} (a
 * package-private class). The real evaluator:
 * <ul>
 *   <li>Supports phase-aware evaluation ({@code ConfigurationCondition} with
 *       PARSE_CONFIGURATION vs. REGISTER_BEAN phases)</li>
 *   <li>Sorts conditions by {@code @Order} / {@code Ordered} before evaluation</li>
 *   <li>Uses Spring's merged annotation model ({@code AnnotatedTypeMetadata})
 *       for deep meta-annotation traversal</li>
 * </ul>
 *
 * <p>We simplify to a single evaluation phase and single-level
 * meta-annotation traversal, which covers all standard use cases.
 *
 * @see Condition
 * @see Conditional
 * @see ConditionContext
 * @see org.springframework.context.annotation.ConditionEvaluator
 */
public class ConditionEvaluator {

    private final ConditionContext context;

    /**
     * Create a new evaluator with the given registry and environment.
     *
     * @param registry    the bean definition registry
     * @param environment the environment (may be {@code null})
     */
    public ConditionEvaluator(BeanDefinitionRegistry registry, Environment environment) {
        this.context = new DefaultConditionContext(registry, environment);
    }

    /**
     * Determine if the given annotated element should be skipped based on
     * its {@code @Conditional} annotations.
     *
     * <p>Returns {@code true} (skip) if any condition does not match.
     * Returns {@code false} (do not skip) if there are no conditions
     * or all conditions match.
     *
     * <p>This maps to Spring's {@code ConditionEvaluator.shouldSkip()} which
     * is called from {@code ConfigurationClassParser.processConfigurationClass()}
     * (for class-level conditions) and from
     * {@code ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod()}
     * (for method-level conditions).
     *
     * @param element the class or method to evaluate
     * @return {@code true} if the element should be skipped
     */
    public boolean shouldSkip(AnnotatedElement element) {
        List<Condition> conditions = collectConditions(element);
        if (conditions.isEmpty()) {
            return false;
        }

        for (Condition condition : conditions) {
            if (!condition.matches(context, element)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Collect all {@link Condition} instances from {@code @Conditional}
     * annotations on the given element — both direct and meta-annotations.
     *
     * <p>In the real Spring Framework, this is
     * {@code ConditionEvaluator.collectConditions()} which uses
     * {@code metadata.getAllAnnotationAttributes(Conditional.class.getName(), true)}
     * to traverse the full meta-annotation hierarchy. We use a simpler
     * single-level check: inspect each annotation on the element and check
     * if that annotation type carries {@code @Conditional}.
     *
     * @param element the annotated element to inspect
     * @return all conditions found (may be empty, never {@code null})
     */
    private List<Condition> collectConditions(AnnotatedElement element) {
        List<Condition> conditions = new ArrayList<>();

        // 1. Direct @Conditional on the element itself
        Conditional direct = element.getAnnotation(Conditional.class);
        if (direct != null) {
            for (Class<? extends Condition> condClass : direct.value()) {
                conditions.add(instantiate(condClass));
            }
        }

        // 2. @Conditional as meta-annotation: check each annotation on the element
        //    to see if the annotation TYPE carries @Conditional
        for (Annotation ann : element.getAnnotations()) {
            Class<? extends Annotation> annType = ann.annotationType();

            // Skip JDK meta-annotations and @Conditional itself (already handled)
            if (annType.getName().startsWith("java.lang.annotation")) {
                continue;
            }
            if (annType == Conditional.class) {
                continue;
            }

            Conditional meta = annType.getAnnotation(Conditional.class);
            if (meta != null) {
                for (Class<? extends Condition> condClass : meta.value()) {
                    conditions.add(instantiate(condClass));
                }
            }
        }

        return conditions;
    }

    /**
     * Instantiate a {@link Condition} class using its no-arg constructor.
     *
     * <p>In the real Spring Framework, this is done by
     * {@code BeanUtils.instantiateClass()} which handles both public and
     * private constructors and wraps exceptions consistently.
     */
    private Condition instantiate(Class<? extends Condition> conditionClass) {
        try {
            var ctor = conditionClass.getDeclaredConstructor();
            ctor.setAccessible(true);
            return ctor.newInstance();
        } catch (Exception ex) {
            throw new IllegalStateException(
                    "Failed to instantiate condition class '" + conditionClass.getName() + "'", ex);
        }
    }

    // -----------------------------------------------------------------------
    // Default ConditionContext implementation
    // -----------------------------------------------------------------------

    /**
     * Simple immutable implementation of {@link ConditionContext}.
     *
     * <p>In the real Spring Framework, this is
     * {@code ConditionEvaluator.ConditionContextImpl} — a private inner class
     * that deduces the bean factory, environment, resource loader, and class
     * loader from the registry (if it's also a {@code BeanFactory} or
     * {@code ApplicationContext}). We receive all values explicitly.
     */
    private static class DefaultConditionContext implements ConditionContext {

        private final BeanDefinitionRegistry registry;
        private final Environment environment;
        private final ClassLoader classLoader;

        DefaultConditionContext(BeanDefinitionRegistry registry, Environment environment) {
            this.registry = registry;
            this.environment = environment;
            ClassLoader cl = Thread.currentThread().getContextClassLoader();
            this.classLoader = (cl != null) ? cl : getClass().getClassLoader();
        }

        @Override
        public BeanDefinitionRegistry getRegistry() {
            return registry;
        }

        @Override
        public Environment getEnvironment() {
            return environment;
        }

        @Override
        public ClassLoader getClassLoader() {
            return classLoader;
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

            if (!beanClass.isAnnotationPresent(Configuration.class)) {
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

            if (!beanClass.isAnnotationPresent(Configuration.class)) {
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

#### File: `iris-framework/src/main/java/com/iris/framework/context/AnnotationConfigApplicationContext.java` [MODIFIED]

Only the `invokeBeanFactoryPostProcessors()` method changed — now passes `this.environment` to `processConfigurationClasses()`. Full file omitted for brevity since only one line changed (line 263):

```java
private void invokeBeanFactoryPostProcessors() {
    ConfigurationClassProcessor processor = new ConfigurationClassProcessor();
    processor.processConfigurationClasses(this.beanFactory, this.environment);
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/ConditionalOnClass.java` [NEW]

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.context.annotation.Conditional;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
    Class<?>[] value() default {};
    String[] name() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/OnClassCondition.java` [NEW]

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.reflect.AnnotatedElement;

import com.iris.framework.context.annotation.Condition;
import com.iris.framework.context.annotation.ConditionContext;

public class OnClassCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnClass annotation = element.getAnnotation(ConditionalOnClass.class);
        if (annotation == null) {
            return true;
        }

        ClassLoader classLoader = context.getClassLoader();

        for (Class<?> clazz : annotation.value()) {
            try {
                Class.forName(clazz.getName(), false, classLoader);
            } catch (ClassNotFoundException e) {
                return false;
            }
        }

        for (String className : annotation.name()) {
            try {
                Class.forName(className, false, classLoader);
            } catch (ClassNotFoundException e) {
                return false;
            }
        }

        return true;
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/ConditionalOnMissingBean.java` [NEW]

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.context.annotation.Conditional;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnMissingBean {
    Class<?>[] value() default {};
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/OnBeanCondition.java` [NEW]

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.context.annotation.Condition;
import com.iris.framework.context.annotation.ConditionContext;

public class OnBeanCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnMissingBean annotation = element.getAnnotation(ConditionalOnMissingBean.class);
        if (annotation == null) {
            return true;
        }

        Class<?>[] types = annotation.value();

        if (types.length == 0 && element instanceof Method method) {
            types = new Class<?>[]{ method.getReturnType() };
        }

        if (types.length == 0) {
            return true;
        }

        BeanDefinitionRegistry registry = context.getRegistry();

        for (Class<?> type : types) {
            if (hasBeanOfType(registry, type)) {
                return false;
            }
        }

        return true;
    }

    private boolean hasBeanOfType(BeanDefinitionRegistry registry, Class<?> type) {
        for (String name : registry.getBeanDefinitionNames()) {
            BeanDefinition bd = registry.getBeanDefinition(name);
            if (type.isAssignableFrom(bd.getBeanClass())) {
                return true;
            }
        }
        return false;
    }
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/ConditionalOnProperty.java` [NEW]

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.context.annotation.Conditional;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {
    String[] value() default {};
    String prefix() default "";
    String[] name() default {};
    String havingValue() default "";
    boolean matchIfMissing() default false;
}
```

#### File: `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/condition/OnPropertyCondition.java` [NEW]

```java
package com.iris.boot.autoconfigure.condition;

import java.lang.reflect.AnnotatedElement;

import com.iris.framework.context.annotation.Condition;
import com.iris.framework.context.annotation.ConditionContext;
import com.iris.framework.core.env.Environment;

public class OnPropertyCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedElement element) {
        ConditionalOnProperty annotation = element.getAnnotation(ConditionalOnProperty.class);
        if (annotation == null) {
            return true;
        }

        Environment environment = context.getEnvironment();
        if (environment == null) {
            return false;
        }

        String prefix = annotation.prefix();
        String havingValue = annotation.havingValue();
        boolean matchIfMissing = annotation.matchIfMissing();

        String[] names = annotation.name().length > 0 ? annotation.name() : annotation.value();

        for (String name : names) {
            String key = prefix.isEmpty() ? name : prefix + "." + name;
            String value = environment.getProperty(key);

            if (value == null) {
                if (!matchIfMissing) {
                    return false;
                }
            } else {
                if (!havingValue.isEmpty()) {
                    if (!havingValue.equalsIgnoreCase(value)) {
                        return false;
                    }
                } else {
                    if ("false".equalsIgnoreCase(value)) {
                        return false;
                    }
                }
            }
        }

        return true;
    }
}
```

### Test Code

#### File: `iris-framework/src/test/java/com/iris/framework/context/annotation/ConditionEvaluatorTest.java` [NEW]

```java
package com.iris.framework.context.annotation;

import static org.assertj.core.api.Assertions.assertThat;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;

class ConditionEvaluatorTest {

    private DefaultBeanFactory factory;
    private ConditionEvaluator evaluator;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        evaluator = new ConditionEvaluator(factory, null);
    }

    // Test conditions and annotations omitted for brevity — see source file

    @Nested
    @DisplayName("Basic condition evaluation")
    class BasicEvaluation {
        @Test void shouldNotSkip_WhenNoConditions() { /* ... */ }
        @Test void shouldNotSkip_WhenDirectConditionMatches() { /* ... */ }
        @Test void shouldSkip_WhenDirectConditionDoesNotMatch() { /* ... */ }
    }

    @Nested
    @DisplayName("Meta-annotation support")
    class MetaAnnotationSupport {
        @Test void shouldNotSkip_WhenMetaConditionMatches() { /* ... */ }
        @Test void shouldSkip_WhenMetaConditionDoesNotMatch() { /* ... */ }
    }

    @Nested
    @DisplayName("Multiple conditions (ALL must match)")
    class MultipleConditions {
        @Test void shouldSkip_WhenOneOfMultipleDirectConditionsFails() { /* ... */ }
        @Test void shouldNotSkip_WhenAllDirectConditionsPass() { /* ... */ }
        @Test void shouldSkip_WhenOneOfMultipleMetaConditionsFails() { /* ... */ }
    }

    @Nested
    @DisplayName("Method-level conditions")
    class MethodLevelConditions {
        @Test void shouldNotSkipMethod_WhenConditionMatches() { /* ... */ }
        @Test void shouldSkipMethod_WhenConditionDoesNotMatch() { /* ... */ }
        @Test void shouldSkipMethod_WhenMetaConditionDoesNotMatch() { /* ... */ }
        @Test void shouldNotSkipMethod_WhenNoConditions() { /* ... */ }
    }

    @Nested
    @DisplayName("Integration with ConfigurationClassProcessor")
    class ProcessorIntegration {
        @Test void shouldSkipEntireConfig_WhenClassLevelConditionFails() { /* ... */ }
        @Test void shouldSkipBean_WhenMethodLevelConditionFails() { /* ... */ }
        @Test void shouldRegisterBeans_WhenClassLevelConditionPasses() { /* ... */ }
    }
}
```

(Full test source available at `iris-framework/src/test/java/com/iris/framework/context/annotation/ConditionEvaluatorTest.java`)

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnClassConditionTest.java` [NEW]

(Full test source available at `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnClassConditionTest.java`)

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnBeanConditionTest.java` [NEW]

(Full test source available at `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnBeanConditionTest.java`)

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnPropertyConditionTest.java` [NEW]

(Full test source available at `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/OnPropertyConditionTest.java`)

#### File: `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/integration/ConditionalBeansIntegrationTest.java` [NEW]

(Full test source available at `iris-boot-core/src/test/java/com/iris/boot/autoconfigure/condition/integration/ConditionalBeansIntegrationTest.java`)

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`Condition`** | A `@FunctionalInterface` with `matches(context, element)` — the SPI for custom conditions |
| **`@Conditional`** | Meta-annotation linking a class/method to one or more `Condition` implementations |
| **Meta-annotation pattern** | `@ConditionalOnClass` carries `@Conditional(OnClassCondition.class)` — the framework discovers Boot conditions without importing them |
| **`ConditionEvaluator`** | Collects conditions from direct + meta-annotations and evaluates them — `shouldSkip()` returns true if any condition fails |
| **`@ConditionalOnClass`** | Checks `Class.forName()` — guards beans behind classpath presence |
| **`@ConditionalOnMissingBean`** | Checks bean definitions by type — enables "back off if user provides their own" |
| **`@ConditionalOnProperty`** | Checks `Environment.getProperty()` — enables feature flags and property-based gating |

**Next: Chapter 13 — Auto-Configuration Engine** — Automatically discover and apply `@AutoConfiguration` classes listed in `META-INF/iris/AutoConfiguration.imports`, using the conditional beans system from this chapter to decide which auto-configurations actually take effect
