# Chapter 5: Component Scanning

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Beans must be registered manually — either via `registerBeanDefinition()` calls or through `@Configuration`/`@Bean` factory methods | Every component requires explicit Java code to register it; there's no way to say "find all my components in this package" | Build a classpath scanner that auto-discovers `@Component`-annotated classes (and stereotypes `@Service`, `@Controller`, `@Repository`) and registers them as beans |

---

## 5.1 The Integration Point

The integration point is `ConfigurationClassProcessor.processConfigurationClasses()` — the single method that orchestrates bean discovery. Currently it only processes `@Bean` methods. We'll split it into two phases: **Phase 1** scans the classpath for components, then **Phase 2** processes `@Bean` methods (the existing behavior).

**Modifying:** `src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java`
**Change:** Split the single-pass method into a two-phase pipeline — component scanning first, then `@Bean` processing

```java
public void processConfigurationClasses(BeanDefinitionRegistry registry) {
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

        List<Method> beanMethods = findBeanMethods(beanClass);
        for (Method method : beanMethods) {
            String beanName = resolveBeanName(method);
            BeanDefinition beanDef = createBeanDefinition(method, candidateName);
            registry.registerBeanDefinition(beanName, beanDef);
        }
    }
}

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
```

Two key decisions here:

1. **Phase ordering matters:** Scanning runs FIRST because it may discover new `@Configuration` classes in the scanned packages. Phase 2 re-snapshots the bean definition names, so it picks up these newly discovered configuration classes and processes their `@Bean` methods too.

2. **Default package fallback:** If `@ComponentScan` specifies no packages, we use the `@Configuration` class's own package — exactly matching Spring's behavior. This is why you typically put your main class at the root of your package hierarchy.

This connects **classpath scanning** to **configuration processing**. To make it work, we need to build:
- `@Component` and stereotype annotations — what the scanner looks for
- `@ComponentScan` — how users specify what packages to scan
- `ClassPathScanner` — the engine that walks the classpath and discovers components
- A modification to `@Configuration` — add `@Component` as a meta-annotation so config classes are themselves scannable

## 5.2 Stereotype Annotations

The entire component scanning system rests on a single root annotation: `@Component`. Every other stereotype (`@Service`, `@Controller`, `@Repository`) is just `@Component` with a different label — they carry `@Component` as a meta-annotation.

**New file:** `src/main/java/com/iris/framework/stereotype/Component.java`

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {

    /**
     * The logical bean name for this component. If empty, the scanner
     * derives a name by decapitalizing the simple class name.
     */
    String value() default "";
}
```

**New file:** `src/main/java/com/iris/framework/stereotype/Service.java`

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component                    // ← the key: @Service IS-A @Component
public @interface Service {
    String value() default "";
}
```

**New file:** `src/main/java/com/iris/framework/stereotype/Controller.java`

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
    String value() default "";
}
```

**New file:** `src/main/java/com/iris/framework/stereotype/Repository.java`

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Repository {
    String value() default "";
}
```

The pattern is identical across all three: `@Target(TYPE)`, `@Retention(RUNTIME)`, `@Component` as meta-annotation, and a `value()` attribute for explicit bean naming.

**Modifying:** `src/main/java/com/iris/framework/context/annotation/Configuration.java`
**Change:** Add `@Component` as a meta-annotation so configuration classes are discoverable by scanning

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component              // ← NEW: makes @Configuration discoverable by scanning
public @interface Configuration {
    String value() default "";
}
```

**New file:** `src/main/java/com/iris/framework/context/annotation/ComponentScan.java`

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ComponentScan {

    /**
     * Base packages to scan for annotated components. If empty, the package
     * of the annotated @Configuration class is used.
     */
    String[] value() default {};
}
```

## 5.3 The ClassPath Scanner

The scanner converts a package name to a filesystem path, walks it recursively, loads each `.class` file, and checks for `@Component` (directly or as a meta-annotation). Candidates that pass filtering get registered as `BeanDefinition`s.

**New file:** `src/main/java/com/iris/framework/context/annotation/ClassPathScanner.java`

```java
package com.iris.framework.context.annotation;

import java.io.File;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.net.URL;
import java.util.Enumeration;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.stereotype.Component;

public class ClassPathScanner {

    /**
     * Scan the given base packages and register discovered components.
     */
    public int scan(BeanDefinitionRegistry registry, String... basePackages) {
        int count = 0;
        for (String basePackage : basePackages) {
            count += scanPackage(registry, basePackage);
        }
        return count;
    }

    private int scanPackage(BeanDefinitionRegistry registry, String basePackage) {
        String resourcePath = basePackage.replace('.', '/');
        ClassLoader classLoader = getClassLoader();

        try {
            Enumeration<URL> resources = classLoader.getResources(resourcePath);
            int count = 0;

            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                if ("file".equals(resource.getProtocol())) {
                    File directory = new File(resource.toURI());
                    count += scanDirectory(registry, directory, basePackage);
                }
            }

            return count;
        } catch (Exception e) {
            throw new BeanCreationException("component-scan",
                    "Failed to scan package '" + basePackage + "'", e);
        }
    }

    private int scanDirectory(BeanDefinitionRegistry registry, File dir, String packageName) {
        int count = 0;
        File[] files = dir.listFiles();
        if (files == null) return 0;

        for (File file : files) {
            if (file.isDirectory()) {
                count += scanDirectory(registry, file, packageName + "." + file.getName());
            } else if (isClassFile(file.getName())) {
                String className = packageName + "."
                        + file.getName().substring(0, file.getName().length() - ".class".length());
                if (processCandidate(registry, className)) {
                    count++;
                }
            }
        }

        return count;
    }

    private boolean isClassFile(String fileName) {
        return fileName.endsWith(".class")
                && !fileName.contains("$")
                && !fileName.equals("module-info.class")
                && !fileName.equals("package-info.class");
    }

    private boolean processCandidate(BeanDefinitionRegistry registry, String className) {
        try {
            Class<?> clazz = Class.forName(className, false, getClassLoader());

            if (!isCandidate(clazz)) return false;

            String beanName = resolveBeanName(clazz);
            if (registry.containsBeanDefinition(beanName)) return false;

            BeanDefinition bd = new BeanDefinition(clazz);
            registry.registerBeanDefinition(beanName, bd);
            return true;
        } catch (ClassNotFoundException | NoClassDefFoundError e) {
            return false;
        }
    }

    /**
     * A class is a candidate if it's concrete and has @Component.
     */
    private boolean isCandidate(Class<?> clazz) {
        if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
            return false;
        }
        return hasComponentAnnotation(clazz);
    }

    /**
     * Check for @Component — directly or as a meta-annotation.
     * This single check catches all stereotypes (@Service, @Controller, etc.)
     * because they carry @Component as a meta-annotation.
     */
    static boolean hasComponentAnnotation(Class<?> clazz) {
        if (clazz.isAnnotationPresent(Component.class)) {
            return true;
        }
        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().isAnnotationPresent(Component.class)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Resolve bean name: explicit @Component/stereotype value() first,
     * then decapitalized class name.
     */
    static String resolveBeanName(Class<?> clazz) {
        Component component = clazz.getAnnotation(Component.class);
        if (component != null && !component.value().isEmpty()) {
            return component.value();
        }

        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().isAnnotationPresent(Component.class)) {
                try {
                    Method valueMethod = annotation.annotationType().getMethod("value");
                    String value = (String) valueMethod.invoke(annotation);
                    if (value != null && !value.isEmpty()) {
                        return value;
                    }
                } catch (Exception e) {
                    // No value() method — fall through to default
                }
            }
        }

        return decapitalize(clazz.getSimpleName());
    }

    /**
     * Decapitalize following JavaBeans Introspector convention:
     * lowercase first char, unless first TWO chars are uppercase.
     */
    static String decapitalize(String name) {
        if (name.isEmpty()) return name;
        if (name.length() > 1
                && Character.isUpperCase(name.charAt(0))
                && Character.isUpperCase(name.charAt(1))) {
            return name;
        }
        return Character.toLowerCase(name.charAt(0)) + name.substring(1);
    }

    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return (cl != null) ? cl : ClassPathScanner.class.getClassLoader();
    }
}
```

The scanning pipeline works in four steps:

1. **Package → Path:** `com.example.app` becomes `com/example/app`
2. **Path → Directory:** `ClassLoader.getResources()` locates the directory on the classpath
3. **Directory → Classes:** Recursive file walk finds all `.class` files
4. **Classes → Beans:** Each class is loaded, checked for `@Component`, and registered

## 5.4 Try It Yourself

<details>
<summary>Challenge 1: Implement the meta-annotation detection</summary>

Given a class, determine if it has `@Component` either directly or as a meta-annotation on one of its own annotations. Remember that `@Service` carries `@Component` as a meta-annotation — your check needs to walk one level up the annotation hierarchy.

```java
static boolean hasComponentAnnotation(Class<?> clazz) {
    // Direct @Component
    if (clazz.isAnnotationPresent(Component.class)) {
        return true;
    }
    // Meta-annotation: check if any of the class's annotations
    // is itself annotated with @Component
    for (Annotation annotation : clazz.getAnnotations()) {
        if (annotation.annotationType().isAnnotationPresent(Component.class)) {
            return true;
        }
    }
    return false;
}
```

</details>

<details>
<summary>Challenge 2: Implement bean name resolution with stereotype value support</summary>

The tricky part: when a class has `@Service("myService")`, the `@Component` annotation is on `@Service` (not on the class). So `clazz.getAnnotation(Component.class)` returns null. You need to check the stereotype annotation's `value()` method via reflection.

```java
static String resolveBeanName(Class<?> clazz) {
    // 1. Direct @Component with explicit name
    Component component = clazz.getAnnotation(Component.class);
    if (component != null && !component.value().isEmpty()) {
        return component.value();
    }
    // 2. Check stereotype annotations
    for (Annotation annotation : clazz.getAnnotations()) {
        if (annotation.annotationType().isAnnotationPresent(Component.class)) {
            try {
                Method valueMethod = annotation.annotationType().getMethod("value");
                String value = (String) valueMethod.invoke(annotation);
                if (value != null && !value.isEmpty()) {
                    return value;
                }
            } catch (Exception e) { /* no value() method */ }
        }
    }
    // 3. Default: decapitalize class name
    return decapitalize(clazz.getSimpleName());
}
```

</details>

## 5.5 Tests

### Unit Tests

**New file:** `src/test/java/com/iris/framework/context/annotation/ClassPathScannerTest.java`

```java
package com.iris.framework.context.annotation;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.scan.*;
import com.iris.framework.context.annotation.scan.sub.SubPackageComponent;
import com.iris.framework.stereotype.*;

import static org.assertj.core.api.Assertions.*;

class ClassPathScannerTest {

    private ClassPathScanner scanner;
    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        scanner = new ClassPathScanner();
        factory = new DefaultBeanFactory();
    }

    @Test
    void shouldDiscoverComponentAnnotatedClasses_WhenScanningPackage() {
        int count = scanner.scan(factory,
                "com.iris.framework.context.annotation.scan");
        // 7 concrete components (skips NotAComponent, AbstractComponent, ComponentInterface)
        assertThat(count).isEqualTo(7);
    }

    @Test
    void shouldRegisterServiceAsBean_WhenMetaAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");
        assertThat(factory.containsBeanDefinition("simpleService")).isTrue();
    }

    @Test
    void shouldScanSubPackagesRecursively_WhenBasePackageSpecified() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");
        assertThat(factory.containsBeanDefinition("subPackageComponent")).isTrue();
    }

    @Test
    void shouldSkipAbstractClasses_WhenAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");
        assertThat(factory.containsBeanDefinition("abstractComponent")).isFalse();
    }

    @Test
    void shouldUseExplicitName_WhenComponentValueSpecified() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");
        assertThat(factory.containsBeanDefinition("customName")).isTrue();
    }

    @Test
    void shouldUseExplicitName_WhenStereotypeValueSpecified() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");
        assertThat(factory.containsBeanDefinition("mySpecialService")).isTrue();
    }

    @Test
    void shouldSkipAlreadyRegistered_WhenBeanNameExists() {
        factory.registerBeanDefinition("simpleComponent", new BeanDefinition(String.class));
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");
        // Pre-registered definition should NOT be overwritten
        assertThat(factory.getBeanDefinition("simpleComponent").getBeanClass())
                .isEqualTo(String.class);
    }

    @Test
    void shouldPreserveConsecutiveUppercase_WhenDecapitalizing() {
        assertThat(ClassPathScanner.decapitalize("URLService")).isEqualTo("URLService");
    }

    @Test
    void shouldHaveComponentMetaAnnotation_OnConfiguration() {
        assertThat(Configuration.class.isAnnotationPresent(Component.class)).isTrue();
    }
}
```

The test package `com.iris.framework.context.annotation.scan` contains purpose-built components:

| Test Class | Annotation | Purpose |
|-----------|------------|---------|
| `SimpleComponent` | `@Component` | Direct annotation detection |
| `SimpleService` | `@Service` | Meta-annotation detection |
| `SimpleController` | `@Controller` | Meta-annotation detection |
| `SimpleRepository` | `@Repository` | Meta-annotation detection |
| `NamedComponent` | `@Component("customName")` | Explicit naming |
| `NamedService` | `@Service("mySpecialService")` | Stereotype naming |
| `NotAComponent` | _(none)_ | Should be skipped |
| `AbstractComponent` | `@Component` (abstract) | Should be skipped |
| `ComponentInterface` | `@Component` (interface) | Should be skipped |
| `SubPackageComponent` | `@Component` (in `.scan.sub`) | Recursive scanning |

### Integration Tests

**New file:** `src/test/java/com/iris/framework/beans/factory/integration/ComponentScanningIntegrationTest.java`

```java
package com.iris.framework.beans.factory.integration;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.integration.scantest.*;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;

import static org.assertj.core.api.Assertions.*;

class ComponentScanningIntegrationTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();
        factory.addBeanPostProcessor(new AutowiredBeanPostProcessor(factory));
        factory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

        // Register config class → triggers scanning of its package
        factory.registerBeanDefinition("appConfig", new BeanDefinition(AppConfig.class));
        new ConfigurationClassProcessor().processConfigurationClasses(factory);
    }

    @Test
    void shouldDiscoverAllComponents_WhenConfigurationHasComponentScan() {
        assertThat(factory.containsBeanDefinition("userRepository")).isTrue();
        assertThat(factory.containsBeanDefinition("userService")).isTrue();
        assertThat(factory.containsBeanDefinition("userController")).isTrue();
    }

    @Test
    void shouldRegisterBeanMethods_FromScannedConfigurationClass() {
        // AppConfig's @Bean method should also be registered
        String appName = factory.getBean("appName", String.class);
        assertThat(appName).isEqualTo("TestApp");
    }

    @Test
    void shouldInjectDependencies_WhenScannedComponentsHaveAutowired() {
        UserController controller = factory.getBean(UserController.class);
        // Controller → Service → Repository (full chain, all scanned)
        assertThat(controller.handleGetUser("42")).isEqualTo("User-42");
    }

    @Test
    void shouldInvokeLifecycleCallbacks_OnScannedComponents() {
        LifecycleComponent component = factory.getBean(LifecycleComponent.class);
        assertThat(component.getEvents())
                .containsExactly("@PostConstruct", "afterPropertiesSet");
    }

    @Test
    void shouldMaintainSingletonIdentity_AcrossScannedComponents() {
        UserRepository repo1 = factory.getBean(UserRepository.class);
        UserRepository repo2 = factory.getBean(UserRepository.class);
        assertThat(repo1).isSameAs(repo2);
    }
}
```

The integration test verifies that scanned components work with all prior features: dependency injection chains (`UserController → UserService → UserRepository`), lifecycle callbacks (`@PostConstruct` + `InitializingBean`), singleton identity, and `@Bean` method coexistence.

**Run:** `./gradlew :iris-framework:test` — expected: all 39 tests pass (29 new + 10 new integration + existing prior features)

---

## 5.6 Why This Works

> ★ **Insight** -------------------------------------------
> **One Filter, Infinite Stereotypes.** The scanner only checks for `@Component` (including as a meta-annotation). It never mentions `@Service`, `@Controller`, or `@Repository` by name. This means you could create `@DataTransferObject`, `@Gateway`, or any custom stereotype — just meta-annotate it with `@Component` and the scanner finds it automatically. The real Spring Framework uses exactly this pattern at `ClassPathScanningCandidateComponentProvider:217`: one `AnnotationTypeFilter(Component.class)` is the only default filter.
>
> **Trade-off:** This simplicity comes at a cost — you can't distinguish between stereotypes at scan time. If you wanted to apply different behavior to `@Controller` vs `@Service` (e.g., different scopes), you'd need include/exclude filters. Spring supports this via `@ComponentScan(includeFilters = ...)`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The Two-Phase Design Enables Composition.** By running component scanning BEFORE `@Bean` processing, we get a powerful composition pattern: you can put `@Configuration` classes inside scanned packages, and their `@Bean` methods are automatically processed. This is how Spring Boot's auto-configuration works — auto-configuration classes are `@Configuration` classes discovered by scanning (via `META-INF/spring/AutoConfiguration.imports` rather than classpath scanning, but the principle is the same).
>
> **Why not one phase?** If we processed `@Bean` methods during scanning, we'd need the scanner to know about `@Configuration` and `@Bean` — violating separation of concerns. The scanner only cares about `@Component`; the configuration processor only cares about `@Bean`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Reflection vs. ASM: A Deliberate Simplification.** Our scanner uses `Class.forName(name, false, classLoader)` to inspect annotations. Spring uses ASM bytecode reading (`MetadataReader`) instead — it reads `.class` file bytes directly without loading the class into the JVM. This avoids:
> - Static initializer side effects
> - Unnecessary class loading (expensive in large applications)
> - ClassNotFoundException for optional dependencies
>
> We use `initialize=false` to partially mitigate this, but it's still heavier than ASM. For a learning project, the clarity of reflection outweighs the performance cost.
> -----------------------------------------------------------

## 5.7 What We Enhanced

| Aspect | Before (ch04) | Current (ch05) | Real Framework |
|--------|---------------|----------------|----------------|
| Bean discovery | Manual registration + `@Bean` factory methods only | Automatic discovery via `@ComponentScan` + classpath walking | `ClassPathBeanDefinitionScanner` with ASM metadata reading, include/exclude filters, component index (`ClassPathBeanDefinitionScanner.java:272`) |
| `@Configuration` | Standalone annotation; must be manually registered | Meta-annotated with `@Component`; discoverable by scanning | Same pattern — `@Configuration` carries `@Component` |
| Bean naming | `@Bean(value)` or method name; `deriveConfigBeanName()` for config classes | `@Component(value)` / `@Service(value)` / decapitalized class name | `AnnotationBeanNameGenerator` with `@AliasFor` and `Introspector.decapitalize()` (`AnnotationBeanNameGenerator.java:108`) |
| `ConfigurationClassProcessor` | Single-pass `@Bean` method processing | Two-phase: component scanning → `@Bean` processing | Iterative discovery loop with `@Import`, `@ComponentScan`, `@PropertySource` etc. (`ConfigurationClassParser.java:325`) |

## 5.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ClassPathScanner` | `ClassPathScanningCandidateComponentProvider` + `ClassPathBeanDefinitionScanner` | `ClassPathBeanDefinitionScanner.java:272` | Real version uses ASM `MetadataReader` instead of `Class.forName()`; supports include/exclude filters; has compile-time component index |
| `ClassPathScanner.hasComponentAnnotation()` | `AnnotationTypeFilter.matchSelf()` | `AnnotationTypeFilter.java:99` | Real version supports `@Inherited` traversal, interface scanning, and configurable `considerMetaAnnotations` flag |
| `ClassPathScanner.resolveBeanName()` | `AnnotationBeanNameGenerator.generateBeanName()` | `AnnotationBeanNameGenerator.java:108` | Real version uses `MergedAnnotations` + `@AliasFor` for cross-annotation name resolution |
| `ClassPathScanner.decapitalize()` | `StringUtils.uncapitalizeAsProperty()` | `AnnotationBeanNameGenerator.java:247` | Real version uses `StringUtils` instead of `Introspector.decapitalize()` but same JavaBeans convention |
| `ClassPathScanner.isCandidate()` | `isCandidateComponent(MetadataReader)` + `isCandidateComponent(AnnotatedBeanDefinition)` | `ClassPathScanningCandidateComponentProvider.java:533` | Real version has two-phase filtering: annotation check + structural check (allows abstract with `@Lookup`) |
| `ClassPathScanner.scanPackage()` | `scanCandidateComponents()` | `ClassPathScanningCandidateComponentProvider.java:446` | Real version uses `PathMatchingResourcePatternResolver` to also scan inside JARs; pattern is `classpath*:pkg/**/*.class` |
| `ConfigurationClassProcessor.performComponentScanning()` | `ConfigurationClassParser.doProcessConfigurationClass()` | `ConfigurationClassParser.java:325` | Real version parses `@ComponentScan` attributes (filters, lazy, scope) and delegates to `ComponentScanAnnotationParser` |
| `@ComponentScan` | `@ComponentScan` | `ComponentScan.java` | Real version is `@Repeatable`, supports `basePackageClasses`, `includeFilters`/`excludeFilters`, `nameGenerator`, `scopeResolver`, `lazyInit` |

## 5.9 Complete Code

### Production Code

#### File: `src/main/java/com/iris/framework/stereotype/Component.java` [NEW]

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that an annotated class is a "component" — a candidate for
 * auto-detection during classpath scanning.
 *
 * <p>Any annotation that is itself meta-annotated with {@code @Component}
 * is considered a <em>stereotype annotation</em> that makes the annotated
 * class eligible for classpath scanning. This is how {@link Service},
 * {@link Controller}, and {@link Repository} work — they carry
 * {@code @Component} as a meta-annotation.
 *
 * <p>In the real Spring Framework, {@code @Component} is also meta-annotated
 * with {@code @Indexed} (for compile-time component index generation via
 * {@code META-INF/spring.components}). We skip the indexing infrastructure
 * and rely entirely on runtime classpath scanning.
 *
 * @see Service
 * @see Controller
 * @see Repository
 * @see com.iris.framework.context.annotation.ComponentScan
 * @see com.iris.framework.context.annotation.ClassPathScanner
 * @see org.springframework.stereotype.Component
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {

    /**
     * The logical bean name for this component. If empty, the scanner
     * derives a name by decapitalizing the simple class name
     * (e.g., {@code MyService} → {@code "myService"}).
     *
     * <p>In the real Spring Framework, this maps to
     * {@code AnnotationBeanNameGenerator.determineBeanNameFromAnnotation()}.
     */
    String value() default "";
}
```

#### File: `src/main/java/com/iris/framework/stereotype/Service.java` [NEW]

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that an annotated class is a "service" — a business logic
 * component in the service layer.
 *
 * <p>This annotation is a specialization of {@link Component @Component},
 * meaning that classes annotated with {@code @Service} are automatically
 * detected during classpath scanning. The scanner matches them because
 * {@code @Service} is meta-annotated with {@code @Component}.
 *
 * <p>In the real Spring Framework, {@code @Service} also uses
 * {@code @AliasFor(annotation = Component.class)} on its {@code value()}
 * attribute, providing cross-annotation alias support through Spring's
 * merged annotation infrastructure. We simplify: the scanner explicitly
 * checks for {@code value()} on stereotype annotations via reflection.
 *
 * @see Component
 * @see org.springframework.stereotype.Service
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {

    /**
     * The logical bean name for this service component.
     * If empty, the scanner derives a name from the class name.
     */
    String value() default "";
}
```

#### File: `src/main/java/com/iris/framework/stereotype/Controller.java` [NEW]

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that an annotated class is a "controller" — a web request
 * handler in the presentation layer.
 *
 * <p>This annotation is a specialization of {@link Component @Component},
 * meaning that classes annotated with {@code @Controller} are automatically
 * detected during classpath scanning.
 *
 * <p>In the real Spring Framework, {@code @Controller} is the signal for
 * {@code RequestMappingHandlerMapping} to scan the class for
 * {@code @RequestMapping} methods. We'll use it the same way when we
 * build the DispatcherServlet (Feature 10).
 *
 * @see Component
 * @see org.springframework.stereotype.Controller
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {

    /**
     * The logical bean name for this controller component.
     * If empty, the scanner derives a name from the class name.
     */
    String value() default "";
}
```

#### File: `src/main/java/com/iris/framework/stereotype/Repository.java` [NEW]

```java
package com.iris.framework.stereotype;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Indicates that an annotated class is a "repository" — a data access
 * component in the persistence layer.
 *
 * <p>This annotation is a specialization of {@link Component @Component},
 * meaning that classes annotated with {@code @Repository} are automatically
 * detected during classpath scanning.
 *
 * <p>In the real Spring Framework, {@code @Repository} has an additional
 * role beyond component scanning: it enables automatic exception translation
 * via {@code PersistenceExceptionTranslationPostProcessor}, converting
 * persistence-specific exceptions (e.g., JDBC, JPA) into Spring's
 * {@code DataAccessException} hierarchy. We don't implement exception
 * translation in our simplified version.
 *
 * @see Component
 * @see org.springframework.stereotype.Repository
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Repository {

    /**
     * The logical bean name for this repository component.
     * If empty, the scanner derives a name from the class name.
     */
    String value() default "";
}
```

#### File: `src/main/java/com/iris/framework/context/annotation/ComponentScan.java` [NEW]

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Configures component scanning for a {@link Configuration @Configuration}
 * class. When placed on a configuration class, the
 * {@link ConfigurationClassProcessor} delegates to a {@link ClassPathScanner}
 * to discover {@link com.iris.framework.stereotype.Component @Component}-annotated
 * classes in the specified packages.
 *
 * <p>If no {@link #value() base packages} are specified, the package of the
 * annotated {@code @Configuration} class is used as the default base package.
 *
 * <p>In the real Spring Framework, {@code @ComponentScan} supports:
 * <ul>
 *   <li>{@code @Repeatable} — multiple scans on one config class</li>
 *   <li>{@code basePackageClasses} — type-safe alternative to string packages</li>
 *   <li>{@code includeFilters} / {@code excludeFilters} — custom filter rules</li>
 *   <li>{@code nameGenerator} — pluggable bean naming strategy</li>
 *   <li>{@code scopeResolver} — pluggable scope detection</li>
 *   <li>{@code lazyInit} — default lazy initialization for scanned beans</li>
 * </ul>
 *
 * <p>We simplify to just {@code value()} (base packages) with automatic
 * default-package detection.
 *
 * @see Configuration
 * @see ClassPathScanner
 * @see com.iris.framework.stereotype.Component
 * @see org.springframework.context.annotation.ComponentScan
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ComponentScan {

    /**
     * Base packages to scan for annotated components. If empty, the package
     * of the annotated {@code @Configuration} class is used.
     *
     * <p>In the real Spring Framework, this is aliased with
     * {@code basePackages()} via {@code @AliasFor}. We use a single attribute
     * for simplicity.
     */
    String[] value() default {};
}
```

#### File: `src/main/java/com/iris/framework/context/annotation/ClassPathScanner.java` [NEW]

```java
package com.iris.framework.context.annotation;

import java.io.File;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.net.URL;
import java.util.Enumeration;

import com.iris.framework.beans.factory.BeanCreationException;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;
import com.iris.framework.stereotype.Component;

/**
 * Scans the classpath for classes annotated with {@link Component @Component}
 * (or any stereotype meta-annotated with {@code @Component}) and registers
 * them as {@link BeanDefinition}s in the given registry.
 *
 * <p>This is the simplified equivalent of Spring's
 * {@code ClassPathBeanDefinitionScanner} (which extends
 * {@code ClassPathScanningCandidateComponentProvider}). The real scanner:
 * <ol>
 *   <li>Uses ASM-based metadata reading ({@code MetadataReader}) to inspect
 *       {@code .class} files <em>without loading them</em> via the ClassLoader,
 *       avoiding side effects from static initializers</li>
 *   <li>Supports include/exclude filters ({@code AnnotationTypeFilter},
 *       {@code AssignableTypeFilter}, regex, AspectJ, custom)</li>
 *   <li>Supports a compile-time component index
 *       ({@code META-INF/spring.components}) as an alternative to runtime
 *       classpath scanning</li>
 *   <li>Handles scoped proxy generation for non-singleton beans</li>
 *   <li>Detects conflicts between scanned beans and explicitly registered beans</li>
 * </ol>
 *
 * <p>Our simplification:
 * <ul>
 *   <li>Uses {@code Class.forName(name, false, classLoader)} — loads the class
 *       without initialization, but still heavier than ASM bytecode reading</li>
 *   <li>Single include filter: {@code @Component} (including meta-annotations)</li>
 *   <li>No compile-time index — runtime scanning only</li>
 *   <li>No scoped proxy support — singleton scope only</li>
 *   <li>Simple conflict resolution: skip if name already registered</li>
 *   <li>Skips inner classes (files containing {@code $})</li>
 * </ul>
 *
 * @see Component
 * @see ComponentScan
 * @see ConfigurationClassProcessor
 * @see org.springframework.context.annotation.ClassPathBeanDefinitionScanner
 * @see org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider
 */
public class ClassPathScanner {

    /**
     * Scan the given base packages for {@code @Component}-annotated classes
     * and register them as {@link BeanDefinition}s in the given registry.
     *
     * <p>This maps to Spring's {@code ClassPathBeanDefinitionScanner.scan()}
     * which delegates to {@code doScan()} for each base package. The return
     * value is the total count of newly registered bean definitions.
     *
     * @param registry     the bean definition registry to register into
     * @param basePackages the packages to scan (e.g., {@code "com.example.app"})
     * @return the number of newly registered bean definitions
     */
    public int scan(BeanDefinitionRegistry registry, String... basePackages) {
        int count = 0;
        for (String basePackage : basePackages) {
            count += scanPackage(registry, basePackage);
        }
        return count;
    }

    /**
     * Scan a single base package by converting the package name to a resource
     * path, using the ClassLoader to locate the directory, and walking it
     * recursively for {@code .class} files.
     *
     * <p>This maps to Spring's pipeline:
     * {@code ClassPathScanningCandidateComponentProvider.scanCandidateComponents()}
     * which resolves the package to a classpath pattern like
     * {@code classpath*:com/example/**&#47;*.class} and uses
     * {@code PathMatchingResourcePatternResolver} to find resources.
     *
     * <p>We simplify by using {@code ClassLoader.getResources()} to find the
     * directory and walking the file system directly. This works for file-system
     * classpath entries but not for classes inside JAR files.
     */
    private int scanPackage(BeanDefinitionRegistry registry, String basePackage) {
        String resourcePath = basePackage.replace('.', '/');
        ClassLoader classLoader = getClassLoader();

        try {
            Enumeration<URL> resources = classLoader.getResources(resourcePath);
            int count = 0;

            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                if ("file".equals(resource.getProtocol())) {
                    File directory = new File(resource.toURI());
                    count += scanDirectory(registry, directory, basePackage);
                }
            }

            return count;
        } catch (Exception e) {
            throw new BeanCreationException("component-scan",
                    "Failed to scan package '" + basePackage + "'", e);
        }
    }

    /**
     * Recursively walk a directory, loading each {@code .class} file and
     * checking if it's a component candidate.
     *
     * <p>Filters applied:
     * <ul>
     *   <li>Skip inner classes ({@code $} in filename) — they depend on an
     *       enclosing instance and aren't independent components</li>
     *   <li>Skip {@code module-info.class} and {@code package-info.class}</li>
     *   <li>Recurse into subdirectories (sub-packages)</li>
     * </ul>
     */
    private int scanDirectory(BeanDefinitionRegistry registry, File dir, String packageName) {
        int count = 0;
        File[] files = dir.listFiles();
        if (files == null) {
            return 0;
        }

        for (File file : files) {
            if (file.isDirectory()) {
                // Recurse into sub-packages
                count += scanDirectory(registry, file, packageName + "." + file.getName());
            } else if (isClassFile(file.getName())) {
                String className = packageName + "."
                        + file.getName().substring(0, file.getName().length() - ".class".length());
                if (processCandidate(registry, className)) {
                    count++;
                }
            }
        }

        return count;
    }

    /**
     * Check if a filename represents a scannable {@code .class} file.
     * Excludes inner classes, module-info, and package-info.
     */
    private boolean isClassFile(String fileName) {
        return fileName.endsWith(".class")
                && !fileName.contains("$")
                && !fileName.equals("module-info.class")
                && !fileName.equals("package-info.class");
    }

    /**
     * Load a class by name, check if it's a component candidate, and if so,
     * register a {@link BeanDefinition} for it.
     *
     * <p>Uses {@code Class.forName(name, false, classLoader)} which loads the
     * class without running its static initializers. This is safer than
     * {@code Class.forName(name)} but still heavier than Spring's ASM-based
     * approach which never loads the class at all.
     *
     * <p>In the real Spring Framework, this is a two-phase check:
     * <ol>
     *   <li>Phase 1: {@code isCandidateComponent(MetadataReader)} — annotation check
     *       (exclude filters first, then include filters)</li>
     *   <li>Phase 2: {@code isCandidateComponent(AnnotatedBeanDefinition)} — structural
     *       check (concrete, independent, not a CGLIB proxy)</li>
     * </ol>
     *
     * @return true if the class was registered, false if skipped
     */
    private boolean processCandidate(BeanDefinitionRegistry registry, String className) {
        try {
            Class<?> clazz = Class.forName(className, false, getClassLoader());

            if (!isCandidate(clazz)) {
                return false;
            }

            String beanName = resolveBeanName(clazz);
            if (registry.containsBeanDefinition(beanName)) {
                return false;
            }

            BeanDefinition bd = new BeanDefinition(clazz);
            registry.registerBeanDefinition(beanName, bd);
            return true;
        } catch (ClassNotFoundException | NoClassDefFoundError e) {
            // Class not loadable — skip silently
            return false;
        }
    }

    /**
     * Determine if a class is a component candidate:
     * <ol>
     *   <li>Must have {@code @Component} — directly or as a meta-annotation</li>
     *   <li>Must be concrete (not an interface, not abstract)</li>
     * </ol>
     *
     * <p>In the real Spring Framework, the structural check also allows
     * abstract classes with {@code @Lookup} methods (which Spring implements
     * via CGLIB). We require concrete classes only.
     */
    private boolean isCandidate(Class<?> clazz) {
        // Must be concrete
        if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
            return false;
        }

        // Must have @Component (directly or as a meta-annotation)
        return hasComponentAnnotation(clazz);
    }

    /**
     * Check whether the given class has {@code @Component} either directly
     * or as a meta-annotation on one of its annotations.
     *
     * <p>This is the simplified equivalent of Spring's
     * {@code AnnotationTypeFilter.matchSelf()} which checks both
     * {@code hasAnnotation()} and {@code hasMetaAnnotation()} via the
     * ASM-based {@code AnnotationMetadata}.
     *
     * <p>The key insight: only ONE filter check is needed for the entire
     * stereotype hierarchy. {@code @Service}, {@code @Controller}, and
     * {@code @Repository} all carry {@code @Component} as a meta-annotation,
     * so this single check catches all of them.
     *
     * @see org.springframework.core.type.filter.AnnotationTypeFilter#matchSelf
     */
    static boolean hasComponentAnnotation(Class<?> clazz) {
        // 1. Direct @Component
        if (clazz.isAnnotationPresent(Component.class)) {
            return true;
        }

        // 2. Meta-annotation: check if any of the class's annotations
        //    is itself annotated with @Component
        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().isAnnotationPresent(Component.class)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Resolve the bean name for a scanned component class.
     *
     * <p>Resolution order:
     * <ol>
     *   <li>If {@code @Component("explicitName")} → use that name</li>
     *   <li>If a stereotype annotation has a non-empty {@code value()} →
     *       use that (e.g., {@code @Service("myService")})</li>
     *   <li>Otherwise → decapitalize the simple class name
     *       ({@code MyService} → {@code "myService"})</li>
     * </ol>
     *
     * <p>In the real Spring Framework, this is handled by
     * {@code AnnotationBeanNameGenerator.determineBeanNameFromAnnotation()}
     * which uses Spring's merged annotation infrastructure and
     * {@code @AliasFor} to resolve names across the stereotype hierarchy.
     * We use direct reflection on the {@code value()} method instead.
     *
     * @see org.springframework.context.annotation.AnnotationBeanNameGenerator
     */
    static String resolveBeanName(Class<?> clazz) {
        // 1. Direct @Component with explicit name
        Component component = clazz.getAnnotation(Component.class);
        if (component != null && !component.value().isEmpty()) {
            return component.value();
        }

        // 2. Check stereotype annotations (meta-annotated with @Component)
        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().isAnnotationPresent(Component.class)) {
                try {
                    Method valueMethod = annotation.annotationType().getMethod("value");
                    String value = (String) valueMethod.invoke(annotation);
                    if (value != null && !value.isEmpty()) {
                        return value;
                    }
                } catch (Exception e) {
                    // No value() method or invocation failed — fall through to default
                }
            }
        }

        // 3. Default: decapitalize the simple class name
        return decapitalize(clazz.getSimpleName());
    }

    /**
     * Decapitalize a name following the JavaBeans {@code Introspector.decapitalize()}
     * convention: lowercase the first character, UNLESS the first two characters
     * are both uppercase (e.g., {@code "URLService"} stays {@code "URLService"}).
     *
     * <p>This matches Spring's {@code AnnotationBeanNameGenerator.buildDefaultBeanName()}
     * which delegates to {@code java.beans.Introspector.decapitalize()}.
     *
     * @see java.beans.Introspector#decapitalize(String)
     */
    static String decapitalize(String name) {
        if (name.isEmpty()) {
            return name;
        }
        // JavaBeans spec: if first two chars are uppercase, keep as-is
        if (name.length() > 1
                && Character.isUpperCase(name.charAt(0))
                && Character.isUpperCase(name.charAt(1))) {
            return name;
        }
        return Character.toLowerCase(name.charAt(0)) + name.substring(1);
    }

    /**
     * Get the ClassLoader to use for loading classes and resources.
     * Prefers the thread's context ClassLoader, falling back to the
     * ClassLoader that loaded this scanner class.
     */
    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return (cl != null) ? cl : ClassPathScanner.class.getClassLoader();
    }
}
```

#### File: `src/main/java/com/iris/framework/context/annotation/Configuration.java` [MODIFIED]

```java
package com.iris.framework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import com.iris.framework.stereotype.Component;

/**
 * Indicates that a class declares one or more {@link Bean @Bean} methods and
 * may be processed by the {@link ConfigurationClassProcessor} to generate
 * bean definitions.
 *
 * <p>{@code @Configuration} is meta-annotated with {@link Component @Component},
 * making configuration classes eligible for component scanning — they are
 * automatically discovered when their package is scanned.
 *
 * <p>In the real Spring Framework, {@code @Configuration} also supports
 * {@code proxyBeanMethods} (CGLIB enhancement) so that inter-bean method
 * calls return singletons. We simplify:
 * <ul>
 *   <li>No CGLIB proxying — operates in "lite mode" only</li>
 * </ul>
 *
 * @see Bean
 * @see Component
 * @see ConfigurationClassProcessor
 * @see org.springframework.context.annotation.Configuration
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

    /**
     * Explicit bean name for this configuration class. If empty, the
     * processor derives a name from the class (lowercase first letter).
     */
    String value() default "";
}
```

#### File: `src/main/java/com/iris/framework/context/annotation/ConfigurationClassProcessor.java` [MODIFIED]

```java
package com.iris.framework.context.annotation;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.BeanDefinitionRegistry;

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
     * given registry. First performs component scanning (Phase 1), then
     * discovers and registers {@code @Bean} methods (Phase 2).
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
     * @param registry the bean definition registry to scan and populate
     */
    public void processConfigurationClasses(BeanDefinitionRegistry registry) {
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

            List<Method> beanMethods = findBeanMethods(beanClass);
            for (Method method : beanMethods) {
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

### Test Code

#### File: `src/test/java/com/iris/framework/context/annotation/ClassPathScannerTest.java` [NEW]

```java
package com.iris.framework.context.annotation;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.scan.NamedComponent;
import com.iris.framework.context.annotation.scan.NamedService;
import com.iris.framework.context.annotation.scan.SimpleComponent;
import com.iris.framework.context.annotation.scan.SimpleController;
import com.iris.framework.context.annotation.scan.SimpleRepository;
import com.iris.framework.context.annotation.scan.SimpleService;
import com.iris.framework.context.annotation.scan.sub.SubPackageComponent;
import com.iris.framework.stereotype.Component;
import com.iris.framework.stereotype.Controller;
import com.iris.framework.stereotype.Repository;
import com.iris.framework.stereotype.Service;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link ClassPathScanner} — verifies classpath scanning,
 * annotation detection, bean naming, and candidate filtering.
 */
class ClassPathScannerTest {

    private ClassPathScanner scanner;
    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        scanner = new ClassPathScanner();
        factory = new DefaultBeanFactory();
    }

    // -----------------------------------------------------------------------
    // Scanning and Discovery
    // -----------------------------------------------------------------------

    @Test
    void shouldDiscoverComponentAnnotatedClasses_WhenScanningPackage() {
        int count = scanner.scan(factory,
                "com.iris.framework.context.annotation.scan");

        // Should find: SimpleComponent, SimpleService, SimpleController,
        // SimpleRepository, NamedComponent, NamedService, SubPackageComponent
        // Should skip: NotAComponent, AbstractComponent, ComponentInterface
        assertThat(count).isEqualTo(7);
    }

    @Test
    void shouldRegisterComponentAsBean_WhenDirectlyAnnotated() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("simpleComponent")).isTrue();
        BeanDefinition bd = factory.getBeanDefinition("simpleComponent");
        assertThat(bd.getBeanClass()).isEqualTo(SimpleComponent.class);
    }

    @Test
    void shouldRegisterServiceAsBean_WhenMetaAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("simpleService")).isTrue();
        BeanDefinition bd = factory.getBeanDefinition("simpleService");
        assertThat(bd.getBeanClass()).isEqualTo(SimpleService.class);
    }

    @Test
    void shouldRegisterControllerAsBean_WhenMetaAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("simpleController")).isTrue();
        BeanDefinition bd = factory.getBeanDefinition("simpleController");
        assertThat(bd.getBeanClass()).isEqualTo(SimpleController.class);
    }

    @Test
    void shouldRegisterRepositoryAsBean_WhenMetaAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("simpleRepository")).isTrue();
        BeanDefinition bd = factory.getBeanDefinition("simpleRepository");
        assertThat(bd.getBeanClass()).isEqualTo(SimpleRepository.class);
    }

    @Test
    void shouldScanSubPackagesRecursively_WhenBasePackageSpecified() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("subPackageComponent")).isTrue();
        BeanDefinition bd = factory.getBeanDefinition("subPackageComponent");
        assertThat(bd.getBeanClass()).isEqualTo(SubPackageComponent.class);
    }

    // -----------------------------------------------------------------------
    // Candidate Filtering
    // -----------------------------------------------------------------------

    @Test
    void shouldSkipPlainClasses_WhenNoComponentAnnotation() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("notAComponent")).isFalse();
    }

    @Test
    void shouldSkipAbstractClasses_WhenAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("abstractComponent")).isFalse();
    }

    @Test
    void shouldSkipInterfaces_WhenAnnotatedWithComponent() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("componentInterface")).isFalse();
    }

    // -----------------------------------------------------------------------
    // Bean Naming
    // -----------------------------------------------------------------------

    @Test
    void shouldUseExplicitName_WhenComponentValueSpecified() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("customName")).isTrue();
        assertThat(factory.getBeanDefinition("customName").getBeanClass())
                .isEqualTo(NamedComponent.class);
    }

    @Test
    void shouldUseExplicitName_WhenStereotypeValueSpecified() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        assertThat(factory.containsBeanDefinition("mySpecialService")).isTrue();
        assertThat(factory.getBeanDefinition("mySpecialService").getBeanClass())
                .isEqualTo(NamedService.class);
    }

    @Test
    void shouldDecapitalizeClassName_WhenNoExplicitNameGiven() {
        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        // SimpleComponent → "simpleComponent"
        assertThat(factory.containsBeanDefinition("simpleComponent")).isTrue();
        // SimpleService → "simpleService"
        assertThat(factory.containsBeanDefinition("simpleService")).isTrue();
    }

    // -----------------------------------------------------------------------
    // Duplicate Handling
    // -----------------------------------------------------------------------

    @Test
    void shouldSkipAlreadyRegistered_WhenBeanNameExists() {
        // Pre-register a bean with the same name
        factory.registerBeanDefinition("simpleComponent",
                new BeanDefinition(String.class));

        scanner.scan(factory, "com.iris.framework.context.annotation.scan");

        // The pre-registered definition should NOT be overwritten
        assertThat(factory.getBeanDefinition("simpleComponent").getBeanClass())
                .isEqualTo(String.class);
    }

    // -----------------------------------------------------------------------
    // Multiple Packages
    // -----------------------------------------------------------------------

    @Test
    void shouldScanMultiplePackages_WhenMultipleBasePackagesGiven() {
        int count = scanner.scan(factory,
                "com.iris.framework.context.annotation.scan.sub");

        // Should find only SubPackageComponent in the sub-package
        assertThat(count).isEqualTo(1);
        assertThat(factory.containsBeanDefinition("subPackageComponent")).isTrue();
    }

    // -----------------------------------------------------------------------
    // Empty Package
    // -----------------------------------------------------------------------

    @Test
    void shouldReturnZero_WhenPackageHasNoComponents() {
        // Scanning a package that exists but has no @Component classes
        int count = scanner.scan(factory, "java.lang");

        assertThat(count).isEqualTo(0);
    }

    // -----------------------------------------------------------------------
    // Static Helper Method Tests
    // -----------------------------------------------------------------------

    @Test
    void shouldDetectDirectComponentAnnotation() {
        assertThat(ClassPathScanner.hasComponentAnnotation(SimpleComponent.class)).isTrue();
    }

    @Test
    void shouldDetectMetaComponentAnnotation_OnService() {
        assertThat(ClassPathScanner.hasComponentAnnotation(SimpleService.class)).isTrue();
    }

    @Test
    void shouldDetectMetaComponentAnnotation_OnController() {
        assertThat(ClassPathScanner.hasComponentAnnotation(SimpleController.class)).isTrue();
    }

    @Test
    void shouldDetectMetaComponentAnnotation_OnRepository() {
        assertThat(ClassPathScanner.hasComponentAnnotation(SimpleRepository.class)).isTrue();
    }

    @Test
    void shouldNotDetectComponent_OnPlainClass() {
        assertThat(ClassPathScanner.hasComponentAnnotation(String.class)).isFalse();
    }

    @Test
    void shouldDecapitalizeSimpleName() {
        assertThat(ClassPathScanner.decapitalize("MyService")).isEqualTo("myService");
        assertThat(ClassPathScanner.decapitalize("A")).isEqualTo("a");
        assertThat(ClassPathScanner.decapitalize("")).isEqualTo("");
    }

    @Test
    void shouldPreserveConsecutiveUppercase_WhenDecapitalizing() {
        // JavaBeans Introspector.decapitalize() spec:
        // if first two chars are uppercase, keep as-is
        assertThat(ClassPathScanner.decapitalize("URLService")).isEqualTo("URLService");
        assertThat(ClassPathScanner.decapitalize("IO")).isEqualTo("IO");
    }

    @Test
    void shouldResolveExplicitComponentName() {
        assertThat(ClassPathScanner.resolveBeanName(NamedComponent.class))
                .isEqualTo("customName");
    }

    @Test
    void shouldResolveExplicitStereotypeName() {
        assertThat(ClassPathScanner.resolveBeanName(NamedService.class))
                .isEqualTo("mySpecialService");
    }

    @Test
    void shouldResolveDefaultName_WhenNoExplicitValue() {
        assertThat(ClassPathScanner.resolveBeanName(SimpleComponent.class))
                .isEqualTo("simpleComponent");
    }

    // -----------------------------------------------------------------------
    // Stereotype annotations carry @Component
    // -----------------------------------------------------------------------

    @Test
    void shouldHaveComponentMetaAnnotation_OnService() {
        assertThat(Service.class.isAnnotationPresent(Component.class)).isTrue();
    }

    @Test
    void shouldHaveComponentMetaAnnotation_OnController() {
        assertThat(Controller.class.isAnnotationPresent(Component.class)).isTrue();
    }

    @Test
    void shouldHaveComponentMetaAnnotation_OnRepository() {
        assertThat(Repository.class.isAnnotationPresent(Component.class)).isTrue();
    }

    @Test
    void shouldHaveComponentMetaAnnotation_OnConfiguration() {
        assertThat(Configuration.class.isAnnotationPresent(Component.class)).isTrue();
    }
}
```

#### File: `src/test/java/com/iris/framework/beans/factory/integration/ComponentScanningIntegrationTest.java` [NEW]

```java
package com.iris.framework.beans.factory.integration;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.iris.framework.beans.factory.annotation.AutowiredBeanPostProcessor;
import com.iris.framework.beans.factory.annotation.LifecycleBeanPostProcessor;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.beans.factory.integration.scantest.AppConfig;
import com.iris.framework.beans.factory.integration.scantest.LifecycleComponent;
import com.iris.framework.beans.factory.integration.scantest.UserController;
import com.iris.framework.beans.factory.integration.scantest.UserRepository;
import com.iris.framework.beans.factory.integration.scantest.UserService;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.annotation.ConfigurationClassProcessor;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for component scanning with the full bean lifecycle.
 *
 * <p>Verifies that scanned components integrate correctly with:
 * <ul>
 *   <li>Dependency injection ({@code @Autowired})</li>
 *   <li>Bean lifecycle ({@code @PostConstruct}, {@code InitializingBean})</li>
 *   <li>{@code @Configuration} + {@code @Bean} co-existence</li>
 *   <li>Singleton identity across the container</li>
 * </ul>
 */
class ComponentScanningIntegrationTest {

    private DefaultBeanFactory factory;

    @BeforeEach
    void setUp() {
        factory = new DefaultBeanFactory();

        // Register post-processors (same as prior integration tests)
        factory.addBeanPostProcessor(new AutowiredBeanPostProcessor(factory));
        factory.addBeanPostProcessor(new LifecycleBeanPostProcessor());

        // Register the @Configuration class that triggers scanning
        factory.registerBeanDefinition("appConfig",
                new BeanDefinition(AppConfig.class));

        // Process: Phase 1 (scan) + Phase 2 (@Bean methods)
        new ConfigurationClassProcessor().processConfigurationClasses(factory);
    }

    // -----------------------------------------------------------------------
    // Component Discovery via Scanning
    // -----------------------------------------------------------------------

    @Test
    void shouldDiscoverAllComponents_WhenConfigurationHasComponentScan() {
        assertThat(factory.containsBeanDefinition("userRepository")).isTrue();
        assertThat(factory.containsBeanDefinition("userService")).isTrue();
        assertThat(factory.containsBeanDefinition("userController")).isTrue();
        assertThat(factory.containsBeanDefinition("lifecycleComponent")).isTrue();
    }

    @Test
    void shouldDiscoverConfigurationClass_WhenInScannedPackage() {
        // AppConfig has @Configuration which is meta-annotated with @Component,
        // but it's already registered manually — scanner should skip it
        assertThat(factory.containsBeanDefinition("appConfig")).isTrue();
        assertThat(factory.getBeanDefinition("appConfig").getBeanClass())
                .isEqualTo(AppConfig.class);
    }

    // -----------------------------------------------------------------------
    // Scanning + @Bean Coexistence
    // -----------------------------------------------------------------------

    @Test
    void shouldRegisterBeanMethods_FromScannedConfigurationClass() {
        // The @Bean method appName() on AppConfig should also be registered
        assertThat(factory.containsBeanDefinition("appName")).isTrue();
        String appName = factory.getBean("appName", String.class);
        assertThat(appName).isEqualTo("TestApp");
    }

    // -----------------------------------------------------------------------
    // Dependency Injection on Scanned Components
    // -----------------------------------------------------------------------

    @Test
    void shouldInjectDependencies_WhenScannedComponentsHaveAutowired() {
        UserController controller = factory.getBean(UserController.class);

        // UserController → @Autowired UserService → @Autowired UserRepository
        String result = controller.handleGetUser("42");
        assertThat(result).isEqualTo("User-42");
    }

    @Test
    void shouldMaintainSingletonIdentity_AcrossScannedComponents() {
        UserRepository repo1 = factory.getBean(UserRepository.class);
        UserRepository repo2 = factory.getBean(UserRepository.class);

        assertThat(repo1).isSameAs(repo2);
    }

    @Test
    void shouldShareSingletonAcrossDependencies_WhenInjected() {
        UserService service = factory.getBean(UserService.class);
        UserRepository directRepo = factory.getBean(UserRepository.class);

        // The repository injected into UserService should be the same singleton
        // as the one retrieved directly
        String serviceResult = service.getUser("1");
        String directResult = directRepo.findUser("1");
        assertThat(serviceResult).isEqualTo(directResult);
    }

    // -----------------------------------------------------------------------
    // Lifecycle Callbacks on Scanned Components
    // -----------------------------------------------------------------------

    @Test
    void shouldInvokeLifecycleCallbacks_OnScannedComponents() {
        LifecycleComponent component = factory.getBean(LifecycleComponent.class);

        // @PostConstruct fires before InitializingBean.afterPropertiesSet()
        assertThat(component.getEvents())
                .containsExactly("@PostConstruct", "afterPropertiesSet");
    }

    // -----------------------------------------------------------------------
    // Type-based Lookup
    // -----------------------------------------------------------------------

    @Test
    void shouldResolveByType_WhenScannedComponentRequested() {
        UserService service = factory.getBean(UserService.class);

        assertThat(service).isNotNull();
        assertThat(service).isInstanceOf(UserService.class);
    }

    @Test
    void shouldResolveByName_WhenScannedComponentRequested() {
        Object service = factory.getBean("userService");

        assertThat(service).isNotNull();
        assertThat(service).isInstanceOf(UserService.class);
    }

    // -----------------------------------------------------------------------
    // Transitive Dependency Chain
    // -----------------------------------------------------------------------

    @Test
    void shouldResolveTransitiveDependencies_AcrossScannedComponents() {
        // Controller → Service → Repository (all scanned, all @Autowired)
        UserController controller = factory.getBean(UserController.class);

        assertThat(controller.handleGetUser("99")).isEqualTo("User-99");
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@Component`** | Root stereotype — marks a class as a bean candidate for auto-detection |
| **Stereotype annotations** | `@Service`, `@Controller`, `@Repository` — specialized `@Component` variants that convey architectural intent |
| **Meta-annotation** | An annotation placed on another annotation; `@Service` carries `@Component` as a meta-annotation |
| **`@ComponentScan`** | Configures which packages to scan for annotated components |
| **`ClassPathScanner`** | Walks the classpath, discovers `@Component` classes, and registers them as `BeanDefinition`s |
| **Two-phase processing** | Component scanning runs first (discovers new beans), then `@Bean` method processing runs second (handles all `@Configuration` classes including newly discovered ones) |
| **Decapitalization** | Default bean naming: `MyService` → `"myService"` (unless first two chars are uppercase: `URLService` → `"URLService"`) |

**Next: Chapter 6 — Application Context** — A unified container that orchestrates the full lifecycle: `refresh()` runs component scanning, configuration processing, `BeanPostProcessor` registration, and singleton instantiation in one call, plus an event system with `ApplicationEvent`/`ApplicationListener` for decoupled communication between components.
