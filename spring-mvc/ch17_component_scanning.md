# Chapter 17: Component Scanning

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Controllers and advice beans must be manually instantiated and registered in the BeanContainer before the server starts | Adding a new controller requires changing the bootstrap code — the framework can't discover components on its own | Build a `ClasspathScanner` that automatically discovers `@Component`-annotated classes on the classpath, instantiates them, and registers them in the BeanContainer |

---

## 17.1 The Integration Point

The integration point for component scanning is the **application bootstrap** — the code that sets up the `BeanContainer` before the `DispatcherServlet` starts.

**Before (manual registration):**

```java
SimpleBeanContainer container = new SimpleBeanContainer();
container.registerBean(new UserController());
container.registerBean(new OrderController());
container.registerBean(new GlobalExceptionHandler());
// ... every new controller requires a line here

SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
```

**After (component scanning):**

```java
SimpleBeanContainer container = new SimpleBeanContainer();
ClasspathScanner scanner = new ClasspathScanner(container);
scanner.scan("com.example");  // discovers everything automatically

SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
```

Two key decisions here:

1. **The scanner operates on the existing `BeanContainer`** rather than replacing it — this lets manual and scanned registration coexist (the real framework does the same via `checkCandidate()` conflict detection).
2. **Discovery is based on `@Component`** as the root marker, with `@Controller` and `@ControllerAdvice` carrying `@Component` as a meta-annotation — matching the real framework's stereotype annotation hierarchy.

This connects **classpath resources** (`.class` files) to the **BeanContainer** (ch01). To make it work, we need to build:
- `@Component` annotation — the base marker that makes a class discoverable
- `ClasspathScanner` — converts package names to filesystem paths, walks directories, loads classes, checks annotations, instantiates, and registers
- Update `@Controller` and `@ControllerAdvice` to carry `@Component` as a meta-annotation
- Update `MergedAnnotationUtils.hasAnnotation()` to support recursive meta-annotation walking (needed for `@RestController → @Controller → @Component`)

## 17.2 The `@Component` Annotation

The root of the stereotype hierarchy. Every class eligible for scanning carries `@Component` — either directly or through a meta-annotation chain.

**New file:** `src/main/java/com/simplespringmvc/annotation/Component.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {
}
```

## 17.3 Meta-Annotating `@Controller` and `@ControllerAdvice`

In the real Spring Framework, `@Controller` is meta-annotated with `@Component` (at `Controller.java:45`). We add the same chain to our simplified annotations.

**Modifying:** `src/main/java/com/simplespringmvc/annotation/Controller.java`
**Change:** Add `@Component` meta-annotation

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component           // ← NEW: makes @Controller classes eligible for scanning
public @interface Controller {
}
```

**Modifying:** `src/main/java/com/simplespringmvc/annotation/ControllerAdvice.java`
**Change:** Add `@Component` meta-annotation

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component           // ← NEW: makes @ControllerAdvice classes eligible for scanning
public @interface ControllerAdvice {
}
```

`@RestController` doesn't need changes — it already carries `@Controller`, and through the new chain `@RestController → @Controller → @Component`, it's automatically eligible for scanning.

## 17.4 Upgrading `MergedAnnotationUtils` for Recursive Walking

Here's the problem: `@RestController → @Controller → @Component` is **two levels deep**. The previous `hasAnnotation()` only checked one level. We need recursive meta-annotation walking.

**Modifying:** `src/main/java/com/simplespringmvc/annotation/MergedAnnotationUtils.java`
**Change:** Replace the one-level `hasAnnotation()` loop with a recursive `hasMetaAnnotation()` helper

```java
public static boolean hasAnnotation(AnnotatedElement element,
                                    Class<? extends Annotation> annotationType) {
    // Direct presence check
    if (element.isAnnotationPresent(annotationType)) {
        return true;
    }

    // Recursive meta-annotation check
    Set<Class<? extends Annotation>> visited = new HashSet<>();
    for (Annotation ann : element.getAnnotations()) {
        if (hasMetaAnnotation(ann.annotationType(), annotationType, visited)) {
            return true;
        }
    }
    return false;
}

private static boolean hasMetaAnnotation(Class<? extends Annotation> currentAnnotation,
                                          Class<? extends Annotation> targetAnnotation,
                                          Set<Class<? extends Annotation>> visited) {
    if (currentAnnotation.isAnnotationPresent(targetAnnotation)) {
        return true;
    }
    if (!visited.add(currentAnnotation)) {
        return false;  // cycle guard
    }
    for (Annotation metaAnn : currentAnnotation.getAnnotations()) {
        Class<? extends Annotation> metaType = metaAnn.annotationType();
        // Skip java.lang.annotation.* — can never carry our annotations
        if (metaType.getName().startsWith("java.lang.annotation")) {
            continue;
        }
        if (hasMetaAnnotation(metaType, targetAnnotation, visited)) {
            return true;
        }
    }
    return false;
}
```

The `visited` set prevents infinite loops from circular meta-annotations (defensive programming — unlikely in practice but correct). Skipping `java.lang.annotation.*` avoids walking into JDK annotations (`@Target`, `@Retention`, etc.) that can never carry our custom annotations.

## 17.5 The `ClasspathScanner`

The core class. It converts package names to filesystem paths, walks directories for `.class` files, loads each class, checks for `@Component`, and registers matches in the container.

**New file:** `src/main/java/com/simplespringmvc/scan/ClasspathScanner.java`

```java
package com.simplespringmvc.scan;

import com.simplespringmvc.annotation.Component;
import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.container.BeanContainer;

import java.beans.Introspector;
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Modifier;
import java.net.JarURLConnection;
import java.net.URL;
import java.util.Enumeration;
import java.util.LinkedHashSet;
import java.util.Set;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

public class ClasspathScanner {

    private final BeanContainer beanContainer;

    public ClasspathScanner(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    public int scan(String... basePackages) {
        int count = 0;
        for (String basePackage : basePackages) {
            count += doScan(basePackage);
        }
        return count;
    }

    private int doScan(String basePackage) {
        Set<Class<?>> candidates = findCandidateComponents(basePackage);
        int count = 0;
        for (Class<?> clazz : candidates) {
            String beanName = deriveBeanName(clazz);
            if (!beanContainer.containsBean(beanName)) {
                Object instance = instantiate(clazz);
                beanContainer.registerBean(beanName, instance);
                count++;
            }
        }
        return count;
    }

    private Set<Class<?>> findCandidateComponents(String basePackage) {
        Set<Class<?>> candidates = new LinkedHashSet<>();
        String packagePath = basePackage.replace('.', '/');
        try {
            ClassLoader classLoader = getClassLoader();
            Enumeration<URL> resources = classLoader.getResources(packagePath);
            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                if ("file".equals(resource.getProtocol())) {
                    scanDirectory(new File(resource.toURI()), basePackage, candidates);
                } else if ("jar".equals(resource.getProtocol())) {
                    scanJar(resource, packagePath, candidates);
                }
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to scan package: " + basePackage, e);
        }
        return candidates;
    }

    private void scanDirectory(File directory, String packageName, Set<Class<?>> candidates) {
        File[] files = directory.listFiles();
        if (files == null) return;
        for (File file : files) {
            if (file.isDirectory()) {
                scanDirectory(file, packageName + "." + file.getName(), candidates);
            } else if (file.getName().endsWith(".class")) {
                String className = packageName + "."
                        + file.getName().substring(0, file.getName().length() - ".class".length());
                loadAndCheck(className, candidates);
            }
        }
    }

    private void scanJar(URL jarUrl, String packagePath, Set<Class<?>> candidates)
            throws IOException {
        JarURLConnection connection = (JarURLConnection) jarUrl.openConnection();
        try (JarFile jarFile = connection.getJarFile()) {
            Enumeration<JarEntry> entries = jarFile.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                String entryName = entry.getName();
                if (entryName.startsWith(packagePath + "/") && entryName.endsWith(".class")) {
                    String className = entryName
                            .substring(0, entryName.length() - ".class".length())
                            .replace('/', '.');
                    loadAndCheck(className, candidates);
                }
            }
        }
    }

    private void loadAndCheck(String className, Set<Class<?>> candidates) {
        try {
            Class<?> clazz = Class.forName(className, false, getClassLoader());
            if (isCandidateComponent(clazz)) {
                candidates.add(clazz);
            }
        } catch (ClassNotFoundException | NoClassDefFoundError e) {
            // Skip — class can't be loaded
        }
    }

    private boolean isCandidateComponent(Class<?> clazz) {
        if (!MergedAnnotationUtils.hasAnnotation(clazz, Component.class)) {
            return false;
        }
        if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
            return false;
        }
        if (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers())) {
            return false;
        }
        return true;
    }

    private Object instantiate(Class<?> clazz) {
        try {
            var constructor = clazz.getDeclaredConstructor();
            constructor.setAccessible(true);
            return constructor.newInstance();
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(
                    "Component class " + clazz.getName() + " must have a no-arg constructor.", e);
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException("Failed to instantiate component: " + clazz.getName(), e);
        }
    }

    private String deriveBeanName(Class<?> clazz) {
        return Introspector.decapitalize(clazz.getSimpleName());
    }

    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return cl != null ? cl : getClass().getClassLoader();
    }
}
```

### How it works — the scanning pipeline:

```
scanner.scan("com.example")
  │
  ├── Convert: "com.example" → "com/example"
  │
  ├── ClassLoader.getResources("com/example")
  │     → returns URLs pointing to directories or JARs
  │
  ├── For each URL:
  │     ├── file: protocol → scanDirectory() — recursive File.listFiles()
  │     └── jar: protocol  → scanJar() — iterate JarFile entries
  │
  ├── For each .class file:
  │     ├── Class.forName(name, false, classLoader)
  │     │     └── false = don't run static initializers
  │     │
  │     └── isCandidateComponent(clazz):
  │           ├── Has @Component? (via recursive MergedAnnotationUtils.hasAnnotation)
  │           ├── Not abstract? Not interface?
  │           └── Independent? (top-level or static nested)
  │
  └── For each candidate:
        ├── deriveBeanName() → Introspector.decapitalize(simpleName)
        ├── Skip if already registered
        └── instantiate() → clazz.getDeclaredConstructor().newInstance()
            → beanContainer.registerBean(name, instance)
```

## 17.6 Try It Yourself

<details>
<summary>Challenge 1: What gets discovered?</summary>

Given these classes in package `com.example`:

```java
@Controller
public class UserController { ... }

@RestController
public class ApiController { ... }

public class HelperService { ... }

@Component
public abstract class BaseHandler { ... }

@Component
public interface Validator { ... }

@Component
static class InnerComponent { ... }  // static inner class
```

Which ones will `ClasspathScanner.scan("com.example")` register?

**Answer:** `UserController`, `ApiController`, and `InnerComponent`.
- `UserController` — `@Controller → @Component` (meta-annotation chain)
- `ApiController` — `@RestController → @Controller → @Component` (two-level chain)
- `InnerComponent` — `@Component` directly, and it's static (independent)
- `HelperService` — EXCLUDED: no `@Component`
- `BaseHandler` — EXCLUDED: abstract class
- `Validator` — EXCLUDED: interface

</details>

<details>
<summary>Challenge 2: Implement the `isCandidateComponent` method</summary>

Given the `@Component` annotation and `MergedAnnotationUtils.hasAnnotation()`, write the `isCandidateComponent(Class<?>)` method that checks if a class should be registered as a bean.

Requirements:
- Must have `@Component` (directly or via meta-annotation)
- Must not be abstract or an interface
- Must be independent (top-level or static nested)

```java
private boolean isCandidateComponent(Class<?> clazz) {
    if (!MergedAnnotationUtils.hasAnnotation(clazz, Component.class)) {
        return false;
    }
    if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
        return false;
    }
    if (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers())) {
        return false;
    }
    return true;
}
```

</details>

## 17.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/scan/ClasspathScannerTest.java`

```java
@Test
void shouldDiscoverControllerAnnotatedClass_WhenScanningPackage() {
    scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(container.containsBean("annotatedController")).isTrue();
    assertThat(container.getBean("annotatedController")).isInstanceOf(AnnotatedController.class);
}

@Test
void shouldDiscoverRestControllerAnnotatedClass_WhenScanningPackage() {
    scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(container.containsBean("annotatedRestController")).isTrue();
}

@Test
void shouldNotDiscoverPlainClass_WhenScanningPackage() {
    scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(container.containsBean("plainClass")).isFalse();
}

@Test
void shouldNotDiscoverAbstractClass_WhenScanningPackage() {
    scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(container.containsBean("abstractComponent")).isFalse();
}

@Test
void shouldDiscoverClassesInSubPackages_WhenScanningParentPackage() {
    scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(container.containsBean("subPackageController")).isTrue();
}

@Test
void shouldNotReRegisterBean_WhenAlreadyRegisteredManually() {
    AnnotatedController manualBean = new AnnotatedController();
    container.registerBean("annotatedController", manualBean);
    scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(container.getBean("annotatedController")).isSameAs(manualBean);
}

@Test
void shouldReturnCountOfNewlyRegisteredBeans_WhenScanning() {
    int count = scanner.scan("com.simplespringmvc.scan.testpackage");
    assertThat(count).isEqualTo(5);  // 5 concrete @Component classes
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/ComponentScanningIntegrationTest.java`

```java
@Test
void shouldHandleRequest_WhenControllerDiscoveredByScanning() throws Exception {
    SimpleBeanContainer container = new SimpleBeanContainer();
    ClasspathScanner scanner = new ClasspathScanner(container);
    scanner.scan("com.simplespringmvc.scan.testpackage");

    SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
    tomcat = new EmbeddedTomcat(0, servlet);
    tomcat.start();

    HttpResponse<String> response = client.send(
            HttpRequest.newBuilder()
                    .uri(URI.create("http://localhost:" + tomcat.getPort() + "/annotated/hello"))
                    .GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("hello from scanned controller");
}

@Test
void shouldRegisterHandlerMappings_WhenControllersDiscoveredByScanning() {
    SimpleBeanContainer container = new SimpleBeanContainer();
    new ClasspathScanner(container).scan("com.simplespringmvc.scan.testpackage");

    SimpleHandlerMapping mapping = new SimpleHandlerMapping();
    mapping.init(container);

    assertThat(mapping.lookupHandler("/annotated/hello", "GET")).isNotNull();
    assertThat(mapping.lookupHandler("/api/data", "GET")).isNotNull();
    assertThat(mapping.lookupHandler("/sub/hello", "GET")).isNotNull();
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior 502+ tests)

---

## 17.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Stereotype annotations create a discoverable type hierarchy.** By making `@Controller` carry `@Component`, any class annotated with `@Controller` (or `@RestController`, which carries `@Controller`) becomes eligible for scanning without any additional configuration. The scanner only needs to know about `@Component` — it doesn't need to enumerate every possible stereotype. This is the Open/Closed Principle applied to annotation discovery: open for extension (add new stereotypes by meta-annotating with `@Component`), closed for modification (the scanner never changes).
>
> In the real framework, this same pattern extends to `@Service`, `@Repository`, and any custom annotation the user creates — as long as it's meta-annotated with `@Component` (or another stereotype), it's scannable.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **ASM vs Class.forName() — why the real framework avoids loading classes.** We use `Class.forName(name, false, classLoader)` which loads the class into the JVM but skips static initialization. The real framework uses ASM's `ClassReader` to parse bytecode directly, extracting annotation metadata without ever loading the class. This matters at scale: scanning thousands of classes with `Class.forName()` allocates JVM metadata for each, while ASM reads binary data and discards it. For our educational purposes, `Class.forName()` is clearer and sufficient — but the real framework's choice of ASM is a deliberate performance optimization.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **ClassLoader.getResources() is the JDK's built-in package discovery.** We leverage `ClassLoader.getResources(packagePath)` which returns URLs for every classpath entry containing that package — filesystem directories for development builds, JAR URLs for packaged dependencies. The real framework wraps this in `PathMatchingResourcePatternResolver` which adds Ant-style pattern matching (`**/*.class`), module path scanning, and caching. Our simplified approach handles the two most common cases (directory + JAR) directly.
> -----------------------------------------------------------

## 17.9 What We Enhanced

| Aspect | Before (ch16) | Current (ch17) | Real Framework |
|--------|---------------|----------------|----------------|
| **Bean discovery** | Manual `container.registerBean(new Controller())` for every bean | Automatic: `scanner.scan("com.example")` finds all `@Component` classes | `ClassPathBeanDefinitionScanner.doScan()` at `ClassPathBeanDefinitionScanner.java:272` |
| **Meta-annotation walking** | One-level deep (`@RestController → @Controller`) | Recursive with cycle detection (`@RestController → @Controller → @Component`) | `MergedAnnotations` with `TypeMappedAnnotations` — full depth with caching |
| **@Controller eligibility** | Only a handler marker for `SimpleHandlerMapping` | Also a stereotype: carries `@Component`, eligible for scanning | `@Controller` → `@Component` at `Controller.java:45` |
| **@ControllerAdvice eligibility** | Only an exception handler marker | Also a stereotype: carries `@Component`, eligible for scanning | `@ControllerAdvice` → `@Component` at `ControllerAdvice.java:34` |

## 17.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ClasspathScanner` | `ClassPathBeanDefinitionScanner` | `ClassPathBeanDefinitionScanner.java:64` | Real version separates discovery (parent class) from registration, supports scopes, `@Lazy`, `@Primary`, conflict detection |
| `ClasspathScanner.findCandidateComponents()` | `ClassPathScanningCandidateComponentProvider.scanCandidateComponents()` | `ClassPathScanningCandidateComponentProvider.java:446` | Real version uses ASM `MetadataReader` instead of `Class.forName()`, supports include/exclude `TypeFilter` chain, component index optimization |
| `ClasspathScanner.isCandidateComponent()` | `isCandidateComponent(MetadataReader)` + `isCandidateComponent(AnnotatedBeanDefinition)` | `ClassPathScanningCandidateComponentProvider.java:533,571` | Real version is two separate methods — filter-based check + structural check — with `@Conditional` evaluation |
| `ClasspathScanner.deriveBeanName()` | `AnnotationBeanNameGenerator.buildDefaultBeanName()` | `AnnotationBeanNameGenerator.java:150` | Real version checks for explicit name in `@Component(value="...")` before falling back to `Introspector.decapitalize()` |
| `ClasspathScanner.scanDirectory()` | `PathMatchingResourcePatternResolver.doFindPathMatchingFileResources()` | `PathMatchingResourcePatternResolver.java:1011` | Real version uses `Files.walk()` with `FOLLOW_LINKS`, supports Ant-style patterns, has root directory caching |
| `MergedAnnotationUtils.hasMetaAnnotation()` | `TypeMappedAnnotations` recursive search | `TypeMappedAnnotations.java` | Real version builds a complete annotation mapping tree, caches results, supports `@AliasFor` attribute merging |
| `@Component` | `@Component` | `Component.java:70` | Real version has `@Indexed` for compile-time component index, and `value()` attribute for explicit bean naming |

## 17.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/Component.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as a component eligible for classpath scanning.
 *
 * Maps to: {@code org.springframework.stereotype.Component}
 *
 * In the real framework, @Component is the base stereotype annotation. All other
 * stereotype annotations (@Controller, @Service, @Repository) are meta-annotated
 * with @Component, making them eligible for classpath scanning transitively.
 *
 * <h3>The stereotype hierarchy:</h3>
 * <pre>
 *   @Component                    ← base stereotype
 *     ├── @Controller             ← web handler (ch03)
 *     │     └── @RestController   ← @Controller + @ResponseBody (ch10)
 *     └── @ControllerAdvice       ← global exception handler (ch12)
 * </pre>
 *
 * <h3>How component scanning uses this:</h3>
 * <pre>
 *   ClasspathScanner.scan("com.example")
 *     → finds all .class files in the package
 *     → loads each class
 *     → checks: MergedAnnotationUtils.hasAnnotation(clazz, Component.class)
 *     → if true: instantiate + register in BeanContainer
 * </pre>
 *
 * In the real framework, @Component also carries @Indexed for compile-time
 * component index optimization (META-INF/spring.components). We skip that
 * optimization — it's a performance concern, not a design concern.
 *
 * Simplifications:
 * <ul>
 *   <li>No value() attribute for custom bean naming</li>
 *   <li>No @Indexed meta-annotation</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {
}
```

#### File: `src/main/java/com/simplespringmvc/scan/ClasspathScanner.java` [NEW]

```java
package com.simplespringmvc.scan;

import com.simplespringmvc.annotation.Component;
import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.container.BeanContainer;

import java.beans.Introspector;
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Modifier;
import java.net.JarURLConnection;
import java.net.URL;
import java.util.Enumeration;
import java.util.LinkedHashSet;
import java.util.Set;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

/**
 * Scans the classpath for classes annotated with {@link Component} (directly or via
 * meta-annotation) and registers them as singleton beans in the {@link BeanContainer}.
 *
 * Maps to: {@code org.springframework.context.annotation.ClassPathBeanDefinitionScanner}
 * + {@code ClassPathScanningCandidateComponentProvider}
 *
 * <h3>End-to-end scanning flow (real framework):</h3>
 * <pre>
 *   ClassPathBeanDefinitionScanner.doScan("com.example")
 *     → findCandidateComponents("com.example")                     [inherited from parent]
 *       → scanCandidateComponents("com.example")
 *         → builds pattern: "classpath*:com/example/&#42;&#42;/&#42;.class"
 *         → PathMatchingResourcePatternResolver.getResources(pattern)
 *           → for each Resource (.class file):
 *             → MetadataReader (ASM-based, NO class loading)
 *             → isCandidateComponent(MetadataReader)               [filter chain]
 *             → isCandidateComponent(AnnotatedBeanDefinition)      [structural check]
 *     → for each candidate:
 *       → generate bean name (AnnotationBeanNameGenerator)
 *       → process @Lazy, @Primary, @DependsOn
 *       → check for conflicts
 *       → register BeanDefinition in registry
 * </pre>
 *
 * <h3>Our simplified flow:</h3>
 * <pre>
 *   ClasspathScanner.scan("com.example")
 *     → findCandidateComponents("com.example")
 *       → ClassLoader.getResources("com/example")
 *       → for each URL:
 *         → file: protocol → walk directory tree
 *         → jar: protocol  → iterate JarFile entries
 *       → for each .class file:
 *         → Class.forName() (loads the class — simpler than ASM)
 *         → isCandidateComponent(Class)
 *     → for each candidate:
 *       → generate bean name (Introspector.decapitalize)
 *       → instantiate via no-arg constructor
 *       → register in BeanContainer
 * </pre>
 *
 * <h3>Key simplification: Class.forName() vs ASM MetadataReader</h3>
 * The real framework uses ASM to read class bytecode WITHOUT loading the class.
 * This avoids triggering static initializers and is much faster when scanning
 * thousands of classes. We use Class.forName() with {@code initialize=false}
 * to avoid static initializers while keeping the code simple.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>Uses Class.forName() instead of ASM MetadataReader</li>
 *   <li>No include/exclude TypeFilter chain</li>
 *   <li>No @Conditional evaluation</li>
 *   <li>No component index (META-INF/spring.components) optimization</li>
 *   <li>No scope/proxy support</li>
 *   <li>No @Lazy/@Primary/@DependsOn processing</li>
 *   <li>No BeanDefinition — instantiates directly</li>
 *   <li>No conflict detection — skips already-registered beans</li>
 * </ul>
 */
public class ClasspathScanner {

    private final BeanContainer beanContainer;

    public ClasspathScanner(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Scan the given base packages for component classes and register them
     * as beans in the container.
     *
     * Maps to: {@code ClassPathBeanDefinitionScanner.scan(String...)} (line 251)
     *
     * @param basePackages the packages to scan (e.g., "com.example.controllers")
     * @return the number of newly registered beans
     */
    public int scan(String... basePackages) {
        int count = 0;
        for (String basePackage : basePackages) {
            count += doScan(basePackage);
        }
        return count;
    }

    /**
     * Perform the actual scan for a single base package.
     *
     * Maps to: {@code ClassPathBeanDefinitionScanner.doScan(String...)} (line 272)
     *
     * The real version returns {@code Set<BeanDefinitionHolder>}. We skip the
     * BeanDefinition layer and register beans directly.
     */
    private int doScan(String basePackage) {
        Set<Class<?>> candidates = findCandidateComponents(basePackage);
        int count = 0;

        for (Class<?> clazz : candidates) {
            String beanName = deriveBeanName(clazz);

            // Skip if already registered — prevents double-registration when
            // a bean was registered manually before scanning.
            // Real framework does conflict checking in checkCandidate() (line 335).
            if (!beanContainer.containsBean(beanName)) {
                Object instance = instantiate(clazz);
                beanContainer.registerBean(beanName, instance);
                count++;
            }
        }

        return count;
    }

    /**
     * Find all classes in the given package that are annotated with @Component
     * (directly or via meta-annotation).
     *
     * Maps to: {@code ClassPathScanningCandidateComponentProvider.findCandidateComponents()}
     * (line 312) → {@code scanCandidateComponents()} (line 446)
     *
     * The real version builds a resource pattern like "classpath*:com/example/&#42;&#42;/&#42;.class"
     * and uses PathMatchingResourcePatternResolver to find resources. We use
     * ClassLoader.getResources() which gives us the root directories/JARs for
     * the package, then walk them ourselves.
     */
    private Set<Class<?>> findCandidateComponents(String basePackage) {
        Set<Class<?>> candidates = new LinkedHashSet<>();
        String packagePath = basePackage.replace('.', '/');

        try {
            ClassLoader classLoader = getClassLoader();
            Enumeration<URL> resources = classLoader.getResources(packagePath);

            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                String protocol = resource.getProtocol();

                if ("file".equals(protocol)) {
                    // Filesystem: compiled classes in a directory (typical for development)
                    File directory = new File(resource.toURI());
                    scanDirectory(directory, basePackage, candidates);
                } else if ("jar".equals(protocol)) {
                    // JAR file: classes packaged in a jar (typical for dependencies)
                    scanJar(resource, packagePath, candidates);
                }
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to scan package: " + basePackage, e);
        }

        return candidates;
    }

    /**
     * Recursively scan a filesystem directory for .class files.
     *
     * Maps to: {@code PathMatchingResourcePatternResolver.doFindPathMatchingFileResources()}
     * (line 1011) which uses {@code Files.walk()} with FOLLOW_LINKS.
     *
     * We use simple recursion through File.listFiles() — more readable for the
     * educational purpose, and equivalent for small codebases.
     *
     * @param directory   the directory to scan
     * @param packageName the Java package corresponding to this directory
     * @param candidates  accumulator for discovered component classes
     */
    private void scanDirectory(File directory, String packageName, Set<Class<?>> candidates) {
        File[] files = directory.listFiles();
        if (files == null) {
            return;
        }

        for (File file : files) {
            if (file.isDirectory()) {
                // Recurse into subdirectories — each subdirectory is a sub-package
                scanDirectory(file, packageName + "." + file.getName(), candidates);
            } else if (file.getName().endsWith(".class")) {
                // Convert filename to fully-qualified class name
                // e.g., "UserController.class" → "com.example.UserController"
                String className = packageName + "."
                        + file.getName().substring(0, file.getName().length() - ".class".length());
                loadAndCheck(className, candidates);
            }
        }
    }

    /**
     * Scan a JAR file for .class files under the given package path.
     *
     * Maps to: {@code PathMatchingResourcePatternResolver.doFindPathMatchingJarResources()}
     * (line 864) which opens a JarURLConnection, iterates entries, and matches
     * against the sub-pattern using AntPathMatcher.
     *
     * We simplify by iterating all entries and checking the prefix.
     */
    private void scanJar(URL jarUrl, String packagePath, Set<Class<?>> candidates) throws IOException {
        JarURLConnection connection = (JarURLConnection) jarUrl.openConnection();
        try (JarFile jarFile = connection.getJarFile()) {
            Enumeration<JarEntry> entries = jarFile.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                String entryName = entry.getName();

                // Only process .class files under our target package
                if (entryName.startsWith(packagePath + "/") && entryName.endsWith(".class")) {
                    // Convert JAR entry path to class name
                    // e.g., "com/example/UserController.class" → "com.example.UserController"
                    String className = entryName
                            .substring(0, entryName.length() - ".class".length())
                            .replace('/', '.');
                    loadAndCheck(className, candidates);
                }
            }
        }
    }

    /**
     * Load a class by name and add it to the candidate set if it passes the
     * component check.
     *
     * Uses {@code Class.forName(name, false, classLoader)} — the {@code false}
     * parameter prevents static initializer execution, similar (in spirit) to
     * the real framework's ASM-based MetadataReader which never loads the class.
     */
    private void loadAndCheck(String className, Set<Class<?>> candidates) {
        try {
            Class<?> clazz = Class.forName(className, false, getClassLoader());
            if (isCandidateComponent(clazz)) {
                candidates.add(clazz);
            }
        } catch (ClassNotFoundException | NoClassDefFoundError e) {
            // Skip classes that can't be loaded — may have unsatisfied dependencies.
            // The real framework logs at TRACE level and continues.
        }
    }

    /**
     * Check if a class is eligible for component registration.
     *
     * Maps to two checks in the real framework:
     * <ol>
     *   <li>{@code isCandidateComponent(MetadataReader)} (line 533) — filter-based:
     *       checks exclude filters, then include filters (default: @Component)</li>
     *   <li>{@code isCandidateComponent(AnnotatedBeanDefinition)} (line 571) — structural:
     *       must be independent (top-level or static nested) and concrete</li>
     * </ol>
     *
     * We combine both checks into one method.
     */
    private boolean isCandidateComponent(Class<?> clazz) {
        // Filter check: must have @Component (directly or via meta-annotation)
        // Real framework uses AnnotationTypeFilter(Component.class) as the default include filter
        if (!MergedAnnotationUtils.hasAnnotation(clazz, Component.class)) {
            return false;
        }

        // Structural check 1: must not be an interface or abstract class
        // Real framework: AnnotatedBeanDefinition.getMetadata().isConcrete()
        if (clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers())) {
            return false;
        }

        // Structural check 2: must be independent — top-level or static nested
        // Real framework: AnnotatedBeanDefinition.getMetadata().isIndependent()
        // Inner (non-static) classes need an enclosing instance and can't be
        // independently instantiated.
        if (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers())) {
            return false;
        }

        return true;
    }

    /**
     * Instantiate a component class using its no-arg constructor.
     *
     * Maps to: In the real framework, instantiation happens much later —
     * {@code AbstractAutowireCapableBeanFactory.createBeanInstance()} (line 1235)
     * which supports constructor injection, factory methods, and more.
     *
     * We require a no-arg constructor — no dependency injection support.
     *
     * @throws RuntimeException if instantiation fails
     */
    private Object instantiate(Class<?> clazz) {
        try {
            var constructor = clazz.getDeclaredConstructor();
            constructor.setAccessible(true);
            return constructor.newInstance();
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(
                    "Component class " + clazz.getName() + " must have a no-arg constructor. "
                            + "Dependency injection is not supported in the simplified framework.", e);
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException("Failed to instantiate component: " + clazz.getName(), e);
        }
    }

    /**
     * Derive a bean name from the class: short name with first letter decapitalized.
     *
     * Maps to: {@code AnnotationBeanNameGenerator.buildDefaultBeanName()} (line 150)
     * which calls {@code Introspector.decapitalize(shortClassName)}.
     *
     * Uses JDK's {@code Introspector.decapitalize()} to match the real framework's
     * behavior — notably, if the first TWO characters are uppercase (e.g., "URL"),
     * the name is kept as-is ("URL" not "uRL").
     *
     * Examples:
     * <ul>
     *   <li>UserController → "userController"</li>
     *   <li>SSEEmitter → "SSEEmitter" (first two chars uppercase — keep as-is)</li>
     *   <li>HTMLParser → "HTMLParser" (first two chars uppercase — keep as-is)</li>
     * </ul>
     */
    private String deriveBeanName(Class<?> clazz) {
        String shortName = clazz.getSimpleName();
        return Introspector.decapitalize(shortName);
    }

    private ClassLoader getClassLoader() {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return cl != null ? cl : getClass().getClassLoader();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/Controller.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as a web controller whose methods can handle HTTP requests.
 *
 * Maps to: {@code org.springframework.stereotype.Controller}
 *
 * In real Spring, @Controller is meta-annotated with @Component so it gets
 * picked up by component scanning. Our simplified version follows the same
 * pattern — classes annotated with @Controller (or @RestController, which
 * carries @Controller) are automatically discovered by ClasspathScanner.
 *
 * <h3>The meta-annotation chain:</h3>
 * <pre>
 *   @Controller
 *     └── @Component  → eligible for classpath scanning (ch17)
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No value() attribute for bean naming</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/ControllerAdvice.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as a global exception handler (and future model-attribute /
 * init-binder provider) for all {@link Controller} classes.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.ControllerAdvice}
 *
 * In the real framework, {@code @ControllerAdvice} is meta-annotated with
 * {@code @Component}, so it's discovered by component scanning. It also
 * supports filtering by base packages, assignable types, and target annotations.
 *
 * <h3>How it works:</h3>
 * <pre>
 *   @ControllerAdvice
 *   public class GlobalExceptionHandler {
 *
 *       @ExceptionHandler(IllegalArgumentException.class)
 *       @ResponseBody
 *       public Map&lt;String, String&gt; handleBadRequest(IllegalArgumentException ex) {
 *           return Map.of("error", ex.getMessage());
 *       }
 *   }
 * </pre>
 *
 * The {@code ExceptionHandlerExceptionResolver} discovers these beans and
 * consults them after checking the controller's own {@code @ExceptionHandler}
 * methods — controller-local handlers always take priority.
 *
 * <h3>ch17 Enhancement:</h3>
 * Now meta-annotated with {@code @Component}, making @ControllerAdvice classes
 * eligible for automatic discovery via {@code ClasspathScanner.scan()}.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No filtering by basePackages, assignableTypes, or annotations</li>
 *   <li>No ordering via {@code @Order} / {@code Ordered}</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/RestController.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Convenience annotation that combines {@link Controller} and {@link ResponseBody}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.RestController}
 *
 * A class annotated with {@code @RestController} is:
 * <ul>
 *   <li>Detected as a handler by {@link com.simplespringmvc.mapping.SimpleHandlerMapping}
 *       (because it carries @Controller as a meta-annotation)</li>
 *   <li>Has all return values serialized to the response body via
 *       {@link com.simplespringmvc.adapter.ResponseBodyReturnValueHandler}
 *       (because it carries @ResponseBody as a meta-annotation)</li>
 *   <li>Eligible for classpath scanning via {@link com.simplespringmvc.scan.ClasspathScanner}
 *       (because @Controller carries @Component as a meta-annotation)</li>
 * </ul>
 *
 * <h3>The meta-annotation chain:</h3>
 * <pre>
 *   @RestController
 *     ├── @Controller    → detected by SimpleHandlerMapping.isHandler()
 *     │     └── @Component  → eligible for classpath scanning (ch17)
 *     └── @ResponseBody  → detected by ResponseBodyReturnValueHandler.supportsReturnType()
 * </pre>
 *
 * In the real framework, @RestController also carries @Indexed (via @Component)
 * for AOT support. The value() attribute is aliased to
 * Controller.value() → Component.value() for bean naming.
 *
 * Simplifications:
 * <ul>
 *   <li>No value() attribute for bean naming</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/MergedAnnotationUtils.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;
import java.util.HashSet;
import java.util.Set;

/**
 * Utility for walking meta-annotation hierarchies to detect annotations that
 * are present either directly or as meta-annotations on composed annotations.
 *
 * Maps to: {@code org.springframework.core.annotation.AnnotatedElementUtils}
 * which delegates to the {@code MergedAnnotations} API internally.
 *
 * <h3>What is a meta-annotation?</h3>
 * A meta-annotation is an annotation placed on another annotation's definition.
 * For example, {@code @GetMapping} is annotated with {@code @RequestMapping(method = "GET")},
 * making {@code @RequestMapping} a meta-annotation of {@code @GetMapping}.
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. AnnotatedElementUtils.hasAnnotation() — uses SearchStrategy.TYPE_HIERARCHY
 *      to exhaustively search superclasses, interfaces, and all meta-annotations
 *   2. AnnotatedElementUtils.findMergedAnnotation() — searches + synthesizes an
 *      annotation proxy with merged attribute values via @AliasFor
 *   3. MergedAnnotations.from(element, strategy) — the low-level API that builds
 *      a tree of TypeMappedAnnotations with merged attribute views
 * </pre>
 *
 * <h3>ch17 Enhancement:</h3>
 * {@code hasAnnotation()} now performs recursive meta-annotation walking to support
 * chains like {@code @RestController → @Controller → @Component}. Previous chapters
 * only needed one-level search (e.g., {@code @GetMapping → @RequestMapping}).
 *
 * <h3>Simplifications:</h3>
 * <ul>
 *   <li>No @AliasFor attribute merging — we extract attributes via reflection</li>
 *   <li>No SearchStrategy (no superclass/interface traversal)</li>
 *   <li>No annotation synthesis — we return the raw annotation instances</li>
 *   <li>No caching of resolved annotations</li>
 * </ul>
 */
public class MergedAnnotationUtils {

    /**
     * Check if an annotation type is present on the element — either directly
     * or as a meta-annotation at any depth on one of the element's annotations.
     *
     * Maps to: {@code AnnotatedElementUtils.hasAnnotation()} (line 537)
     * Real version uses find semantics (TYPE_HIERARCHY search), walking
     * superclasses, interfaces, and the full meta-annotation depth.
     *
     * <h3>ch17 Enhancement:</h3>
     * Now performs recursive meta-annotation walking to support annotation chains
     * like {@code @RestController → @Controller → @Component}. Previous chapters
     * only searched one level deep, which was sufficient for
     * {@code @GetMapping → @RequestMapping}. Component scanning introduced
     * deeper chains when @Component became a meta-annotation of @Controller.
     *
     * Examples:
     * <ul>
     *   <li>{@code hasAnnotation(MyController.class, Controller.class)} → true
     *       if class has @Controller directly</li>
     *   <li>{@code hasAnnotation(MyRestController.class, Controller.class)} → true
     *       if class has @RestController (which is meta-annotated with @Controller)</li>
     *   <li>{@code hasAnnotation(MyRestController.class, Component.class)} → true
     *       via @RestController → @Controller → @Component (2 levels deep)</li>
     * </ul>
     *
     * @param element        the class, method, or field to inspect
     * @param annotationType the annotation type to search for
     * @return true if the annotation is present directly or as a meta-annotation
     */
    public static boolean hasAnnotation(AnnotatedElement element,
                                        Class<? extends Annotation> annotationType) {
        // Direct presence check — O(1) via JDK annotation cache
        if (element.isAnnotationPresent(annotationType)) {
            return true;
        }

        // Recursive meta-annotation check — walk the annotation tree.
        // A visited set prevents infinite loops from circular meta-annotations
        // (unlikely in practice, but defensive).
        Set<Class<? extends Annotation>> visited = new HashSet<>();
        for (Annotation ann : element.getAnnotations()) {
            if (hasMetaAnnotation(ann.annotationType(), annotationType, visited)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Recursively check if the given annotation type carries the target annotation
     * as a meta-annotation at any depth.
     *
     * <h3>ch17 Enhancement:</h3>
     * Extracted from {@code hasAnnotation()} to enable recursive walking.
     * Skips JDK built-in annotations (java.lang.annotation.*) which can never
     * carry our custom stereotype annotations.
     *
     * @param currentAnnotation the annotation type to inspect
     * @param targetAnnotation  the annotation type we're searching for
     * @param visited           set of already-visited annotation types (cycle prevention)
     * @return true if targetAnnotation is found in the meta-annotation hierarchy
     */
    private static boolean hasMetaAnnotation(Class<? extends Annotation> currentAnnotation,
                                              Class<? extends Annotation> targetAnnotation,
                                              Set<Class<? extends Annotation>> visited) {
        // Direct check
        if (currentAnnotation.isAnnotationPresent(targetAnnotation)) {
            return true;
        }

        // Prevent re-visiting the same annotation (cycle guard)
        if (!visited.add(currentAnnotation)) {
            return false;
        }

        // Recurse into meta-annotations, skipping JDK built-ins
        for (Annotation metaAnn : currentAnnotation.getAnnotations()) {
            Class<? extends Annotation> metaType = metaAnn.annotationType();

            // Skip java.lang.annotation.* — these are JDK meta-annotations
            // (@Target, @Retention, @Documented, @Inherited, @Repeatable)
            // that can never carry our custom annotations.
            if (metaType.getName().startsWith("java.lang.annotation")) {
                continue;
            }

            if (hasMetaAnnotation(metaType, targetAnnotation, visited)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Find the annotation of the given type on the element — either directly
     * present or as a meta-annotation on one of the element's annotations.
     *
     * Maps to: {@code AnnotatedElementUtils.findMergedAnnotation()} (line 631)
     * Real version returns a synthesized annotation proxy with merged attributes
     * from @AliasFor declarations. We return the raw annotation instance.
     *
     * <p>When found as a meta-annotation (e.g., @RequestMapping on @GetMapping),
     * the returned instance is the meta-annotation declaration — its attributes
     * reflect what was declared on the composed annotation definition, NOT what
     * the user wrote. For example, @GetMapping("/users") → the returned
     * @RequestMapping has method="GET" but path="" (the path is on @GetMapping itself).
     *
     * @param element        the class, method, or field to inspect
     * @param annotationType the annotation type to search for
     * @return the annotation instance, or null if not found
     */
    public static <A extends Annotation> A findAnnotation(AnnotatedElement element,
                                                           Class<A> annotationType) {
        // Direct presence — return the annotation as-is
        A direct = element.getAnnotation(annotationType);
        if (direct != null) {
            return direct;
        }

        // Meta-annotation — return the meta-annotation instance from
        // whichever composed annotation carries it
        for (Annotation ann : element.getAnnotations()) {
            A meta = ann.annotationType().getAnnotation(annotationType);
            if (meta != null) {
                return meta;
            }
        }

        return null;
    }

    /**
     * Find the composed annotation on the element that carries the given
     * meta-annotation type. Returns the "carrier" annotation (e.g., @GetMapping),
     * not the meta-annotation itself (e.g., @RequestMapping).
     *
     * This is useful when you need to read attributes from the composed annotation
     * that aren't on the meta-annotation — for example, reading the path from
     * @GetMapping("/users") when searching for @RequestMapping.
     *
     * Maps to: There's no direct equivalent in the real framework because
     * the MergedAnnotation API handles attribute merging transparently.
     * We need this because we don't implement @AliasFor.
     *
     * @param element            the class, method, or field to inspect
     * @param metaAnnotationType the meta-annotation type to search for
     * @return the composed annotation carrying the meta-annotation, or null
     */
    public static Annotation findComposedAnnotation(AnnotatedElement element,
                                                     Class<? extends Annotation> metaAnnotationType) {
        // If the annotation is directly present, it's not "composed" — skip
        if (element.isAnnotationPresent(metaAnnotationType)) {
            return null;
        }

        for (Annotation ann : element.getAnnotations()) {
            if (ann.annotationType().isAnnotationPresent(metaAnnotationType)) {
                return ann;
            }
        }

        return null;
    }

    /**
     * Extract a String attribute value from an annotation via reflection.
     *
     * This is our simplified substitute for Spring's @AliasFor mechanism.
     * Instead of synthesizing annotation proxies that merge attributes,
     * we read attributes directly from whichever annotation has them.
     *
     * Maps to: {@code MergedAnnotation.getString()} which reads attribute
     * values from the merged annotation view, honoring @AliasFor declarations.
     *
     * @param annotation    the annotation to read from
     * @param attributeName the name of the attribute to read
     * @return the attribute value, or empty string if not found
     */
    public static String getStringAttribute(Annotation annotation, String attributeName) {
        try {
            Method attrMethod = annotation.annotationType().getMethod(attributeName);
            Object value = attrMethod.invoke(annotation);
            return value instanceof String s ? s : "";
        } catch (NoSuchMethodException e) {
            return "";
        } catch (ReflectiveOperationException e) {
            throw new IllegalStateException(
                    "Failed to read attribute '" + attributeName + "' from " + annotation, e);
        }
    }

    /**
     * Extract a String[] attribute value from an annotation via reflection.
     *
     * <h3>ch14 Enhancement:</h3>
     * Added to support reading {@code produces()} and {@code consumes()} attributes
     * from composed annotations like @GetMapping. These return String[] instead of
     * String, so we need a separate method.
     *
     * Maps to: {@code MergedAnnotation.getStringArray()} which reads array attribute
     * values from the merged annotation view.
     *
     * @param annotation    the annotation to read from
     * @param attributeName the name of the attribute to read
     * @return the attribute value, or empty array if not found
     */
    public static String[] getStringArrayAttribute(Annotation annotation, String attributeName) {
        try {
            Method attrMethod = annotation.annotationType().getMethod(attributeName);
            Object value = attrMethod.invoke(annotation);
            if (value instanceof String[] arr) {
                return arr;
            }
            return new String[0];
        } catch (NoSuchMethodException e) {
            return new String[0];
        } catch (ReflectiveOperationException e) {
            throw new IllegalStateException(
                    "Failed to read attribute '" + attributeName + "' from " + annotation, e);
        }
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/scan/ClasspathScannerTest.java` [NEW]

```java
package com.simplespringmvc.scan;

import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.scan.testpackage.AdviceBean;
import com.simplespringmvc.scan.testpackage.AnnotatedController;
import com.simplespringmvc.scan.testpackage.AnnotatedRestController;
import com.simplespringmvc.scan.testpackage.DirectComponentBean;
import com.simplespringmvc.scan.testpackage.PlainClass;
import com.simplespringmvc.scan.testpackage.sub.SubPackageController;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Unit tests for ClasspathScanner — verifies component discovery and registration.
 */
class ClasspathScannerTest {

    private SimpleBeanContainer container;
    private ClasspathScanner scanner;

    @BeforeEach
    void setUp() {
        container = new SimpleBeanContainer();
        scanner = new ClasspathScanner(container);
    }

    // ── Discovery by annotation type ───────────────────────────────────

    @Test
    void shouldDiscoverControllerAnnotatedClass_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("annotatedController")).isTrue();
        assertThat(container.getBean("annotatedController"))
                .isInstanceOf(AnnotatedController.class);
    }

    @Test
    void shouldDiscoverRestControllerAnnotatedClass_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("annotatedRestController")).isTrue();
        assertThat(container.getBean("annotatedRestController"))
                .isInstanceOf(AnnotatedRestController.class);
    }

    @Test
    void shouldDiscoverDirectComponentAnnotatedClass_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("directComponentBean")).isTrue();
        assertThat(container.getBean("directComponentBean"))
                .isInstanceOf(DirectComponentBean.class);
    }

    @Test
    void shouldDiscoverControllerAdviceAnnotatedClass_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("adviceBean")).isTrue();
        assertThat(container.getBean("adviceBean"))
                .isInstanceOf(AdviceBean.class);
    }

    // ── Filtering non-components ───────────────────────────────────────

    @Test
    void shouldNotDiscoverPlainClass_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("plainClass")).isFalse();
    }

    @Test
    void shouldNotDiscoverAbstractClass_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("abstractComponent")).isFalse();
    }

    @Test
    void shouldNotDiscoverInterface_WhenScanningPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("componentInterface")).isFalse();
    }

    // ── Recursive scanning ─────────────────────────────────────────────

    @Test
    void shouldDiscoverClassesInSubPackages_WhenScanningParentPackage() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        assertThat(container.containsBean("subPackageController")).isTrue();
        assertThat(container.getBean("subPackageController"))
                .isInstanceOf(SubPackageController.class);
    }

    // ── Return count ───────────────────────────────────────────────────

    @Test
    void shouldReturnCountOfNewlyRegisteredBeans_WhenScanning() {
        int count = scanner.scan("com.simplespringmvc.scan.testpackage");

        // Should discover: AnnotatedController, AnnotatedRestController,
        // DirectComponentBean, AdviceBean, SubPackageController
        // Should NOT discover: PlainClass, AbstractComponent, ComponentInterface
        assertThat(count).isEqualTo(5);
    }

    // ── Skip already registered ────────────────────────────────────────

    @Test
    void shouldNotReRegisterBean_WhenAlreadyRegisteredManually() {
        // Register manually first
        AnnotatedController manualBean = new AnnotatedController();
        container.registerBean("annotatedController", manualBean);

        scanner.scan("com.simplespringmvc.scan.testpackage");

        // Should keep the manually registered instance, not replace it
        assertThat(container.getBean("annotatedController")).isSameAs(manualBean);
    }

    @Test
    void shouldReturnReducedCount_WhenSomeBeanAlreadyRegistered() {
        container.registerBean("annotatedController", new AnnotatedController());

        int count = scanner.scan("com.simplespringmvc.scan.testpackage");

        // One fewer because annotatedController was already registered
        assertThat(count).isEqualTo(4);
    }

    // ── Multiple packages ──────────────────────────────────────────────

    @Test
    void shouldScanMultiplePackages_WhenGivenMultipleBasePackages() {
        // Scan only the sub-package, then the parent with non-overlapping results
        int count = scanner.scan(
                "com.simplespringmvc.scan.testpackage.sub",
                "com.simplespringmvc.scan.testpackage"
        );

        // SubPackageController from first scan, rest from second
        // SubPackageController should not be double-counted
        assertThat(container.containsBean("subPackageController")).isTrue();
        assertThat(container.containsBean("annotatedController")).isTrue();
        assertThat(count).isEqualTo(5);
    }

    // ── Empty/nonexistent package ──────────────────────────────────────

    @Test
    void shouldReturnZero_WhenScanningNonexistentPackage() {
        int count = scanner.scan("com.nonexistent.package.that.does.not.exist");

        assertThat(count).isEqualTo(0);
        assertThat(container.getBeanNames()).isEmpty();
    }

    // ── Bean name derivation ───────────────────────────────────────────

    @Test
    void shouldDeriveBeanNameWithDecapitalizedSimpleName_WhenRegistering() {
        scanner.scan("com.simplespringmvc.scan.testpackage");

        // Verify the expected bean names follow Introspector.decapitalize() rules
        List<String> names = container.getBeanNames();
        assertThat(names).contains(
                "annotatedController",
                "annotatedRestController",
                "directComponentBean",
                "adviceBean",
                "subPackageController"
        );
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/ComponentScanningIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.scan.ClasspathScanner;
import com.simplespringmvc.scan.testpackage.AnnotatedController;
import com.simplespringmvc.scan.testpackage.AnnotatedRestController;
import com.simplespringmvc.scan.testpackage.sub.SubPackageController;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration test verifying that component scanning works end-to-end:
 * scan → register → handler mapping → dispatch → response.
 *
 * Previously, tests manually registered controllers in the BeanContainer.
 * This test uses ClasspathScanner instead, demonstrating the "no manual
 * registration" workflow.
 */
class ComponentScanningIntegrationTest {

    private EmbeddedTomcat tomcat;

    @AfterEach
    void tearDown() throws Exception {
        if (tomcat != null) {
            tomcat.stop();
        }
    }

    @Test
    void shouldHandleRequest_WhenControllerDiscoveredByScanning() throws Exception {
        // Arrange: scan instead of manually registering
        SimpleBeanContainer container = new SimpleBeanContainer();
        ClasspathScanner scanner = new ClasspathScanner(container);
        scanner.scan("com.simplespringmvc.scan.testpackage");

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        // Act: send request to the scanned controller's endpoint
        HttpClient client = HttpClient.newHttpClient();
        HttpResponse<String> response = client.send(
                HttpRequest.newBuilder()
                        .uri(URI.create("http://localhost:" + tomcat.getPort() + "/annotated/hello"))
                        .GET()
                        .build(),
                HttpResponse.BodyHandlers.ofString()
        );

        // Assert: the scanned controller handled the request
        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("hello from scanned controller");
    }

    @Test
    void shouldHandleRequest_WhenSubPackageControllerDiscoveredByScanning() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        ClasspathScanner scanner = new ClasspathScanner(container);
        scanner.scan("com.simplespringmvc.scan.testpackage");

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        HttpClient client = HttpClient.newHttpClient();
        HttpResponse<String> response = client.send(
                HttpRequest.newBuilder()
                        .uri(URI.create("http://localhost:" + tomcat.getPort() + "/sub/hello"))
                        .GET()
                        .build(),
                HttpResponse.BodyHandlers.ofString()
        );

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("hello from sub-package");
    }

    @Test
    void shouldRegisterHandlerMappings_WhenControllersDiscoveredByScanning() {
        SimpleBeanContainer container = new SimpleBeanContainer();
        ClasspathScanner scanner = new ClasspathScanner(container);
        scanner.scan("com.simplespringmvc.scan.testpackage");

        // Initialize handler mapping from the scanned beans
        SimpleHandlerMapping mapping = new SimpleHandlerMapping();
        mapping.init(container);

        // Verify handlers were registered for the scanned controllers
        HandlerMethod handler1 = mapping.lookupHandler("/annotated/hello", "GET");
        assertThat(handler1).isNotNull();
        assertThat(handler1.getBean()).isInstanceOf(AnnotatedController.class);

        HandlerMethod handler2 = mapping.lookupHandler("/api/data", "GET");
        assertThat(handler2).isNotNull();
        assertThat(handler2.getBean()).isInstanceOf(AnnotatedRestController.class);

        HandlerMethod handler3 = mapping.lookupHandler("/sub/hello", "GET");
        assertThat(handler3).isNotNull();
        assertThat(handler3.getBean()).isInstanceOf(SubPackageController.class);
    }

    @Test
    void shouldAllowMixingManualAndScannedRegistration() {
        SimpleBeanContainer container = new SimpleBeanContainer();

        // Manually register one controller
        AnnotatedController manual = new AnnotatedController();
        container.registerBean("annotatedController", manual);

        // Then scan — should not replace the manual bean
        ClasspathScanner scanner = new ClasspathScanner(container);
        scanner.scan("com.simplespringmvc.scan.testpackage");

        // Manual bean is preserved
        assertThat(container.getBean("annotatedController")).isSameAs(manual);

        // Other beans were still discovered
        assertThat(container.containsBean("annotatedRestController")).isTrue();
        assertThat(container.containsBean("subPackageController")).isTrue();
    }
}
```

#### Test Fixture Files

**`src/test/java/com/simplespringmvc/scan/testpackage/AnnotatedController.java`** [NEW] — `@Controller` with `@GetMapping("/hello")`
**`src/test/java/com/simplespringmvc/scan/testpackage/AnnotatedRestController.java`** [NEW] — `@RestController` with `@GetMapping("/api/data")`
**`src/test/java/com/simplespringmvc/scan/testpackage/DirectComponentBean.java`** [NEW] — directly annotated with `@Component`
**`src/test/java/com/simplespringmvc/scan/testpackage/AdviceBean.java`** [NEW] — `@ControllerAdvice` with `@ExceptionHandler`
**`src/test/java/com/simplespringmvc/scan/testpackage/PlainClass.java`** [NEW] — no annotation (should not be discovered)
**`src/test/java/com/simplespringmvc/scan/testpackage/AbstractComponent.java`** [NEW] — abstract `@Component` (should not be discovered)
**`src/test/java/com/simplespringmvc/scan/testpackage/ComponentInterface.java`** [NEW] — `@Component` interface (should not be discovered)
**`src/test/java/com/simplespringmvc/scan/testpackage/sub/SubPackageController.java`** [NEW] — `@Controller` in sub-package (tests recursive scanning)

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@Component`** | Base stereotype annotation marking a class as eligible for classpath scanning |
| **Stereotype hierarchy** | `@Controller → @Component`, `@ControllerAdvice → @Component` — meta-annotations create a discoverable type tree |
| **Recursive meta-annotation walking** | Searching annotation chains at any depth (e.g., `@RestController → @Controller → @Component`) with cycle detection |
| **`ClasspathScanner`** | Converts package names to filesystem paths, walks directories/JARs for `.class` files, loads and filters candidates, then registers them in the `BeanContainer` |
| **Candidate component check** | Two-part: (1) has `@Component` via any meta-annotation chain, (2) is concrete and independent (not abstract, not interface, not inner class) |

**Next: Chapter 18 — Session Management & Flash Attributes** — Support multi-step form wizards with `@SessionAttributes` and the Post/Redirect/Get pattern via flash attributes that survive exactly one redirect
