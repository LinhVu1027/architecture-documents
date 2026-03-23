# Chapter 10: Composed Annotations

> **Source version:** Spring Framework commit `11ab0b4351`
> **Feature ring:** PIPELINE — Cross-Cutting Behavior Around Dispatch

---

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| Handler mapping detects `@Controller` and `@RequestMapping` via direct annotation checks | Cannot see annotations through meta-annotation layers — `@RestController` and `@GetMapping` are invisible | Walk the meta-annotation hierarchy so composed annotations work transparently |

**What you'll build:** A `MergedAnnotationUtils` utility that walks annotation hierarchies, `@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping` composed annotations, and `@RestController` — all wired into the existing handler mapping and return value handler pipeline.

---

## 10.1 The Integration Point: Where Annotations Are Checked

The feature connects at **two seams** — the two places where our code currently uses Java's built-in `isAnnotationPresent()` and `getAnnotation()`, which can only see annotations directly on the element:

### Seam 1: `SimpleHandlerMapping` — handler detection + method scanning

**Modifying:** `src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java`

Two methods need to change:

**`isHandler()`** — currently blind to `@RestController`:

```java
// BEFORE (ch03): direct check only
private boolean isHandler(Class<?> beanType) {
    return beanType.isAnnotationPresent(Controller.class);
}

// AFTER (ch10): walks meta-annotation hierarchy
private boolean isHandler(Class<?> beanType) {
    return MergedAnnotationUtils.hasAnnotation(beanType, Controller.class);
}
```

**`detectHandlerMethods()`** — currently blind to `@GetMapping`:

```java
// BEFORE (ch03): direct @RequestMapping check only
RequestMapping methodMapping = method.getAnnotation(RequestMapping.class);
if (methodMapping == null) {
    continue;
}
String fullPath = combinePaths(basePath, methodMapping.path());
String httpMethod = methodMapping.method();

// AFTER (ch10): also checks for composed annotations
RequestMapping directMapping = method.getAnnotation(RequestMapping.class);
if (directMapping != null) {
    path = directMapping.path();
    httpMethod = directMapping.method();
} else {
    // Look for composed annotation (e.g., @GetMapping) meta-annotated with @RequestMapping
    Annotation composed = MergedAnnotationUtils.findComposedAnnotation(
            method, RequestMapping.class);
    if (composed == null) {
        continue;
    }
    // HTTP method from the meta-annotation: @GetMapping → @RequestMapping(method = "GET")
    RequestMapping metaMapping = composed.annotationType().getAnnotation(RequestMapping.class);
    httpMethod = metaMapping.method();
    // Path from the composed annotation's value(): @GetMapping("/users") → "/users"
    path = MergedAnnotationUtils.getStringAttribute(composed, "value");
}
```

### Seam 2: `ResponseBodyReturnValueHandler` — class-level `@ResponseBody` detection

**Modifying:** `src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java`

```java
// BEFORE (ch06): direct check only
return handlerMethod.getBeanType().isAnnotationPresent(ResponseBody.class);

// AFTER (ch10): walks meta-annotation hierarchy
return MergedAnnotationUtils.hasAnnotation(
        handlerMethod.getBeanType(), ResponseBody.class);
```

**Direction:** Both seams need a utility that can "see through" composed annotations to find their meta-annotations. This is `MergedAnnotationUtils` — the simplified version of Spring's `AnnotatedElementUtils`.

---

## 10.2 The Meta-Annotation Walker: `MergedAnnotationUtils`

**Creating:** `src/main/java/com/simplespringmvc/annotation/MergedAnnotationUtils.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;

public class MergedAnnotationUtils {

    /**
     * Check if annotationType is present on element — either directly
     * or as a meta-annotation (on one of element's annotations).
     */
    public static boolean hasAnnotation(AnnotatedElement element,
                                        Class<? extends Annotation> annotationType) {
        // Direct presence check
        if (element.isAnnotationPresent(annotationType)) {
            return true;
        }
        // Meta-annotation check: for each annotation on element,
        // check if IT has annotationType
        for (Annotation ann : element.getAnnotations()) {
            if (ann.annotationType().isAnnotationPresent(annotationType)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Find the annotation of the given type — directly or as a meta-annotation.
     * Returns the annotation instance from where it was found.
     */
    public static <A extends Annotation> A findAnnotation(AnnotatedElement element,
                                                           Class<A> annotationType) {
        A direct = element.getAnnotation(annotationType);
        if (direct != null) {
            return direct;
        }
        for (Annotation ann : element.getAnnotations()) {
            A meta = ann.annotationType().getAnnotation(annotationType);
            if (meta != null) {
                return meta;
            }
        }
        return null;
    }

    /**
     * Find the composed annotation that carries the given meta-annotation type.
     * Returns the "carrier" annotation (e.g., @GetMapping), not the
     * meta-annotation itself (e.g., @RequestMapping).
     */
    public static Annotation findComposedAnnotation(AnnotatedElement element,
                                                     Class<? extends Annotation> metaAnnotationType) {
        if (element.isAnnotationPresent(metaAnnotationType)) {
            return null; // directly present, not composed
        }
        for (Annotation ann : element.getAnnotations()) {
            if (ann.annotationType().isAnnotationPresent(metaAnnotationType)) {
                return ann;
            }
        }
        return null;
    }

    /**
     * Extract a String attribute from an annotation via reflection.
     * Our simplified substitute for Spring's @AliasFor mechanism.
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
}
```

Three methods, each serving a specific purpose:

| Method | Returns | Used by |
|---|---|---|
| `hasAnnotation()` | `boolean` — is this annotation anywhere in the hierarchy? | `isHandler()`, `supportsReturnType()` |
| `findAnnotation()` | The annotation instance (direct or meta) | General-purpose lookup |
| `findComposedAnnotation()` | The carrier annotation (e.g., `@GetMapping`) | `detectHandlerMethods()` — needs the carrier to read its `value()` |

---

## 10.3 The Composed Annotations

### `@GetMapping` — the prototype for all HTTP method shortcuts

**Creating:** `src/main/java/com/simplespringmvc/annotation/GetMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "GET")        // ← the meta-annotation — this is the key line
public @interface GetMapping {

    /** The URL path. Equivalent to @RequestMapping's path attribute. */
    String value() default "";
}
```

The `@RequestMapping(method = "GET")` line is the entire magic. When `MergedAnnotationUtils` examines `@GetMapping`, it finds `@RequestMapping` on it and reads `method = "GET"` from there.

### `@PostMapping`, `@PutMapping`, `@DeleteMapping`

Identical structure — only the HTTP method changes:

**Creating:** `src/main/java/com/simplespringmvc/annotation/PostMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "POST")
public @interface PostMapping {
    String value() default "";
}
```

**Creating:** `src/main/java/com/simplespringmvc/annotation/PutMapping.java`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "PUT")
public @interface PutMapping {
    String value() default "";
}
```

**Creating:** `src/main/java/com/simplespringmvc/annotation/DeleteMapping.java`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "DELETE")
public @interface DeleteMapping {
    String value() default "";
}
```

### `@RestController` — two meta-annotations in one

**Creating:** `src/main/java/com/simplespringmvc/annotation/RestController.java`

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
@Controller          // ← detected by SimpleHandlerMapping.isHandler()
@ResponseBody        // ← detected by ResponseBodyReturnValueHandler.supportsReturnType()
public @interface RestController {
}
```

`@RestController` carries TWO meta-annotations. The handler mapping sees `@Controller`, and the return value handler sees `@ResponseBody`. Each subsystem sees only what it needs — the annotation acts as a bridge between them.

---

## 10.4 Try It Yourself

<details><summary>Challenge 1: Detect @RestController as a handler</summary>

Without looking at the integration point code above, modify `isHandler()` to use `MergedAnnotationUtils.hasAnnotation()` so that `@RestController` classes are detected.

```java
private boolean isHandler(Class<?> beanType) {
    return MergedAnnotationUtils.hasAnnotation(beanType, Controller.class);
}
```

</details>

<details><summary>Challenge 2: Extract path and method from @GetMapping</summary>

Given a method annotated with `@GetMapping("/users")`, write code to extract both the path (`"/users"`) and the HTTP method (`"GET"`).

Hint: The path is on the composed annotation (`@GetMapping.value()`), but the HTTP method is on the meta-annotation (`@RequestMapping.method()`). You need to read from both.

```java
// Find the carrier annotation (@GetMapping)
Annotation composed = MergedAnnotationUtils.findComposedAnnotation(method, RequestMapping.class);

// HTTP method from @RequestMapping meta-annotation on @GetMapping
RequestMapping metaMapping = composed.annotationType().getAnnotation(RequestMapping.class);
String httpMethod = metaMapping.method();  // "GET"

// Path from @GetMapping's value() attribute
String path = MergedAnnotationUtils.getStringAttribute(composed, "value");  // "/users"
```

</details>

<details><summary>Challenge 3: Make @RestController work with JSON responses</summary>

Modify `ResponseBodyReturnValueHandler.supportsReturnType()` so that `@RestController` classes (which carry `@ResponseBody` as a meta-annotation) automatically get JSON serialization.

```java
@Override
public boolean supportsReturnType(HandlerMethod handlerMethod) {
    if (handlerMethod.getMethod().isAnnotationPresent(ResponseBody.class)) {
        return true;
    }
    return MergedAnnotationUtils.hasAnnotation(
            handlerMethod.getBeanType(), ResponseBody.class);
}
```

</details>

---

## 10.5 Tests

### Unit Tests: `MergedAnnotationUtilsTest`

**Creating:** `src/test/java/com/simplespringmvc/annotation/MergedAnnotationUtilsTest.java`

```java
package com.simplespringmvc.annotation;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

class MergedAnnotationUtilsTest {

    // Test annotations
    @Retention(RetentionPolicy.RUNTIME)
    @interface BaseAnnotation { String value() default ""; }

    @Retention(RetentionPolicy.RUNTIME)
    @BaseAnnotation("from-composed")
    @interface ComposedAnnotation { String value() default ""; }

    @Retention(RetentionPolicy.RUNTIME)
    @interface UnrelatedAnnotation {}

    // Test classes
    @BaseAnnotation static class DirectlyAnnotated {}
    @ComposedAnnotation static class MetaAnnotated {}
    @UnrelatedAnnotation static class UnrelatedOnly {}
    static class NoAnnotations {}

    @Nested @DisplayName("hasAnnotation()")
    class HasAnnotation {
        @Test void shouldFindDirectAnnotation_WhenPresentOnClass() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    DirectlyAnnotated.class, BaseAnnotation.class)).isTrue();
        }

        @Test void shouldFindMetaAnnotation_WhenPresentOnComposedAnnotation() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    MetaAnnotated.class, BaseAnnotation.class)).isTrue();
        }

        @Test void shouldReturnFalse_WhenAnnotationAbsent() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    UnrelatedOnly.class, BaseAnnotation.class)).isFalse();
        }
    }

    @Nested @DisplayName("With real annotations")
    class RealAnnotations {
        @RestController static class RestCtrl {}

        @Test void shouldDetectControllerViaRestController() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    RestCtrl.class, Controller.class)).isTrue();
        }

        @Test void shouldDetectResponseBodyViaRestController() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    RestCtrl.class, ResponseBody.class)).isTrue();
        }

        @Test void shouldFindRequestMappingViaGetMapping() throws Exception {
            class TestCtrl { @GetMapping("/users") public void get() {} }
            Method m = TestCtrl.class.getMethod("get");
            RequestMapping found = MergedAnnotationUtils.findAnnotation(m, RequestMapping.class);
            assertThat(found).isNotNull();
            assertThat(found.method()).isEqualTo("GET");
        }
    }
}
```

### Integration Tests: `ComposedAnnotationsIntegrationTest`

**Creating:** `src/test/java/com/simplespringmvc/integration/ComposedAnnotationsIntegrationTest.java`

Key test cases:

```java
@Test void shouldRegisterGetMapping() {
    beanContainer.registerBean(new ComposedController());
    handlerMapping.init(beanContainer);
    assertThat(handlerMapping.lookupHandler("/api/items", "GET")).isNotNull();
}

@Test void shouldDetectRestControllerAsHandler() {
    beanContainer.registerBean(new RestApiController());
    handlerMapping.init(beanContainer);
    assertThat(handlerMapping.getRegisteredMappings()).hasSize(2);
}

@Test void shouldSerializeAsJson_WhenRestController() throws Exception {
    // ... sets up mock request/response, dispatches through full pipeline
    assertThat(json).contains("\"name\"").contains("\"Alice\"");
    verify(response).setContentType("application/json");
}
```

Run: `cd simple-spring-mvc && ./gradlew test`

---

## 10.6 Why This Works

### The Annotation Walk Algorithm

When `hasAnnotation(element, Controller.class)` is called on a `@RestController` class:

```
Step 1: element.isAnnotationPresent(Controller.class) → false
        (@Controller is NOT directly on the class)

Step 2: for each annotation on element:
          @RestController → @RestController.annotationType()
                            .isAnnotationPresent(Controller.class) → true!
        (because @RestController is annotated with @Controller)
```

The walk is one level deep: element → annotation → meta-annotation. This is sufficient because Spring's standard composed annotations are always one hop deep.

`★ Insight ─────────────────────────────────────`
**Why `findComposedAnnotation()` exists separately from `findAnnotation()`:** When processing `@GetMapping("/users")`, we need data from TWO annotations — the path from `@GetMapping` and the HTTP method from the `@RequestMapping` meta-annotation. `findAnnotation()` returns the meta-annotation (which has `method = "GET"` but `path = ""`). `findComposedAnnotation()` returns the carrier (which has `value = "/users"`). In the real framework, `@AliasFor` makes this invisible — `MergedAnnotation<RequestMapping>` synthesizes a proxy where `path()` returns `"/users"` and `method()` returns `GET`, merging attributes from both levels. Our simplified version reads both levels explicitly, making the data flow visible.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**The power of annotation composition:** `@RestController` demonstrates why meta-annotations are so useful. A single annotation replaces `@Controller` + `@ResponseBody` everywhere it's used. But the deeper value is in extensibility — any Spring user can create `@AdminController` meta-annotated with `@RestController` + `@RequestMapping("/admin")`, and the framework handles it with zero code changes. The generic annotation walker (`MergedAnnotationUtils`) makes this possible without hard-coding any specific annotation types.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Attribute merging vs. attribute forwarding:** Our `getStringAttribute()` reads attributes via raw reflection — it's a manual attribute forward. The real `@AliasFor` mechanism is declarative and validated at startup. If you declare `@AliasFor(annotation = RequestMapping.class)` on an attribute that doesn't exist on `@RequestMapping`, Spring fails fast with a clear error. Our approach silently returns `""`. This is a deliberate simplification — implementing `@AliasFor` would require annotation proxying (JDK dynamic proxies for annotation interfaces), which is a feature unto itself.
`─────────────────────────────────────────────────`

---

## 10.7 What We Enhanced

| File | Change | Why |
|---|---|---|
| `SimpleHandlerMapping.isHandler()` | `isAnnotationPresent()` → `MergedAnnotationUtils.hasAnnotation()` | Detect `@RestController` (meta-annotated with `@Controller`) |
| `SimpleHandlerMapping.detectHandlerMethods()` | Added composed annotation fallback branch | Detect `@GetMapping`/`@PostMapping`/etc. (meta-annotated with `@RequestMapping`) |
| `ResponseBodyReturnValueHandler.supportsReturnType()` | Class-level check uses `MergedAnnotationUtils.hasAnnotation()` | Detect `@ResponseBody` on `@RestController` classes |

---

## 10.8 Connection to Real Framework

| Simplified | Real Framework | File:Line |
|---|---|---|
| `MergedAnnotationUtils.hasAnnotation()` | `AnnotatedElementUtils.hasAnnotation()` | `AnnotatedElementUtils.java:537` |
| `MergedAnnotationUtils.findAnnotation()` | `AnnotatedElementUtils.findMergedAnnotation()` | `AnnotatedElementUtils.java:631` |
| `MergedAnnotationUtils.getStringAttribute()` | `@AliasFor` + `MergedAnnotation.getString()` | `AliasFor.java`, `MergedAnnotation.java` |
| `@GetMapping` | `@GetMapping` | `GetMapping.java:52` |
| `@RestController` | `@RestController` | `RestController.java:30` |
| `isHandler()` with meta-annotation walk | `RequestMappingHandlerMapping.isHandler()` | `RequestMappingHandlerMapping.java:181` |
| Composed annotation detection in `detectHandlerMethods()` | `createRequestMappingInfo()` via `MergedAnnotations` | `RequestMappingHandlerMapping.java:269` |

Commit: `11ab0b4351`

---

## 10.9 Complete Code

### Production Code

#### `[NEW] src/main/java/com/simplespringmvc/annotation/MergedAnnotationUtils.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Annotation;
import java.lang.reflect.AnnotatedElement;
import java.lang.reflect.Method;

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
 * <h3>Simplifications:</h3>
 * <ul>
 *   <li>One-level-deep meta-annotation search (enough for @GetMapping → @RequestMapping)</li>
 *   <li>No @AliasFor attribute merging — we extract attributes via reflection</li>
 *   <li>No SearchStrategy (no superclass/interface traversal)</li>
 *   <li>No annotation synthesis — we return the raw annotation instances</li>
 *   <li>No caching of resolved annotations</li>
 * </ul>
 */
public class MergedAnnotationUtils {

    /**
     * Check if an annotation type is present on the element — either directly
     * or as a meta-annotation on one of the element's annotations.
     *
     * Maps to: {@code AnnotatedElementUtils.hasAnnotation()} (line 537)
     * Real version uses find semantics (TYPE_HIERARCHY search), walking
     * superclasses, interfaces, and the full meta-annotation depth.
     * We search one level deep, which covers all standard composed annotations.
     *
     * Examples:
     * <ul>
     *   <li>{@code hasAnnotation(MyController.class, Controller.class)} → true
     *       if class has @Controller directly</li>
     *   <li>{@code hasAnnotation(MyRestController.class, Controller.class)} → true
     *       if class has @RestController (which is meta-annotated with @Controller)</li>
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

        // Meta-annotation check — look at each annotation on the element,
        // and check if THAT annotation is itself annotated with annotationType.
        // This is the core of the meta-annotation walking pattern.
        for (Annotation ann : element.getAnnotations()) {
            if (ann.annotationType().isAnnotationPresent(annotationType)) {
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
}
```

#### `[NEW] src/main/java/com/simplespringmvc/annotation/GetMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "GET")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.GetMapping}
 *
 * This is a <b>composed annotation</b> — it carries {@code @RequestMapping(method = "GET")}
 * as a meta-annotation. When {@link com.simplespringmvc.mapping.SimpleHandlerMapping}
 * scans for handler methods, it finds {@code @RequestMapping} by walking up
 * through this annotation's meta-annotations.
 *
 * <h3>How attribute merging works in the real framework:</h3>
 * <pre>
 *   @GetMapping("/users")
 *   ↓ @AliasFor forwards value → @RequestMapping.path
 *   ↓ @RequestMapping.method is fixed to GET
 *   → MergedAnnotation&lt;RequestMapping&gt; with path="/users", method=GET
 * </pre>
 *
 * <h3>How our simplified version works:</h3>
 * <pre>
 *   @GetMapping("/users")
 *   ↓ SimpleHandlerMapping reads value() from @GetMapping as the path
 *   ↓ SimpleHandlerMapping reads method() from the @RequestMapping meta-annotation
 *   → path="/users", method="GET"
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Uses value() instead of @AliasFor to forward path</li>
 *   <li>No params, headers, consumes, produces attributes</li>
 * </ul>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "GET")
public @interface GetMapping {

    /**
     * The URL path for this mapping.
     * Equivalent to {@link RequestMapping#path()}.
     *
     * In the real framework, this is declared with
     * {@code @AliasFor(annotation = RequestMapping.class, attribute = "path")}
     * which transparently forwards the value to @RequestMapping's path attribute.
     * We read it manually in SimpleHandlerMapping.
     */
    String value() default "";
}
```

#### `[NEW] src/main/java/com/simplespringmvc/annotation/PostMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "POST")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.PostMapping}
 *
 * A composed annotation meta-annotated with {@code @RequestMapping(method = "POST")}.
 * See {@link GetMapping} for a detailed explanation of how composed annotations work.
 *
 * Simplifications:
 * <ul>
 *   <li>Uses value() instead of @AliasFor to forward path</li>
 *   <li>No params, headers, consumes, produces attributes</li>
 * </ul>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "POST")
public @interface PostMapping {

    /**
     * The URL path for this mapping.
     * Equivalent to {@link RequestMapping#path()}.
     */
    String value() default "";
}
```

#### `[NEW] src/main/java/com/simplespringmvc/annotation/PutMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "PUT")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.PutMapping}
 *
 * A composed annotation meta-annotated with {@code @RequestMapping(method = "PUT")}.
 * See {@link GetMapping} for a detailed explanation of how composed annotations work.
 *
 * Simplifications:
 * <ul>
 *   <li>Uses value() instead of @AliasFor to forward path</li>
 *   <li>No params, headers, consumes, produces attributes</li>
 * </ul>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "PUT")
public @interface PutMapping {

    /**
     * The URL path for this mapping.
     * Equivalent to {@link RequestMapping#path()}.
     */
    String value() default "";
}
```

#### `[NEW] src/main/java/com/simplespringmvc/annotation/DeleteMapping.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Shortcut for {@code @RequestMapping(method = "DELETE")}.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.DeleteMapping}
 *
 * A composed annotation meta-annotated with {@code @RequestMapping(method = "DELETE")}.
 * See {@link GetMapping} for a detailed explanation of how composed annotations work.
 *
 * Simplifications:
 * <ul>
 *   <li>Uses value() instead of @AliasFor to forward path</li>
 *   <li>No params, headers, consumes, produces attributes</li>
 * </ul>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = "DELETE")
public @interface DeleteMapping {

    /**
     * The URL path for this mapping.
     * Equivalent to {@link RequestMapping#path()}.
     */
    String value() default "";
}
```

#### `[NEW] src/main/java/com/simplespringmvc/annotation/RestController.java`

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
 * </ul>
 *
 * <h3>The meta-annotation chain:</h3>
 * <pre>
 *   @RestController
 *     ├── @Controller    → detected by SimpleHandlerMapping.isHandler()
 *     └── @ResponseBody  → detected by ResponseBodyReturnValueHandler.supportsReturnType()
 * </pre>
 *
 * In the real framework, @RestController also carries @Component (via @Controller)
 * and @Indexed for AOT support. The value() attribute is aliased to
 * Controller.value() → Component.value() for bean naming.
 *
 * Simplifications:
 * <ul>
 *   <li>No @Component meta-annotation chain (no classpath scanning yet — ch17)</li>
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

#### `[MODIFIED] src/main/java/com/simplespringmvc/mapping/SimpleHandlerMapping.java`

```java
package com.simplespringmvc.mapping;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.BeanContainer;
import jakarta.servlet.http.HttpServletRequest;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Scans all beans in the container for @Controller classes, discovers methods
 * annotated with @RequestMapping, and builds a registry that maps
 * (URL pattern + HTTP method) → HandlerMethod.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping}
 * which extends {@code AbstractHandlerMethodMapping<RequestMappingInfo>}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. afterPropertiesSet() → initHandlerMethods()
 *   2. initHandlerMethods() iterates all beans, calls isHandler() on each
 *   3. isHandler() checks for @Controller annotation (via MergedAnnotations)
 *   4. detectHandlerMethods() introspects each handler class for @RequestMapping methods
 *   5. getMappingForMethod() builds a RequestMappingInfo from the annotation
 *   6. registerHandlerMethod() stores (RequestMappingInfo → HandlerMethod) in MappingRegistry
 *   7. At request time, lookupHandlerMethod() matches the request against all registrations
 * </pre>
 *
 * We collapse steps 1-7 into a simpler flow:
 * <pre>
 *   1. init(BeanContainer) iterates all beans
 *   2. Checks each bean class for @Controller
 *   3. For each @Controller, scans methods for @RequestMapping
 *   4. Combines type-level + method-level paths
 *   5. Stores (RouteKey → HandlerMethod) in a LinkedHashMap
 *   6. lookupHandler() matches request path + method against RouteKeys
 * </pre>
 *
 * <h3>ch07 Enhancement:</h3>
 * Handler mapping now supports URI templates with path variables (e.g., {@code /users/{id}}).
 * When a pattern match succeeds, extracted path variables are stored as a request
 * attribute under {@link #PATH_VARIABLES_ATTRIBUTE}, making them available to the
 * {@code PathVariableArgumentResolver} during method invocation.
 *
 * Simplifications:
 * <ul>
 *   <li>No MappingRegistry inner class with ReadWriteLock</li>
 *   <li>No RequestMappingInfo composite — just RouteKey (path + method)</li>
 *   <li>No direct-path optimization (we scan all entries on every request)</li>
 *   <li>No HandlerExecutionChain wrapping (added in ch11)</li>
 * </ul>
 *
 * <h3>ch10 Enhancement:</h3>
 * Handler detection and method scanning now use {@link MergedAnnotationUtils}
 * to walk meta-annotation hierarchies. This enables:
 * <ul>
 *   <li>{@code @RestController} classes to be detected as handlers
 *       (because @RestController is meta-annotated with @Controller)</li>
 *   <li>{@code @GetMapping}, {@code @PostMapping}, etc. to be detected as
 *       request mappings (because they are meta-annotated with @RequestMapping)</li>
 * </ul>
 */
public class SimpleHandlerMapping {

    /**
     * Request attribute key for extracted path variables.
     *
     * Maps to: {@code HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE} which is
     * {@code "org.springframework.web.servlet.HandlerMapping.uriTemplateVariables"}
     *
     * The real framework sets this in AbstractHandlerMethodMapping.handleMatch()
     * after a successful pattern match. PathVariableMethodArgumentResolver.resolveName()
     * reads it back to resolve @PathVariable arguments.
     */
    public static final String PATH_VARIABLES_ATTRIBUTE =
            "com.simplespringmvc.mapping.SimpleHandlerMapping.pathVariables";

    private final Map<RouteKey, HandlerMethod> registry = new LinkedHashMap<>();

    /**
     * Scan all beans in the container and register handler methods.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.initHandlerMethods()} (line 217)
     * which is called from afterPropertiesSet().
     */
    public void init(BeanContainer beanContainer) {
        for (String beanName : beanContainer.getBeanNames()) {
            Object bean = beanContainer.getBean(beanName);
            if (isHandler(bean.getClass())) {
                detectHandlerMethods(bean);
            }
        }
    }

    /**
     * Check if a bean type is a handler (i.e., annotated with @Controller
     * either directly or via a composed annotation like @RestController).
     *
     * Maps to: {@code RequestMappingHandlerMapping.isHandler()} (line 181)
     * Real version:
     * <pre>
     *   return AnnotatedElementUtils.hasAnnotation(beanType, Controller.class);
     * </pre>
     *
     * <h3>ch10 Enhancement:</h3>
     * Now uses {@link MergedAnnotationUtils#hasAnnotation} to walk the
     * meta-annotation hierarchy. This detects @RestController (which is
     * meta-annotated with @Controller) in addition to direct @Controller.
     */
    private boolean isHandler(Class<?> beanType) {
        return MergedAnnotationUtils.hasAnnotation(beanType, Controller.class);
    }

    /**
     * Discover all @RequestMapping methods on a handler bean and register them.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.detectHandlerMethods()} (line 270)
     * Real version uses MethodIntrospector.selectMethods() to scan the class,
     * then calls getMappingForMethod() for each method.
     *
     * <h3>ch10 Enhancement:</h3>
     * Now detects composed annotations (@GetMapping, @PostMapping, etc.) in addition
     * to direct @RequestMapping. When a composed annotation is found:
     * <ul>
     *   <li>The HTTP method comes from the @RequestMapping meta-annotation
     *       (e.g., "GET" from @GetMapping's @RequestMapping(method = "GET"))</li>
     *   <li>The path comes from the composed annotation's value() attribute
     *       (e.g., "/users" from @GetMapping("/users"))</li>
     * </ul>
     */
    private void detectHandlerMethods(Object bean) {
        Class<?> beanType = bean.getClass();

        // Check for type-level @RequestMapping (base path prefix)
        String basePath = "";
        RequestMapping typeMapping = beanType.getAnnotation(RequestMapping.class);
        if (typeMapping != null) {
            basePath = typeMapping.path();
        }

        // Scan all declared methods for @RequestMapping (direct or composed)
        for (Method method : beanType.getDeclaredMethods()) {
            String path;
            String httpMethod;

            // Try direct @RequestMapping first
            RequestMapping directMapping = method.getAnnotation(RequestMapping.class);
            if (directMapping != null) {
                path = directMapping.path();
                httpMethod = directMapping.method();
            } else {
                // ch10: Look for composed annotations meta-annotated with @RequestMapping.
                // findComposedAnnotation() returns the carrier annotation (e.g., @GetMapping),
                // not the meta-annotation (@RequestMapping) — we need both:
                //   - The carrier's value() for the path
                //   - The meta-annotation's method() for the HTTP method
                Annotation composed = MergedAnnotationUtils.findComposedAnnotation(
                        method, RequestMapping.class);
                if (composed == null) {
                    continue;
                }

                // HTTP method from the @RequestMapping meta-annotation on the composed annotation.
                // e.g., @GetMapping is annotated with @RequestMapping(method = "GET"),
                // so metaMapping.method() returns "GET".
                RequestMapping metaMapping = composed.annotationType()
                        .getAnnotation(RequestMapping.class);
                httpMethod = metaMapping.method();

                // Path from the composed annotation's value() attribute.
                // e.g., @GetMapping("/users") → value() returns "/users".
                // In the real framework, @AliasFor transparently forwards this to
                // @RequestMapping.path(). We read it manually here.
                path = MergedAnnotationUtils.getStringAttribute(composed, "value");
            }

            // Combine type-level base path + method-level path
            String fullPath = combinePaths(basePath, path);

            // Ensure the method is accessible via reflection.
            // Maps to: ReflectionUtils.makeAccessible() used in Spring's
            // InvocableHandlerMethod. Needed when the declaring class
            // is package-private (common in tests, inner classes, etc.)
            method.setAccessible(true);

            RouteKey routeKey = new RouteKey(fullPath, httpMethod);
            HandlerMethod handlerMethod = new HandlerMethod(bean, method);

            // Check for duplicate mappings
            if (registry.containsKey(routeKey)) {
                HandlerMethod existing = registry.get(routeKey);
                throw new IllegalStateException(
                        "Ambiguous mapping: " + routeKey + " is already mapped to " + existing
                                + ". Cannot map " + handlerMethod);
            }

            registry.put(routeKey, handlerMethod);
        }
    }

    /**
     * Look up the handler method for a given request path and HTTP method.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.lookupHandlerMethod()} (line 393)
     *
     * The real version first tries a fast direct-path lookup, then falls back
     * to scanning all mappings. We always scan (simple but O(n)).
     *
     * @return the matching HandlerMethod, or null if no handler found
     */
    public HandlerMethod lookupHandler(String requestPath, String requestMethod) {
        for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
            RouteKey key = entry.getKey();
            if (key.matches(requestPath, requestMethod)) {
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Look up the handler method for a given HTTP request, storing extracted
     * path variables as a request attribute.
     *
     * Maps to: {@code AbstractHandlerMethodMapping.getHandlerInternal()} (line 337)
     * which calls lookupHandlerMethod(), then handleMatch() which stores the
     * extracted URI template variables as a request attribute under
     * {@code HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE}.
     *
     * <h3>ch07 Enhancement:</h3>
     * This overload takes the full HttpServletRequest so it can:
     * <ol>
     *   <li>Match the request path against pattern-based RouteKeys</li>
     *   <li>Extract path variable values (e.g., {@code {id}} → "42")</li>
     *   <li>Store them as a request attribute for the PathVariableArgumentResolver</li>
     * </ol>
     *
     * @return the matching HandlerMethod, or null if no handler found
     */
    public HandlerMethod lookupHandler(HttpServletRequest request) {
        String requestPath = request.getRequestURI();
        String requestMethod = request.getMethod();

        for (Map.Entry<RouteKey, HandlerMethod> entry : registry.entrySet()) {
            RouteKey key = entry.getKey();
            Map<String, String> pathVariables = key.matchAndExtract(requestPath, requestMethod);
            if (pathVariables != null) {
                // Store extracted path variables on the request for argument resolvers.
                // Maps to: AbstractHandlerMethodMapping.handleMatch() which calls
                // request.setAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, uriVariables)
                request.setAttribute(PATH_VARIABLES_ATTRIBUTE, pathVariables);
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * Returns an unmodifiable view of all registered mappings.
     * Useful for debugging and testing.
     */
    public Map<RouteKey, HandlerMethod> getRegisteredMappings() {
        return Map.copyOf(registry);
    }

    /**
     * Combine a base path and method path.
     * Handles cases like "/" + "/users" → "/users", "/api" + "/users" → "/api/users".
     */
    private String combinePaths(String basePath, String methodPath) {
        if (basePath == null || basePath.isEmpty()) {
            return methodPath;
        }
        if (methodPath == null || methodPath.isEmpty()) {
            return basePath;
        }
        // Remove trailing slash from base, ensure method starts with /
        String base = basePath.endsWith("/") ? basePath.substring(0, basePath.length() - 1) : basePath;
        String method = methodPath.startsWith("/") ? methodPath : "/" + methodPath;
        return base + method;
    }
}
```

#### `[MODIFIED] src/main/java/com/simplespringmvc/adapter/ResponseBodyReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.MergedAnnotationUtils;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

/**
 * Handles return values from handler methods annotated with {@link ResponseBody}
 * by serializing the return value through an {@link HttpMessageConverter}.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor}
 *
 * The real version:
 * <ul>
 *   <li>Implements both {@code HandlerMethodArgumentResolver} (for @RequestBody) and
 *       {@code HandlerMethodReturnValueHandler} (for @ResponseBody)</li>
 *   <li>Extends {@code AbstractMessageConverterMethodProcessor} which handles
 *       content negotiation and converter selection</li>
 *   <li>Caches class-level @ResponseBody checks in a ConcurrentHashMap</li>
 *   <li>Uses AnnotatedElementUtils for meta-annotation detection (for @RestController)</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Return value handling only — no @RequestBody argument resolution (ch08)</li>
 *   <li>No content negotiation — uses the first compatible converter (ch14)</li>
 *   <li>No class-level caching (not needed at this scale)</li>
 * </ul>
 */
public class ResponseBodyReturnValueHandler implements HandlerMethodReturnValueHandler {

    private final List<HttpMessageConverter> messageConverters;

    public ResponseBodyReturnValueHandler(List<HttpMessageConverter> messageConverters) {
        this.messageConverters = messageConverters;
    }

    /**
     * Supports methods or classes annotated with {@link ResponseBody}.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.supportsReturnType()} (line 139)
     * Real version:
     * <pre>
     *   return (AnnotatedElementUtils.hasAnnotation(containingClass, ResponseBody.class)
     *           || returnType.hasMethodAnnotation(ResponseBody.class));
     * </pre>
     *
     * <h3>ch10 Enhancement:</h3>
     * Class-level check now uses {@link MergedAnnotationUtils#hasAnnotation} to detect
     * @ResponseBody via meta-annotations. This means @RestController classes (which are
     * meta-annotated with @ResponseBody) automatically get JSON serialization for all
     * handler methods — no need for @ResponseBody on each method.
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        // Check method-level @ResponseBody (direct check is sufficient here —
        // composed annotations like @GetMapping don't carry @ResponseBody)
        if (handlerMethod.getMethod().isAnnotationPresent(ResponseBody.class)) {
            return true;
        }
        // Check class-level @ResponseBody, walking meta-annotations.
        // This is what makes @RestController work — it carries @ResponseBody
        // as a meta-annotation, so hasAnnotation() finds it.
        return MergedAnnotationUtils.hasAnnotation(
                handlerMethod.getBeanType(), ResponseBody.class);
    }

    /**
     * Serialize the return value using the first compatible message converter.
     *
     * Maps to: {@code RequestResponseBodyMethodProcessor.handleReturnValue()} (line 195)
     * → {@code AbstractMessageConverterMethodProcessor.writeWithMessageConverters()} (line 262)
     *
     * The real version:
     * 1. Sets mavContainer.setRequestHandled(true) — signals no view resolution needed
     * 2. Performs content negotiation to determine the media type
     * 3. Iterates converters and picks the first matching one
     * 4. Calls converter.write() with the negotiated media type
     *
     * We skip step 1 (no ModelAndViewContainer) and step 2 (no content negotiation),
     * but follow the same converter iteration pattern.
     */
    @Override
    public void handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                  HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue == null) {
            // Nothing to serialize — the real framework also returns early for null
            response.setStatus(HttpServletResponse.SC_OK);
            return;
        }

        Class<?> valueType = returnValue.getClass();
        for (HttpMessageConverter converter : messageConverters) {
            if (converter.canWrite(valueType)) {
                converter.write(returnValue, response);
                return;
            }
        }

        throw new IllegalStateException(
                "No suitable HttpMessageConverter found for return value of type ["
                        + valueType.getName() + "]");
    }
}
```

### Test Code

#### `[NEW] src/test/java/com/simplespringmvc/annotation/MergedAnnotationUtilsTest.java`

```java
package com.simplespringmvc.annotation;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

class MergedAnnotationUtilsTest {

    // Test annotations
    @Retention(RetentionPolicy.RUNTIME)
    @interface BaseAnnotation { String value() default ""; }

    @Retention(RetentionPolicy.RUNTIME)
    @BaseAnnotation("from-composed")
    @interface ComposedAnnotation { String value() default ""; }

    @Retention(RetentionPolicy.RUNTIME)
    @interface UnrelatedAnnotation {}

    // Test classes
    @BaseAnnotation static class DirectlyAnnotated {}
    @ComposedAnnotation static class MetaAnnotated {}
    @UnrelatedAnnotation static class UnrelatedOnly {}
    static class NoAnnotations {}

    static class MethodTargets {
        @BaseAnnotation("direct") public void directMethod() {}
        @ComposedAnnotation("/users") public void composedMethod() {}
        public void plainMethod() {}
    }

    @Nested @DisplayName("hasAnnotation()")
    class HasAnnotation {

        @Test @DisplayName("should find annotation when directly present on class")
        void shouldFindDirectAnnotation_WhenPresentOnClass() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    DirectlyAnnotated.class, BaseAnnotation.class)).isTrue();
        }

        @Test @DisplayName("should find annotation when present as meta-annotation on class")
        void shouldFindMetaAnnotation_WhenPresentOnComposedAnnotation() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    MetaAnnotated.class, BaseAnnotation.class)).isTrue();
        }

        @Test @DisplayName("should return false when annotation is absent")
        void shouldReturnFalse_WhenAnnotationAbsent() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    UnrelatedOnly.class, BaseAnnotation.class)).isFalse();
        }

        @Test @DisplayName("should return false when class has no annotations")
        void shouldReturnFalse_WhenNoAnnotations() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    NoAnnotations.class, BaseAnnotation.class)).isFalse();
        }

        @Test @DisplayName("should find annotation on method via meta-annotation")
        void shouldFindMetaAnnotation_WhenOnMethod() throws Exception {
            Method method = MethodTargets.class.getMethod("composedMethod");
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    method, BaseAnnotation.class)).isTrue();
        }

        @Test @DisplayName("should find annotation directly on method")
        void shouldFindDirectAnnotation_WhenOnMethod() throws Exception {
            Method method = MethodTargets.class.getMethod("directMethod");
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    method, BaseAnnotation.class)).isTrue();
        }

        @Test @DisplayName("should return false for plain method")
        void shouldReturnFalse_WhenPlainMethod() throws Exception {
            Method method = MethodTargets.class.getMethod("plainMethod");
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    method, BaseAnnotation.class)).isFalse();
        }
    }

    @Nested @DisplayName("findAnnotation()")
    class FindAnnotation {

        @Test @DisplayName("should return annotation when directly present")
        void shouldReturnAnnotation_WhenDirectlyPresent() {
            BaseAnnotation found = MergedAnnotationUtils.findAnnotation(
                    DirectlyAnnotated.class, BaseAnnotation.class);
            assertThat(found).isNotNull();
        }

        @Test @DisplayName("should return meta-annotation when present via composed annotation")
        void shouldReturnMetaAnnotation_WhenViaComposedAnnotation() {
            BaseAnnotation found = MergedAnnotationUtils.findAnnotation(
                    MetaAnnotated.class, BaseAnnotation.class);
            assertThat(found).isNotNull();
            assertThat(found.value()).isEqualTo("from-composed");
        }

        @Test @DisplayName("should return null when annotation is absent")
        void shouldReturnNull_WhenAnnotationAbsent() {
            assertThat(MergedAnnotationUtils.findAnnotation(
                    UnrelatedOnly.class, BaseAnnotation.class)).isNull();
        }

        @Test @DisplayName("should return null when class has no annotations")
        void shouldReturnNull_WhenNoAnnotations() {
            assertThat(MergedAnnotationUtils.findAnnotation(
                    NoAnnotations.class, BaseAnnotation.class)).isNull();
        }
    }

    @Nested @DisplayName("findComposedAnnotation()")
    class FindComposedAnnotation {

        @Test @DisplayName("should return the carrier annotation when meta-annotated")
        void shouldReturnCarrier_WhenMetaAnnotated() {
            java.lang.annotation.Annotation carrier = MergedAnnotationUtils.findComposedAnnotation(
                    MetaAnnotated.class, BaseAnnotation.class);
            assertThat(carrier).isNotNull();
            assertThat(carrier).isInstanceOf(ComposedAnnotation.class);
        }

        @Test @DisplayName("should return null when annotation is directly present (not composed)")
        void shouldReturnNull_WhenDirectlyPresent() {
            java.lang.annotation.Annotation carrier = MergedAnnotationUtils.findComposedAnnotation(
                    DirectlyAnnotated.class, BaseAnnotation.class);
            assertThat(carrier).isNull();
        }

        @Test @DisplayName("should return null when annotation is absent")
        void shouldReturnNull_WhenAnnotationAbsent() {
            java.lang.annotation.Annotation carrier = MergedAnnotationUtils.findComposedAnnotation(
                    UnrelatedOnly.class, BaseAnnotation.class);
            assertThat(carrier).isNull();
        }
    }

    @Nested @DisplayName("getStringAttribute()")
    class GetStringAttribute {

        @Test @DisplayName("should extract string attribute from annotation")
        void shouldExtractStringAttribute() throws Exception {
            Method method = MethodTargets.class.getMethod("composedMethod");
            ComposedAnnotation ann = method.getAnnotation(ComposedAnnotation.class);
            String value = MergedAnnotationUtils.getStringAttribute(ann, "value");
            assertThat(value).isEqualTo("/users");
        }

        @Test @DisplayName("should return empty string for missing attribute")
        void shouldReturnEmpty_WhenAttributeMissing() throws Exception {
            Method method = MethodTargets.class.getMethod("composedMethod");
            ComposedAnnotation ann = method.getAnnotation(ComposedAnnotation.class);
            String value = MergedAnnotationUtils.getStringAttribute(ann, "nonExistent");
            assertThat(value).isEmpty();
        }

        @Test @DisplayName("should return default value when attribute not set")
        void shouldReturnDefault_WhenAttributeNotSet() {
            ComposedAnnotation ann = MetaAnnotated.class.getAnnotation(ComposedAnnotation.class);
            String value = MergedAnnotationUtils.getStringAttribute(ann, "value");
            assertThat(value).isEmpty();
        }
    }

    @Nested @DisplayName("With real simple-spring-mvc annotations")
    class RealAnnotations {

        @Controller static class DirectController {}
        @RestController static class RestControllerClass {}
        static class PlainClass {}

        @Test @DisplayName("should detect @Controller directly")
        void shouldDetectControllerDirectly() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    DirectController.class, Controller.class)).isTrue();
        }

        @Test @DisplayName("should detect @Controller via @RestController meta-annotation")
        void shouldDetectControllerViaRestController() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    RestControllerClass.class, Controller.class)).isTrue();
        }

        @Test @DisplayName("should detect @ResponseBody via @RestController meta-annotation")
        void shouldDetectResponseBodyViaRestController() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    RestControllerClass.class, ResponseBody.class)).isTrue();
        }

        @Test @DisplayName("should not detect @Controller on plain class")
        void shouldNotDetectControllerOnPlainClass() {
            assertThat(MergedAnnotationUtils.hasAnnotation(
                    PlainClass.class, Controller.class)).isFalse();
        }

        @Test @DisplayName("should find @RequestMapping on method via @GetMapping")
        void shouldFindRequestMappingViaGetMapping() throws Exception {
            class TestController { @GetMapping("/users") public void getUsers() {} }
            Method method = TestController.class.getMethod("getUsers");
            RequestMapping found = MergedAnnotationUtils.findAnnotation(method, RequestMapping.class);
            assertThat(found).isNotNull();
            assertThat(found.method()).isEqualTo("GET");
        }

        @Test @DisplayName("should extract path from @GetMapping via getStringAttribute")
        void shouldExtractPathFromGetMapping() throws Exception {
            class TestController { @GetMapping("/users") public void getUsers() {} }
            Method method = TestController.class.getMethod("getUsers");
            GetMapping getMapping = method.getAnnotation(GetMapping.class);
            assertThat(MergedAnnotationUtils.getStringAttribute(getMapping, "value"))
                    .isEqualTo("/users");
        }
    }
}
```

#### `[NEW] src/test/java/com/simplespringmvc/integration/ComposedAnnotationsIntegrationTest.java`

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.adapter.ResponseBodyReturnValueHandler;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.annotation.*;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.RouteKey;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ComposedAnnotationsIntegrationTest {

    @Controller
    @RequestMapping(path = "/api")
    static class ComposedController {
        @GetMapping("/items")
        @ResponseBody
        public Map<String, Object> listItems() {
            return Map.of("count", 2, "items", java.util.List.of("a", "b"));
        }

        @PostMapping("/items")
        @ResponseBody
        public Map<String, String> createItem() {
            return Map.of("status", "created");
        }

        @PutMapping("/items/1")
        @ResponseBody
        public Map<String, String> updateItem() {
            return Map.of("status", "updated");
        }

        @DeleteMapping("/items/1")
        @ResponseBody
        public Map<String, String> deleteItem() {
            return Map.of("status", "deleted");
        }
    }

    @RestController
    @RequestMapping(path = "/api/v2")
    static class RestApiController {
        @GetMapping("/users")
        public Map<String, String> listUsers() {
            return Map.of("name", "Alice");
        }

        @PostMapping("/users")
        public Map<String, String> createUser() {
            return Map.of("status", "created");
        }
    }

    @Controller
    static class MixedController {
        @RequestMapping(path = "/old-style", method = "GET")
        @ResponseBody
        public String oldStyle() { return "old"; }

        @GetMapping("/new-style")
        @ResponseBody
        public String newStyle() { return "new"; }
    }

    @RestController
    static class RootController {
        @GetMapping
        public Map<String, String> root() {
            return Map.of("message", "root");
        }
    }

    private SimpleBeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;

    @BeforeEach
    void setUp() {
        beanContainer = new SimpleBeanContainer();
        handlerMapping = new SimpleHandlerMapping();
    }

    @Nested @DisplayName("Handler Mapping with composed annotations")
    class HandlerMappingDetection {

        @Test void shouldRegisterGetMapping() {
            beanContainer.registerBean(new ComposedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.lookupHandler("/api/items", "GET")).isNotNull();
            assertThat(handlerMapping.lookupHandler("/api/items", "GET")
                    .getMethod().getName()).isEqualTo("listItems");
        }

        @Test void shouldRegisterPostMapping() {
            beanContainer.registerBean(new ComposedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.lookupHandler("/api/items", "POST")).isNotNull();
            assertThat(handlerMapping.lookupHandler("/api/items", "POST")
                    .getMethod().getName()).isEqualTo("createItem");
        }

        @Test void shouldRegisterPutMapping() {
            beanContainer.registerBean(new ComposedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.lookupHandler("/api/items/1", "PUT")).isNotNull();
        }

        @Test void shouldRegisterDeleteMapping() {
            beanContainer.registerBean(new ComposedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.lookupHandler("/api/items/1", "DELETE")).isNotNull();
        }

        @Test void shouldNotMatchGetMapping_WhenWrongMethod() {
            beanContainer.registerBean(new ComposedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.lookupHandler("/api/items", "DELETE")).isNull();
        }

        @Test void shouldRegisterAllFourMethods() {
            beanContainer.registerBean(new ComposedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.getRegisteredMappings()).hasSize(4);
        }
    }

    @Nested @DisplayName("@RestController detection")
    class RestControllerDetection {

        @Test void shouldDetectRestControllerAsHandler() {
            beanContainer.registerBean(new RestApiController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.getRegisteredMappings()).hasSize(2);
        }

        @Test void shouldRegisterGetMapping_WithTypeLevelPath() {
            beanContainer.registerBean(new RestApiController());
            handlerMapping.init(beanContainer);
            HandlerMethod handler = handlerMapping.lookupHandler("/api/v2/users", "GET");
            assertThat(handler).isNotNull();
            assertThat(handler.getMethod().getName()).isEqualTo("listUsers");
        }
    }

    @Nested @DisplayName("@ResponseBody via @RestController")
    class ResponseBodyViaRestController {

        @Test void shouldSerializeAsJson_WhenRestController() throws Exception {
            beanContainer.registerBean(new RestApiController());
            handlerMapping.init(beanContainer);
            HandlerMethod handler = handlerMapping.lookupHandler("/api/v2/users", "GET");

            HttpServletRequest request = mock(HttpServletRequest.class);
            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter writer = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(writer));

            SimpleHandlerAdapter adapter = new SimpleHandlerAdapter();
            adapter.handle(request, response, handler);

            String json = writer.toString();
            assertThat(json).contains("\"name\"").contains("\"Alice\"");
            verify(response).setContentType("application/json");
        }

        @Test void shouldDetectResponseBody_ViaMetaAnnotation() throws Exception {
            RestApiController controller = new RestApiController();
            java.lang.reflect.Method method = controller.getClass().getMethod("listUsers");
            HandlerMethod handlerMethod = new HandlerMethod(controller, method);

            ResponseBodyReturnValueHandler handler = new ResponseBodyReturnValueHandler(
                    java.util.List.of(new com.simplespringmvc.converter.JacksonMessageConverter()));
            assertThat(handler.supportsReturnType(handlerMethod)).isTrue();
        }
    }

    @Nested @DisplayName("Mixed old/new style annotations")
    class MixedAnnotations {

        @Test void shouldSupportMixedAnnotations() {
            beanContainer.registerBean(new MixedController());
            handlerMapping.init(beanContainer);
            assertThat(handlerMapping.getRegisteredMappings()).hasSize(2);
            assertThat(handlerMapping.lookupHandler("/old-style", "GET")).isNotNull();
            assertThat(handlerMapping.lookupHandler("/new-style", "GET")).isNotNull();
        }
    }

    @Nested @DisplayName("Full DispatcherServlet pipeline")
    class FullPipeline {

        @Test void shouldDispatchGetMapping_ThroughFullPipeline() throws Exception {
            beanContainer.registerBean(new RestApiController());
            SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(beanContainer);
            servlet.init();

            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getRequestURI()).thenReturn("/api/v2/users");
            when(request.getMethod()).thenReturn("GET");

            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter writer = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(writer));

            servlet.service(request, response);

            assertThat(writer.toString()).contains("\"name\"").contains("\"Alice\"");
        }

        @Test void shouldDispatchPostMapping_ThroughFullPipeline() throws Exception {
            beanContainer.registerBean(new RestApiController());
            SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(beanContainer);
            servlet.init();

            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getRequestURI()).thenReturn("/api/v2/users");
            when(request.getMethod()).thenReturn("POST");

            HttpServletResponse response = mock(HttpServletResponse.class);
            StringWriter writer = new StringWriter();
            when(response.getWriter()).thenReturn(new PrintWriter(writer));

            servlet.service(request, response);

            assertThat(writer.toString()).contains("\"status\"").contains("\"created\"");
        }
    }

    @Nested @DisplayName("Edge cases")
    class EdgeCases {

        @Test void shouldHandleEmptyPath() {
            beanContainer.registerBean(new RootController());
            handlerMapping.init(beanContainer);
            HandlerMethod handler = handlerMapping.lookupHandler("/", "GET");
            assertThat(handler).isNotNull();
            assertThat(handler.getMethod().getName()).isEqualTo("root");
        }
    }
}
```

---

## Summary

| What | File | Type |
|---|---|---|
| Meta-annotation walker | `MergedAnnotationUtils.java` | New |
| `@GetMapping` | `GetMapping.java` | New |
| `@PostMapping` | `PostMapping.java` | New |
| `@PutMapping` | `PutMapping.java` | New |
| `@DeleteMapping` | `DeleteMapping.java` | New |
| `@RestController` | `RestController.java` | New |
| Handler detection + method scanning | `SimpleHandlerMapping.java` | Modified |
| Class-level `@ResponseBody` detection | `ResponseBodyReturnValueHandler.java` | Modified |

**Next chapter:** *Handler Interceptors* — add a pre/post processing pipeline around handler execution for cross-cutting concerns like logging, auth, and timing.
