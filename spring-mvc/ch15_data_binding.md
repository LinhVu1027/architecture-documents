# Chapter 15: Data Binding

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Handler methods can receive individual `@RequestParam` values and `@RequestBody` JSON objects | A form with 10 fields requires 10 separate `@RequestParam` parameters — verbose and error-prone | Build a `SimpleDataBinder` that automatically maps multiple request parameters onto a POJO's properties via `@ModelAttribute` |

---

## 15.1 The Integration Point

The data binding feature plugs into the **argument resolver chain** inside `SimpleHandlerAdapter`. When a handler method declares a `@ModelAttribute` parameter, a new `ModelAttributeArgumentResolver` needs to resolve it — creating a POJO, binding request parameters to its properties, and returning the populated object.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Register `ModelAttributeArgumentResolver` at the end of the argument resolver chain

```java
argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
// ch15: ModelAttributeArgumentResolver AFTER specific resolvers — they take priority
argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));
```

Two key decisions here:

1. **Why register LAST?** The real framework does the same thing — it creates two instances of `ModelAttributeMethodProcessor`: one early (annotation-required) and one as the very last resolver (catch-all for non-simple types). We place it after `@PathVariable`, `@RequestParam`, and `@RequestBody` resolvers so those specific annotations always win when present. A parameter like `@RequestParam String name` should never accidentally trigger data binding.

2. **Why pass ConversionService, not the whole adapter?** The real framework passes a `WebDataBinderFactory` that creates binders with the full adapter context. We pass just the `ConversionService` because that's the only piece our simplified `SimpleDataBinder` needs for type conversion. This keeps the dependency graph minimal.

This **connects the argument resolution pipeline to a new binding subsystem**. To make it work, we need to build:
- **`@ModelAttribute`** — the annotation that triggers data binding on a parameter
- **`SimpleDataBinder`** — creates a POJO instance and populates it from request parameters
- **`ModelAttributeArgumentResolver`** — the argument resolver that delegates to `SimpleDataBinder`
- **`DataBindingException`** — thrown when binding fails (no constructor, conversion error)

## 15.2 The @ModelAttribute Annotation

**New file:** `src/main/java/com/simplespringmvc/annotation/ModelAttribute.java`

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ModelAttribute {
}
```

The real `@ModelAttribute` supports `value`/`name` (explicit model attribute name) and `binding` (disable data binding). We strip it down to a marker annotation — its only job is to signal "bind request parameters to this POJO."

## 15.3 SimpleDataBinder — The Heart of Binding

**New file:** `src/main/java/com/simplespringmvc/binding/SimpleDataBinder.java`

This is where the real work happens. The binder:
1. Creates an instance of the target type via default constructor
2. Extracts all request parameters
3. For each parameter, finds a matching setter method or field on the target
4. Converts the String value to the property type via `ConversionService`
5. Invokes the setter (or sets the field directly)

```java
package com.simplespringmvc.binding;

import com.simplespringmvc.convert.ConversionService;
import jakarta.servlet.http.HttpServletRequest;

import java.lang.reflect.Method;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.Map;

public class SimpleDataBinder {

    private final ConversionService conversionService;

    public SimpleDataBinder(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    public <T> T bind(Class<T> targetType, HttpServletRequest request) {
        T target = createInstance(targetType);
        Map<String, String> parameterMap = extractParameters(request);
        bindParameters(target, parameterMap);
        return target;
    }

    public <T> T bind(T target, HttpServletRequest request) {
        Map<String, String> parameterMap = extractParameters(request);
        bindParameters(target, parameterMap);
        return target;
    }

    private <T> T createInstance(Class<T> targetType) {
        try {
            return targetType.getDeclaredConstructor().newInstance();
        } catch (NoSuchMethodException e) {
            throw new DataBindingException(
                    "No default constructor found on " + targetType.getName()
                            + ". Data binding requires a no-arg constructor.", e);
        } catch (Exception e) {
            throw new DataBindingException(
                    "Failed to instantiate " + targetType.getName(), e);
        }
    }

    private Map<String, String> extractParameters(HttpServletRequest request) {
        Map<String, String> params = new LinkedHashMap<>();
        Enumeration<String> names = request.getParameterNames();
        while (names.hasMoreElements()) {
            String name = names.nextElement();
            params.put(name, request.getParameter(name));
        }
        return params;
    }

    private void bindParameters(Object target, Map<String, String> parameters) {
        Class<?> targetClass = target.getClass();
        for (Map.Entry<String, String> entry : parameters.entrySet()) {
            String paramName = entry.getKey();
            String paramValue = entry.getValue();

            // Try setter method first (JavaBeans convention: setXxx)
            Method setter = findSetter(targetClass, paramName);
            if (setter != null) {
                setViaSetter(target, setter, paramValue);
                continue;
            }
            // Fall back to direct field access
            setViaField(target, targetClass, paramName, paramValue);
        }
    }

    private Method findSetter(Class<?> targetClass, String propertyName) {
        String setterName = "set" + capitalize(propertyName);
        for (Method method : targetClass.getMethods()) {
            if (method.getName().equals(setterName) && method.getParameterCount() == 1) {
                return method;
            }
        }
        return null;
    }

    private void setViaSetter(Object target, Method setter, String value) {
        Class<?> propertyType = setter.getParameterTypes()[0];
        Object converted = convertValue(value, propertyType);
        try {
            setter.invoke(target, converted);
        } catch (Exception e) {
            throw new DataBindingException(
                    "Failed to invoke " + setter.getName() + " on "
                            + target.getClass().getName(), e);
        }
    }

    private void setViaField(Object target, Class<?> targetClass,
                             String fieldName, String value) {
        try {
            java.lang.reflect.Field field = findField(targetClass, fieldName);
            if (field == null) {
                return; // Skip unknown parameters silently
            }
            field.setAccessible(true);
            Object converted = convertValue(value, field.getType());
            field.set(target, converted);
        } catch (DataBindingException e) {
            throw e;
        } catch (Exception e) {
            throw new DataBindingException(
                    "Failed to set field '" + fieldName + "' on "
                            + targetClass.getName(), e);
        }
    }

    private java.lang.reflect.Field findField(Class<?> clazz, String fieldName) {
        Class<?> current = clazz;
        while (current != null && current != Object.class) {
            try {
                return current.getDeclaredField(fieldName);
            } catch (NoSuchFieldException e) {
                current = current.getSuperclass();
            }
        }
        return null;
    }

    private Object convertValue(String value, Class<?> targetType) {
        if (targetType == String.class) {
            return value;
        }
        if (!conversionService.canConvert(String.class, targetType)) {
            throw new DataBindingException(
                    "No converter found for String → " + targetType.getName());
        }
        return conversionService.convert(value, targetType);
    }

    private String capitalize(String name) {
        if (name == null || name.isEmpty()) {
            return name;
        }
        return Character.toUpperCase(name.charAt(0)) + name.substring(1);
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/binding/DataBindingException.java`

```java
package com.simplespringmvc.binding;

public class DataBindingException extends RuntimeException {
    public DataBindingException(String message) { super(message); }
    public DataBindingException(String message, Throwable cause) { super(message, cause); }
}
```

### How the Real Framework Does It

Our `SimpleDataBinder` collapses a deep hierarchy from the real framework:

```
Real Framework:                          Simplified:
──────────────                           ───────────
DataBinder                               SimpleDataBinder
  └─ WebDataBinder                         (all-in-one)
       └─ ServletRequestDataBinder
            uses → BeanWrapperImpl
                     implements → BeanWrapper
                                    extends → PropertyAccessor
```

The real `DataBinder` (800+ lines) provides:
- **MutablePropertyValues** — a bag of name/value pairs extracted from the request
- **BeanWrapperImpl** — uses `java.beans.Introspector` to discover PropertyDescriptors
- **TypeConverterDelegate** — chains PropertyEditor + ConversionService for type conversion
- **BindingResult** — collects ALL errors instead of failing on the first one
- **AllowedFields / DisallowedFields** — security filtering to prevent mass assignment

We skip all of that and go straight: iterate parameters → find setter → convert → invoke.

## 15.4 ModelAttributeArgumentResolver

**New file:** `src/main/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.binding.SimpleDataBinder;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

public class ModelAttributeArgumentResolver implements HandlerMethodArgumentResolver {

    private final SimpleDataBinder dataBinder;

    public ModelAttributeArgumentResolver(ConversionService conversionService) {
        this.dataBinder = new SimpleDataBinder(conversionService);
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ModelAttribute.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
                                  HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();
        return dataBinder.bind(targetType, request);
    }
}
```

The resolver is intentionally thin — it checks for `@ModelAttribute` and delegates to `SimpleDataBinder`. In the real framework, `ModelAttributeMethodProcessor.resolveArgument()` (line 106) orchestrates a much richer flow: check the model first, create or reuse the attribute, bind, validate, collect errors.

## 15.5 Try It Yourself

<details>
<summary>Challenge 1: What happens when @RequestParam and @ModelAttribute both match?</summary>

Consider this handler method:
```java
@PostMapping("/users")
public String create(@RequestParam String name, @ModelAttribute UserForm form) { ... }
```

With a POST to `/users?name=Alice&age=30`:
- `@RequestParam String name` → resolved by `RequestParamArgumentResolver` → `"Alice"`
- `@ModelAttribute UserForm form` → resolved by `ModelAttributeArgumentResolver` → binds ALL parameters including `name="Alice"` and `age=30`

Both resolvers see the same request parameters, but they serve different purposes. `@RequestParam` extracts a single value; `@ModelAttribute` binds all matching parameters to a POJO. The form's `name` property WILL be set to "Alice" even though there's also a `@RequestParam` for it — they don't conflict because they resolve to different method arguments.

</details>

<details>
<summary>Challenge 2: Why does SimpleDataBinder skip unknown parameters silently?</summary>

When a request has a parameter like `csrfToken=abc123` but the target POJO has no `csrfToken` property, the binder silently ignores it. Why?

Because HTTP requests often carry extra parameters that aren't meant for the form object:
- CSRF tokens from security frameworks
- Submit button names (`<input type="submit" name="save">`)
- Hidden fields for JavaScript state
- Parameters meant for other resolvers (`@RequestParam`)

The real framework does the same — unmatched parameters are simply ignored during binding. However, the real framework adds **security filtering** via `setAllowedFields()` / `setDisallowedFields()` to prevent attackers from binding sensitive properties they shouldn't control (the "mass assignment" vulnerability). Our simplified version doesn't have this protection.

</details>

## 15.6 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/binding/SimpleDataBinderTest.java`

```java
@Test
void shouldBindStringAndIntProperties_WhenSettersExist() {
    setupParameters("name", "Alice", "age", "30", "email", "alice@test.com");
    UserForm result = dataBinder.bind(UserForm.class, request);
    assertThat(result.getName()).isEqualTo("Alice");
    assertThat(result.getAge()).isEqualTo(30);
    assertThat(result.getEmail()).isEqualTo("alice@test.com");
}

@Test
void shouldBindViaDirectFieldAccess_WhenNoSetterExists() { ... }

@Test
void shouldConvertBooleanAndDouble_WhenTypesMatch() { ... }

@Test
void shouldSkipUnknownParameters_WhenNoMatchingProperty() { ... }

@Test
void shouldThrowException_WhenNoDefaultConstructor() { ... }
```

**New file:** `src/test/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolverTest.java`

```java
@Test
void shouldSupportParameter_WhenAnnotatedWithModelAttribute() { ... }

@Test
void shouldNotSupportParameter_WhenAnnotatedWithRequestParam() { ... }

@Test
void shouldResolveAndBindPojo_WhenRequestHasMatchingParams() { ... }
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/DataBindingIntegrationTest.java`

```java
@Test
void shouldBindFormFromRequestParams_WhenPostWithModelAttribute() {
    HttpServletRequest request = mockRequest("POST", "/users",
            "username", "Alice", "age", "28", "active", "true");
    HandlerMethod handler = handlerMapping.lookupHandler(request);
    ModelAndView mav = handlerAdapter.handle(request, response, handler);
    assertThat(responseBody.toString()).contains("Created: Alice, age=28, active=true");
}

@Test
void shouldMixModelAttributeWithRequestParam_WhenBothPresent() {
    HttpServletRequest request = mockRequest("GET", "/users/search",
            "query", "admin", "age", "25");
    HandlerMethod handler = handlerMapping.lookupHandler(request);
    handlerAdapter.handle(request, response, handler);
    assertThat(responseBody.toString()).contains("Search: admin, filterAge=25");
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 15.7 Why This Works

> ★ **Insight** -------------------------------------------
> **The JavaBeans naming convention is what makes data binding possible.** The `setXxx()` naming pattern isn't just a style preference — it's a contract that tools can depend on. Java's `java.beans.Introspector` discovers properties by scanning for getter/setter pairs matching this convention. Spring's `BeanWrapperImpl` builds on top of this, and our `SimpleDataBinder` uses the same principle (though we skip `Introspector` and find setters by string matching). This convention-over-configuration approach means a POJO with `setName(String)` and `setAge(int)` automatically becomes bindable from request parameters `name=X&age=Y` — no mapping configuration needed.
>
> **Trade-off:** This only works for flat objects. Nested properties like `address.city=NYC` require path traversal (the real `BeanWrapperImpl` handles this via `AbstractNestablePropertyAccessor`). Our simplified binder doesn't support this.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Resolver ordering is the framework's priority system.** The resolver chain in `SimpleHandlerAdapter` isn't just a list — it's an ordered priority queue. When multiple resolvers could handle a parameter, the FIRST match wins. This is why `@RequestParam` and `@PathVariable` resolvers are registered before `@ModelAttribute`: a parameter with `@RequestParam String name` should always be resolved as a query parameter, never accidentally bound as a model attribute. The real framework creates 26+ resolvers in a carefully ordered list inside `RequestMappingHandlerAdapter.getDefaultArgumentResolvers()`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Fail-fast vs error-collection is a core design choice.** Our `SimpleDataBinder` throws on the first error. The real framework collects ALL binding errors into a `BindingResult` object, which can be injected as a handler method parameter (`@ModelAttribute UserForm form, BindingResult errors`). This allows the handler to inspect all errors at once and return a meaningful validation response. The error-collection pattern is essential for form validation UX — you want to show ALL field errors, not just the first one.
> -----------------------------------------------------------

## 15.8 What We Enhanced

| Aspect | Before (ch14) | Current (ch15) | Real Framework |
|--------|---------------|----------------|----------------|
| **POJO binding** | Not possible — only `@RequestParam`, `@PathVariable`, `@RequestBody` | `@ModelAttribute` binds all matching request params to a POJO's properties | `ModelAttributeMethodProcessor` with `BeanWrapperImpl` via `java.beans.Introspector` — supports nested/indexed properties, security filtering, and error collection |
| **Object creation** | Manual by developer | `SimpleDataBinder` creates via default constructor | `BeanUtils.instantiateClass()` + constructor binding (data constructors, records, Kotlin) |
| **Property access** | N/A | Setter methods + field fallback | `BeanWrapperImpl` via PropertyDescriptor with nested property path traversal |
| **Error handling** | N/A | Fail-fast exception on first error | `BindingResult` collects all errors for batch reporting |
| **Resolver chain** | 3 resolvers: PathVariable, RequestParam, RequestBody | 4 resolvers: + ModelAttribute at end | 26+ resolvers in carefully ordered list |

## 15.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@ModelAttribute` (marker only) | `@ModelAttribute` | `ModelAttribute.java:72` | Real has `value`/`name` for model key, `binding` flag to disable binding |
| `SimpleDataBinder.bind()` | `ServletRequestDataBinder.bind()` | `ServletRequestDataBinder.java:153` | Real extracts into `MutablePropertyValues`, handles multipart, delegates to `BeanWrapperImpl` |
| `SimpleDataBinder.findSetter()` | `BeanWrapperImpl` via `Introspector` | `BeanWrapperImpl.java` (extends `AbstractNestablePropertyAccessor`) | Real uses `PropertyDescriptor` from Java introspection, supports nested paths like `address.city` |
| `SimpleDataBinder.convertValue()` | `TypeConverterDelegate.convertIfNecessary()` | `TypeConverterDelegate.java:149` | Real chains PropertyEditor + ConversionService, handles generics |
| `ModelAttributeArgumentResolver` | `ServletModelAttributeMethodProcessor` | `ServletModelAttributeMethodProcessor.java:52` | Real checks model first, supports validation, creates `BindingResult` |
| Resolver ordering in `SimpleHandlerAdapter` | `RequestMappingHandlerAdapter.getDefaultArgumentResolvers()` | `RequestMappingHandlerAdapter.java:644` | Real registers 26+ resolvers including two `ModelAttributeMethodProcessor` instances |

## 15.10 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/ModelAttribute.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation that binds a method parameter to a named model attribute,
 * populated via data binding from request parameters.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.ModelAttribute}
 *
 * When placed on a handler method parameter, the framework will:
 * <ol>
 *   <li>Create an instance of the parameter type using its default constructor</li>
 *   <li>Iterate all request parameters and bind matching ones to the object's properties</li>
 *   <li>Convert String parameter values to the property's declared type via ConversionService</li>
 *   <li>Pass the populated object as the method argument</li>
 * </ol>
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}PostMapping("/users")
 * public String createUser({@literal @}ModelAttribute UserForm form) {
 *     // form.getName(), form.getAge() etc. are populated from request params
 * }
 * </pre>
 *
 * The real annotation also supports:
 * <ul>
 *   <li>Method-level placement (to add attributes to the model before every handler invocation)</li>
 *   <li>{@code name} / {@code value} attributes for explicit model attribute naming</li>
 *   <li>{@code binding} attribute to disable data binding</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Parameter-level only — no method-level model population</li>
 *   <li>No {@code name}/{@code value} attribute — attribute name is always derived from type</li>
 *   <li>No {@code binding} flag — binding is always enabled</li>
 * </ul>
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ModelAttribute {
}
```

#### File: `src/main/java/com/simplespringmvc/binding/DataBindingException.java` [NEW]

```java
package com.simplespringmvc.binding;

/**
 * Exception thrown when data binding fails — either due to instantiation errors,
 * type conversion failures, or reflection errors when setting properties.
 *
 * Maps to: Various exceptions in the real framework:
 * <ul>
 *   <li>{@code org.springframework.beans.TypeMismatchException} — conversion failure</li>
 *   <li>{@code org.springframework.beans.BeanInstantiationException} — constructor failure</li>
 *   <li>{@code org.springframework.beans.NotWritablePropertyException} — no setter/field</li>
 * </ul>
 *
 * The real framework collects errors into a {@code BindingResult} rather than
 * throwing immediately, allowing all binding errors to be reported at once.
 * We simplify to fail-fast with a single exception.
 */
public class DataBindingException extends RuntimeException {

    public DataBindingException(String message) {
        super(message);
    }

    public DataBindingException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/binding/SimpleDataBinder.java` [NEW]

```java
package com.simplespringmvc.binding;

import com.simplespringmvc.convert.ConversionService;
import jakarta.servlet.http.HttpServletRequest;

import java.lang.reflect.Method;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Binds HTTP request parameters to a target POJO by matching parameter names
 * to JavaBeans setter methods, using ConversionService for type conversion.
 *
 * Maps to: {@code org.springframework.web.bind.ServletRequestDataBinder}
 * which extends {@code WebDataBinder} → {@code DataBinder}.
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   ServletRequestDataBinder.bind(ServletRequest request):
 *     1. Extract request params → MutablePropertyValues
 *     2. Call doBind(mpvs) → checkFieldDefaults → checkFieldMarkers → super.doBind()
 *     3. DataBinder.doBind() → applyPropertyValues() → getPropertyAccessor().setPropertyValues()
 *     4. The PropertyAccessor (BeanWrapperImpl) uses Java introspection to find
 *        PropertyDescriptors (getter/setter pairs) for each property
 *     5. For each PropertyValue, it calls the setter method via reflection
 *     6. Type conversion happens via TypeConverterDelegate → ConversionService
 * </pre>
 *
 * <h3>The BeanWrapper role:</h3>
 * The real framework uses {@code BeanWrapperImpl} (implements {@code BeanWrapper}) as the
 * property accessor. BeanWrapper wraps a Java object and provides property access via
 * {@code java.beans.Introspector} — it discovers PropertyDescriptors (getter/setter pairs)
 * and uses them to read/write properties. It also supports nested properties
 * (e.g., {@code address.city}) and indexed properties (e.g., {@code items[0]}).
 *
 * We collapse BeanWrapper's functionality directly into SimpleDataBinder:
 * <ul>
 *   <li>Discover setter methods by naming convention ({@code setXxx})</li>
 *   <li>Invoke setters via reflection</li>
 *   <li>Fall back to direct field access if no setter exists</li>
 * </ul>
 *
 * <h3>Property matching algorithm:</h3>
 * For a request parameter named {@code "age"}:
 * <ol>
 *   <li>Look for a method named {@code setAge(T)} on the target class</li>
 *   <li>If found, convert the String parameter value to type T via ConversionService</li>
 *   <li>If no setter, look for a field named {@code age} and set it directly</li>
 *   <li>If neither found, skip the parameter silently (it might be for something else)</li>
 * </ol>
 *
 * Simplifications:
 * <ul>
 *   <li>No nested property support ({@code address.city} won't work)</li>
 *   <li>No indexed property support ({@code items[0]} won't work)</li>
 *   <li>No field markers or field defaults (for HTML checkbox handling)</li>
 *   <li>No allowed/disallowed field filtering (security)</li>
 *   <li>No BindingResult error collection — throws on conversion failure</li>
 *   <li>Single class instead of DataBinder → BeanWrapper → PropertyAccessor hierarchy</li>
 * </ul>
 */
public class SimpleDataBinder {

    private final ConversionService conversionService;

    public SimpleDataBinder(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    /**
     * Create an instance of the target type and bind request parameters to it.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.resolveArgument()} (line 106) which calls
     * {@code createAttribute()} → {@code new BeanWrapperImpl(targetType).getWrappedInstance()}
     * then {@code binder.bind(servletRequest)}.
     *
     * @param targetType the POJO class to instantiate and populate
     * @param request    the HTTP request containing parameters to bind
     * @param <T>        the target type
     * @return a new instance of targetType with properties set from request parameters
     */
    public <T> T bind(Class<T> targetType, HttpServletRequest request) {
        T target = createInstance(targetType);
        Map<String, String> parameterMap = extractParameters(request);
        bindParameters(target, parameterMap);
        return target;
    }

    /**
     * Bind parameters to an existing target instance.
     *
     * Maps to: {@code ServletRequestDataBinder.bind(ServletRequest)}
     *
     * @param target  the object to populate
     * @param request the HTTP request
     * @param <T>     the target type
     * @return the same target instance, with properties set
     */
    public <T> T bind(T target, HttpServletRequest request) {
        Map<String, String> parameterMap = extractParameters(request);
        bindParameters(target, parameterMap);
        return target;
    }

    /**
     * Create an instance of the target type using its default (no-arg) constructor.
     *
     * Maps to: {@code BeanUtils.instantiateClass(targetType)} (used by ModelAttributeMethodProcessor)
     *
     * The real framework also supports:
     * <ul>
     *   <li>Constructor binding (since 6.1) — using a primary or single data constructor</li>
     *   <li>Kotlin data classes</li>
     *   <li>Records (Java 16+)</li>
     * </ul>
     * We only support the default constructor path.
     */
    private <T> T createInstance(Class<T> targetType) {
        try {
            return targetType.getDeclaredConstructor().newInstance();
        } catch (NoSuchMethodException e) {
            throw new DataBindingException(
                    "No default constructor found on " + targetType.getName()
                            + ". Data binding requires a no-arg constructor.", e);
        } catch (Exception e) {
            throw new DataBindingException(
                    "Failed to instantiate " + targetType.getName(), e);
        }
    }

    /**
     * Extract request parameters into a simple map.
     *
     * Maps to: {@code new ServletRequestParameterPropertyValues(request)} (line 157 in
     * ServletRequestDataBinder) which creates MutablePropertyValues from the request's
     * parameter map. The real version also handles multipart files and field markers.
     */
    private Map<String, String> extractParameters(HttpServletRequest request) {
        Map<String, String> params = new LinkedHashMap<>();
        Enumeration<String> names = request.getParameterNames();
        while (names.hasMoreElements()) {
            String name = names.nextElement();
            params.put(name, request.getParameter(name));
        }
        return params;
    }

    /**
     * Match request parameters to setter methods (or fields) on the target object,
     * convert values, and set them.
     *
     * Maps to: {@code BeanWrapperImpl.setPropertyValue(PropertyValue)} which uses
     * {@code PropertyDescriptor.getWriteMethod().invoke()} (line 362 in
     * AbstractNestablePropertyAccessor) combined with type conversion via
     * {@code TypeConverterDelegate.convertIfNecessary()}.
     *
     * The real flow:
     * <pre>
     *   DataBinder.doBind(mpvs)
     *     → applyPropertyValues(mpvs)
     *       → getPropertyAccessor().setPropertyValues(mpvs)     // BeanWrapperImpl
     *         → for each PropertyValue:
     *           → setPropertyValue(pv)
     *             → find PropertyDescriptor via Introspector
     *             → convert value via TypeConverterDelegate
     *             → invoke setter method via reflection
     * </pre>
     *
     * We simplify: iterate parameters → find setter by naming convention → convert → invoke.
     */
    private void bindParameters(Object target, Map<String, String> parameters) {
        Class<?> targetClass = target.getClass();

        for (Map.Entry<String, String> entry : parameters.entrySet()) {
            String paramName = entry.getKey();
            String paramValue = entry.getValue();

            // Try setter method first (JavaBeans convention: setXxx)
            Method setter = findSetter(targetClass, paramName);
            if (setter != null) {
                setViaSetter(target, setter, paramValue);
                continue;
            }

            // Fall back to direct field access
            setViaField(target, targetClass, paramName, paramValue);
        }
    }

    /**
     * Find a setter method matching the JavaBeans naming convention.
     *
     * Maps to: {@code java.beans.Introspector.getBeanInfo(targetClass).getPropertyDescriptors()}
     * which returns PropertyDescriptor[] — each has a getWriteMethod() for the setter.
     *
     * We use a simpler approach: capitalize the first letter of the property name
     * and look for "set" + CapitalizedName.
     *
     * Example: property "firstName" → method "setFirstName"
     */
    private Method findSetter(Class<?> targetClass, String propertyName) {
        String setterName = "set" + capitalize(propertyName);
        for (Method method : targetClass.getMethods()) {
            if (method.getName().equals(setterName) && method.getParameterCount() == 1) {
                return method;
            }
        }
        return null;
    }

    /**
     * Set a property value via its setter method, converting the String value
     * to the setter's parameter type.
     */
    private void setViaSetter(Object target, Method setter, String value) {
        Class<?> propertyType = setter.getParameterTypes()[0];
        Object converted = convertValue(value, propertyType);
        try {
            setter.invoke(target, converted);
        } catch (Exception e) {
            throw new DataBindingException(
                    "Failed to invoke " + setter.getName() + " on "
                            + target.getClass().getName(), e);
        }
    }

    /**
     * Set a property value via direct field access when no setter exists.
     *
     * Maps to: {@code DirectFieldAccessor} in the real framework, which is an
     * alternative to BeanWrapper. Spring's DataBinder can be configured to use
     * either field access or property access (setters). We try both.
     */
    private void setViaField(Object target, Class<?> targetClass, String fieldName, String value) {
        try {
            java.lang.reflect.Field field = findField(targetClass, fieldName);
            if (field == null) {
                // No setter and no field — skip silently.
                // The request parameter might be for something else entirely
                // (e.g., a CSRF token, a submit button name).
                return;
            }
            field.setAccessible(true);
            Object converted = convertValue(value, field.getType());
            field.set(target, converted);
        } catch (DataBindingException e) {
            throw e;
        } catch (Exception e) {
            throw new DataBindingException(
                    "Failed to set field '" + fieldName + "' on "
                            + targetClass.getName(), e);
        }
    }

    /**
     * Find a field by name, searching up the class hierarchy.
     */
    private java.lang.reflect.Field findField(Class<?> clazz, String fieldName) {
        Class<?> current = clazz;
        while (current != null && current != Object.class) {
            try {
                return current.getDeclaredField(fieldName);
            } catch (NoSuchFieldException e) {
                current = current.getSuperclass();
            }
        }
        return null;
    }

    /**
     * Convert a String value to the target type using the ConversionService.
     *
     * Maps to: {@code TypeConverterDelegate.convertIfNecessary()} which delegates
     * to {@code ConversionService.convert()} for types it handles.
     */
    private Object convertValue(String value, Class<?> targetType) {
        if (targetType == String.class) {
            return value;
        }
        if (!conversionService.canConvert(String.class, targetType)) {
            throw new DataBindingException(
                    "No converter found for String → " + targetType.getName());
        }
        return conversionService.convert(value, targetType);
    }

    /**
     * Capitalize the first character of a string (JavaBeans convention).
     * "name" → "Name", "firstName" → "FirstName"
     */
    private String capitalize(String name) {
        if (name == null || name.isEmpty()) {
            return name;
        }
        return Character.toUpperCase(name.charAt(0)) + name.substring(1);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.binding.SimpleDataBinder;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves method arguments annotated with {@link ModelAttribute} by creating
 * a new instance of the parameter type and binding request parameters to it.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor}
 * which extends {@code org.springframework.web.method.annotation.ModelAttributeMethodProcessor}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @ModelAttribute}</li>
 *   <li>{@code resolveArgument()} delegates to {@link SimpleDataBinder} to create
 *       and populate the target POJO from request parameters</li>
 * </ol>
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   ModelAttributeMethodProcessor.resolveArgument():
 *     1. Determine attribute name from @ModelAttribute or parameter type
 *     2. Check if attribute exists in the model (ModelAndViewContainer)
 *     3. If not, create via createAttribute() or DataBinder.construct()
 *     4. Create WebDataBinder for the attribute
 *     5. Call bindRequestParameters(binder, request)
 *        → ServletRequestDataBinder.bind(servletRequest)
 *        → Extracts params into MutablePropertyValues
 *        → BeanWrapperImpl sets properties via setters
 *     6. validateIfApplicable() — runs @Valid if present
 *     7. Check BindingResult for errors
 *     8. Add attribute + BindingResult to model
 * </pre>
 *
 * <h3>The "annotationNotRequired" mode:</h3>
 * The real framework creates TWO instances of ModelAttributeMethodProcessor:
 * <ul>
 *   <li>One with {@code annotationNotRequired=false} — only handles @ModelAttribute</li>
 *   <li>One with {@code annotationNotRequired=true} — handles any non-simple type as a
 *       catch-all at the END of the resolver chain</li>
 * </ul>
 * This means in the real framework, a method parameter like {@code UserForm form}
 * (without @ModelAttribute) is STILL bound from request parameters. We require
 * the explicit annotation for clarity.
 *
 * Simplifications:
 * <ul>
 *   <li>Requires explicit {@code @ModelAttribute} — no "default resolution" mode</li>
 *   <li>No model interaction (no ModelAndViewContainer check)</li>
 *   <li>No validation (@Valid support is Feature 16)</li>
 *   <li>No BindingResult parameter support</li>
 *   <li>No attribute name resolution — always creates a new instance</li>
 * </ul>
 */
public class ModelAttributeArgumentResolver implements HandlerMethodArgumentResolver {

    private final SimpleDataBinder dataBinder;

    public ModelAttributeArgumentResolver(ConversionService conversionService) {
        this.dataBinder = new SimpleDataBinder(conversionService);
    }

    /**
     * Supports parameters annotated with {@code @ModelAttribute}.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.supportsParameter()} (line 91)
     * The real version: {@code parameter.hasParameterAnnotation(ModelAttribute.class)
     *   || (annotationNotRequired && !BeanUtils.isSimpleProperty(parameterType))}
     *
     * We only support the explicit annotation case.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ModelAttribute.class);
    }

    /**
     * Create an instance of the parameter type and bind request parameters to it.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.resolveArgument()} (line 106)
     *
     * The real flow:
     * <pre>
     *   1. name = ModelFactory.getNameForParameter(parameter)
     *   2. if (mavContainer.containsAttribute(name)) → get from model
     *   3. else → createAttribute() or DataBinder.construct()
     *   4. binder = binderFactory.createBinder(request, attribute, name)
     *   5. binder.bind(servletRequest)     // property binding
     *   6. validateIfApplicable(binder)    // @Valid
     *   7. check bindingResult.hasErrors() → throw MethodArgumentNotValidException
     *   8. add attribute + bindingResult to model
     * </pre>
     *
     * We collapse this to: create → bind → return.
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();
        return dataBinder.bind(targetType, request);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.convert.SimpleConversionService;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.converter.PlainTextMessageConverter;
import com.simplespringmvc.http.ContentNegotiationManager;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;

/**
 * The adapter that bridges the DispatcherServlet's generic handler invocation
 * to our specific HandlerMethod-based invocation pipeline.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 *
 * <h3>ch15 Enhancement:</h3>
 * <ul>
 *   <li>Registers a {@link ModelAttributeArgumentResolver} to support {@code @ModelAttribute}
 *       data binding — request parameters are automatically bound to POJO properties</li>
 *   <li>ModelAttributeArgumentResolver is registered AFTER specific annotation resolvers
 *       (PathVariable, RequestParam, RequestBody) so those take priority when annotations
 *       are present, matching the real framework's resolver ordering</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Single adapter instance (not a chain of adapters)</li>
 *   <li>No ModelAndViewContainer — return value handlers return ModelAndView directly</li>
 *   <li>No WebDataBinderFactory</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();
    private final List<HttpMessageConverter> messageConverters = new ArrayList<>();
    private final ConversionService conversionService;
    private final ContentNegotiationManager contentNegotiationManager;

    public SimpleHandlerAdapter() {
        // ch14: Create the ContentNegotiationManager for Accept header parsing
        contentNegotiationManager = new ContentNegotiationManager();

        // ch14: Register both JSON and plain-text converters.
        // ORDER MATTERS: JacksonMessageConverter first means JSON is the default
        // when Accept is */* (most common case). PlainTextMessageConverter handles
        // text/plain requests for String return values.
        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());

        conversionService = new SimpleConversionService();

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers — they take priority
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

        // ch14: ResponseBodyReturnValueHandler now takes a ContentNegotiationManager
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                messageConverters, contentNegotiationManager));
        returnValueHandlers.addHandler(new ModelAndViewReturnValueHandler());
        returnValueHandlers.addHandler(new ViewNameReturnValueHandler());
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    public SimpleHandlerAdapter(List<HandlerMethodArgumentResolver> customResolvers) {
        contentNegotiationManager = new ContentNegotiationManager();

        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());

        conversionService = new SimpleConversionService();

        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                messageConverters, contentNegotiationManager));
        returnValueHandlers.addHandler(new ModelAndViewReturnValueHandler());
        returnValueHandlers.addHandler(new ViewNameReturnValueHandler());
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Object[] args = resolveArguments(handlerMethod, request);
        Object result = invokeHandlerMethod(handlerMethod, args);
        return returnValueHandlers.handleReturnValue(result, handlerMethod, request, response);
    }

    private Object[] resolveArguments(HandlerMethod handlerMethod, HttpServletRequest request) throws Exception {
        MethodParameter[] parameters = handlerMethod.getMethodParameters();
        if (parameters.length == 0) {
            return new Object[0];
        }
        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter param = parameters[i];
            if (argumentResolvers.supportsParameter(param)) {
                args[i] = argumentResolvers.resolveArgument(param, request);
            } else {
                throw new IllegalStateException(
                        "No suitable resolver for argument " + i + " of type '"
                                + param.getParameterType().getName()
                                + "' on method " + handlerMethod);
            }
        }
        return args;
    }

    private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    public HandlerMethodArgumentResolverComposite getArgumentResolvers() { return argumentResolvers; }
    public HandlerMethodReturnValueHandlerComposite getReturnValueHandlers() { return returnValueHandlers; }
    public List<HttpMessageConverter> getMessageConverters() { return messageConverters; }
    public ConversionService getConversionService() { return conversionService; }
    public ContentNegotiationManager getContentNegotiationManager() { return contentNegotiationManager; }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/binding/SimpleDataBinderTest.java` [NEW]

```java
package com.simplespringmvc.binding;

import com.simplespringmvc.convert.SimpleConversionService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import jakarta.servlet.http.HttpServletRequest;
import java.util.Collections;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link SimpleDataBinder}.
 *
 * These tests verify the core data binding behavior: creating instances,
 * matching request parameters to setters, type conversion, and field fallback.
 */
class SimpleDataBinderTest {

    private SimpleDataBinder dataBinder;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        dataBinder = new SimpleDataBinder(new SimpleConversionService());
        request = mock(HttpServletRequest.class);
    }

    // ─── Target POJOs for testing ────────────────────────────────

    public static class UserForm {
        private String name;
        private int age;
        private String email;

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    public static class FieldOnlyPojo {
        public String title;
        public int count;
    }

    public static class MixedPojo {
        private String name;
        public int score;

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        // score has no setter — direct field access
    }

    public static class BooleanForm {
        private boolean active;
        private double price;

        public boolean isActive() { return active; }
        public void setActive(boolean active) { this.active = active; }
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
    }

    public static class NoDefaultConstructor {
        private final String value;
        public NoDefaultConstructor(String value) { this.value = value; }
    }

    // ─── Tests ───────────────────────────────────────────────────

    @Test
    void shouldBindStringAndIntProperties_WhenSettersExist() {
        setupParameters("name", "Alice", "age", "30", "email", "alice@test.com");

        UserForm result = dataBinder.bind(UserForm.class, request);

        assertThat(result.getName()).isEqualTo("Alice");
        assertThat(result.getAge()).isEqualTo(30);
        assertThat(result.getEmail()).isEqualTo("alice@test.com");
    }

    @Test
    void shouldBindViaDirectFieldAccess_WhenNoSetterExists() {
        setupParameters("title", "Hello", "count", "5");

        FieldOnlyPojo result = dataBinder.bind(FieldOnlyPojo.class, request);

        assertThat(result.title).isEqualTo("Hello");
        assertThat(result.count).isEqualTo(5);
    }

    @Test
    void shouldPreferSetterOverField_WhenBothExist() {
        setupParameters("name", "Bob", "score", "99");

        MixedPojo result = dataBinder.bind(MixedPojo.class, request);

        assertThat(result.getName()).isEqualTo("Bob");
        assertThat(result.score).isEqualTo(99);
    }

    @Test
    void shouldConvertBooleanAndDouble_WhenTypesMatch() {
        setupParameters("active", "true", "price", "19.99");

        BooleanForm result = dataBinder.bind(BooleanForm.class, request);

        assertThat(result.isActive()).isTrue();
        assertThat(result.getPrice()).isEqualTo(19.99);
    }

    @Test
    void shouldSkipUnknownParameters_WhenNoMatchingProperty() {
        // "csrfToken" has no matching property — should be silently skipped
        setupParameters("name", "Alice", "csrfToken", "abc123");

        UserForm result = dataBinder.bind(UserForm.class, request);

        assertThat(result.getName()).isEqualTo("Alice");
        // No error — unknown parameters are simply ignored
    }

    @Test
    void shouldReturnEmptyObject_WhenNoParametersProvided() {
        when(request.getParameterNames()).thenReturn(Collections.emptyEnumeration());

        UserForm result = dataBinder.bind(UserForm.class, request);

        assertThat(result).isNotNull();
        assertThat(result.getName()).isNull();
        assertThat(result.getAge()).isEqualTo(0); // primitive default
    }

    @Test
    void shouldThrowException_WhenNoDefaultConstructor() {
        when(request.getParameterNames()).thenReturn(Collections.emptyEnumeration());

        assertThatThrownBy(() -> dataBinder.bind(NoDefaultConstructor.class, request))
                .isInstanceOf(DataBindingException.class)
                .hasMessageContaining("No default constructor");
    }

    @Test
    void shouldBindToExistingInstance_WhenTargetProvided() {
        UserForm existing = new UserForm();
        existing.setName("Old");
        setupParameters("name", "New", "age", "25");

        UserForm result = dataBinder.bind(existing, request);

        assertThat(result).isSameAs(existing);
        assertThat(result.getName()).isEqualTo("New");
        assertThat(result.getAge()).isEqualTo(25);
    }

    @Test
    void shouldHandlePartialBinding_WhenOnlySomeParametersProvided() {
        setupParameters("name", "Charlie");

        UserForm result = dataBinder.bind(UserForm.class, request);

        assertThat(result.getName()).isEqualTo("Charlie");
        assertThat(result.getAge()).isEqualTo(0); // unset primitive
        assertThat(result.getEmail()).isNull();   // unset reference
    }

    // ─── Helpers ─────────────────────────────────────────────────

    private void setupParameters(String... keyValues) {
        java.util.Map<String, String> params = new java.util.LinkedHashMap<>();
        for (int i = 0; i < keyValues.length; i += 2) {
            params.put(keyValues[i], keyValues[i + 1]);
        }
        when(request.getParameterNames()).thenReturn(Collections.enumeration(params.keySet()));
        for (var entry : params.entrySet()) {
            when(request.getParameter(entry.getKey())).thenReturn(entry.getValue());
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.convert.SimpleConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;
import java.util.Collections;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link ModelAttributeArgumentResolver}.
 *
 * Tests the resolver's parameter support check and argument resolution —
 * verifying that @ModelAttribute parameters get bound from request parameters.
 */
class ModelAttributeArgumentResolverTest {

    private ModelAttributeArgumentResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        resolver = new ModelAttributeArgumentResolver(new SimpleConversionService());
        request = mock(HttpServletRequest.class);
    }

    // ─── Test POJO ───────────────────────────────────────────────

    public static class UserForm {
        private String name;
        private int age;

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    // ─── Test handler methods ────────────────────────────────────

    @SuppressWarnings("unused")
    void handlerWithModelAttribute(@ModelAttribute UserForm form) {}

    @SuppressWarnings("unused")
    void handlerWithRequestParam(@RequestParam String name) {}

    @SuppressWarnings("unused")
    void handlerWithoutAnnotation(String plain) {}

    // ─── Tests ───────────────────────────────────────────────────

    @Test
    void shouldSupportParameter_WhenAnnotatedWithModelAttribute() throws Exception {
        MethodParameter param = getParameter("handlerWithModelAttribute", 0);
        assertThat(resolver.supportsParameter(param)).isTrue();
    }

    @Test
    void shouldNotSupportParameter_WhenAnnotatedWithRequestParam() throws Exception {
        MethodParameter param = getParameter("handlerWithRequestParam", 0);
        assertThat(resolver.supportsParameter(param)).isFalse();
    }

    @Test
    void shouldNotSupportParameter_WhenNoAnnotation() throws Exception {
        MethodParameter param = getParameter("handlerWithoutAnnotation", 0);
        assertThat(resolver.supportsParameter(param)).isFalse();
    }

    @Test
    void shouldResolveAndBindPojo_WhenRequestHasMatchingParams() throws Exception {
        setupParameters("name", "Alice", "age", "30");
        MethodParameter param = getParameter("handlerWithModelAttribute", 0);

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isInstanceOf(UserForm.class);
        UserForm form = (UserForm) result;
        assertThat(form.getName()).isEqualTo("Alice");
        assertThat(form.getAge()).isEqualTo(30);
    }

    @Test
    void shouldResolveEmptyPojo_WhenNoRequestParams() throws Exception {
        when(request.getParameterNames()).thenReturn(Collections.emptyEnumeration());
        MethodParameter param = getParameter("handlerWithModelAttribute", 0);

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isInstanceOf(UserForm.class);
        UserForm form = (UserForm) result;
        assertThat(form.getName()).isNull();
        assertThat(form.getAge()).isEqualTo(0);
    }

    // ─── Helpers ─────────────────────────────────────────────────

    private MethodParameter getParameter(String methodName, int index) throws Exception {
        for (Method m : getClass().getDeclaredMethods()) {
            if (m.getName().equals(methodName)) {
                return new MethodParameter(m, index);
            }
        }
        throw new NoSuchMethodException(methodName);
    }

    private void setupParameters(String... keyValues) {
        java.util.Map<String, String> params = new java.util.LinkedHashMap<>();
        for (int i = 0; i < keyValues.length; i += 2) {
            params.put(keyValues[i], keyValues[i + 1]);
        }
        when(request.getParameterNames()).thenReturn(Collections.enumeration(params.keySet()));
        for (var entry : params.entrySet()) {
            when(request.getParameter(entry.getKey())).thenReturn(entry.getValue());
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/DataBindingIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.GetMapping;
import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.annotation.PostMapping;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.Collections;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Integration test for Data Binding (Feature 15) with the full handler pipeline.
 *
 * Tests that @ModelAttribute works end-to-end: handler mapping finds the method,
 * the adapter resolves the @ModelAttribute parameter via data binding, invokes
 * the handler, and processes the return value.
 */
class DataBindingIntegrationTest {

    private SimpleHandlerMapping handlerMapping;
    private SimpleHandlerAdapter handlerAdapter;
    private HttpServletResponse response;
    private StringWriter responseBody;

    // ─── Test POJO ───────────────────────────────────────────────

    public static class CreateUserForm {
        private String username;
        private int age;
        private boolean active;

        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
        public boolean isActive() { return active; }
        public void setActive(boolean active) { this.active = active; }
    }

    // ─── Test Controller ─────────────────────────────────────────

    @Controller
    public static class UserController {

        @PostMapping("/users")
        @ResponseBody
        public String createUser(@ModelAttribute CreateUserForm form) {
            return "Created: " + form.getUsername() + ", age=" + form.getAge()
                    + ", active=" + form.isActive();
        }

        @GetMapping("/users/search")
        @ResponseBody
        public String searchUsers(@RequestParam String query, @ModelAttribute CreateUserForm filter) {
            return "Search: " + query + ", filterAge=" + filter.getAge();
        }
    }

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new UserController());

        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(container);
        handlerAdapter = new SimpleHandlerAdapter();

        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    @Test
    void shouldBindFormFromRequestParams_WhenPostWithModelAttribute() throws Exception {
        HttpServletRequest request = mockRequest("POST", "/users",
                "username", "Alice", "age", "28", "active", "true");

        HandlerMethod handler = handlerMapping.lookupHandler(request);
        assertThat(handler).isNotNull();

        ModelAndView mav = handlerAdapter.handle(request, response, handler);

        assertThat(mav).isNull(); // @ResponseBody returns null ModelAndView
        assertThat(responseBody.toString()).contains("Created: Alice, age=28, active=true");
    }

    @Test
    void shouldBindPartialForm_WhenSomeParamsOmitted() throws Exception {
        HttpServletRequest request = mockRequest("POST", "/users",
                "username", "Bob");

        HandlerMethod handler = handlerMapping.lookupHandler(request);
        ModelAndView mav = handlerAdapter.handle(request, response, handler);

        assertThat(responseBody.toString()).contains("Created: Bob, age=0, active=false");
    }

    @Test
    void shouldMixModelAttributeWithRequestParam_WhenBothPresent() throws Exception {
        HttpServletRequest request = mockRequest("GET", "/users/search",
                "query", "admin", "age", "25");

        HandlerMethod handler = handlerMapping.lookupHandler(request);
        assertThat(handler).isNotNull();

        ModelAndView mav = handlerAdapter.handle(request, response, handler);

        // @RequestParam gets "query", @ModelAttribute gets "age" bound to the form
        assertThat(responseBody.toString()).contains("Search: admin, filterAge=25");
    }

    // ─── Helpers ─────────────────────────────────────────────────

    private HttpServletRequest mockRequest(String method, String path, String... params) {
        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getMethod()).thenReturn(method);
        when(request.getRequestURI()).thenReturn(path);
        // needed for content negotiation
        when(request.getHeader("Accept")).thenReturn("*/*");

        java.util.Map<String, String> paramMap = new java.util.LinkedHashMap<>();
        for (int i = 0; i < params.length; i += 2) {
            paramMap.put(params[i], params[i + 1]);
        }
        when(request.getParameterNames()).thenReturn(Collections.enumeration(paramMap.keySet()));
        for (var entry : paramMap.entrySet()) {
            when(request.getParameter(entry.getKey())).thenReturn(entry.getValue());
        }
        return request;
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Data Binding** | Automatically mapping multiple HTTP request parameters onto a POJO's properties via setter methods or field access |
| **`@ModelAttribute`** | Annotation that triggers data binding for a handler method parameter — the framework creates the object and populates it from the request |
| **`SimpleDataBinder`** | Our collapsed version of Spring's `DataBinder` → `WebDataBinder` → `ServletRequestDataBinder` → `BeanWrapperImpl` hierarchy — creates instances, finds setters, converts types, and sets values |
| **JavaBeans Convention** | The `setXxx()` / `getXxx()` naming pattern that makes property discovery possible without configuration |
| **Resolver Ordering** | Argument resolvers are tried in registration order — specific annotations (`@RequestParam`, `@PathVariable`) take priority over `@ModelAttribute` |

**Next: Chapter 16 — Validation** — Validate `@ModelAttribute` POJOs with `@Valid` and Jakarta Bean Validation, turning constraint violations into structured error responses
