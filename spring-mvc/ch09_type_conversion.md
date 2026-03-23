# Chapter 9: Type Conversion

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `@RequestParam` and `@PathVariable` resolvers extract values from the request and return them as `String`. Controllers can only receive String parameters from query strings and URL paths. | Writing `@RequestParam int page` or `@PathVariable Long id` causes a ClassCastException — there is no mechanism to convert the String `"42"` to the integer `42`. | Build a pluggable `ConversionService` with `Converter<S, T>` interface, pre-register common String→X converters, and integrate into both argument resolvers so typed parameters work automatically. |

---

## 9.1 The Integration Point

The integration point is `RequestParamArgumentResolver.resolveArgument()` and `PathVariableArgumentResolver.resolveArgument()` — the exact methods where String values are returned to the adapter. Adding a conversion step between "extract the String" and "return to caller" connects the existing argument resolution pipeline to the new conversion infrastructure.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/RequestParamArgumentResolver.java`
**Change:** Accept a `ConversionService` in the constructor. After extracting the String value, check if the target type is `String` — if not, delegate to `conversionService.convert()`.

```java
public class RequestParamArgumentResolver implements HandlerMethodArgumentResolver {

    private final ConversionService conversionService;

    public RequestParamArgumentResolver(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String parameterName = getParameterName(parameter);
        String value = request.getParameter(parameterName);

        if (value == null) {
            throw new IllegalStateException(
                    "Missing required request parameter '" + parameterName + "'...");
        }

        Class<?> targetType = parameter.getParameterType();

        // No conversion needed for String parameters
        if (targetType == String.class) {
            return value;
        }

        // Convert String to the declared parameter type
        return conversionService.convert(value, targetType);
    }
    // ...
}
```

**Modifying:** `src/main/java/com/simplespringmvc/adapter/PathVariableArgumentResolver.java`
**Change:** Same pattern — accept `ConversionService` in constructor, convert after extraction.

```java
public class PathVariableArgumentResolver implements HandlerMethodArgumentResolver {

    private final ConversionService conversionService;

    public PathVariableArgumentResolver(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        // ... extract String value from path variables map ...

        Class<?> targetType = parameter.getParameterType();
        if (targetType == String.class) {
            return value;
        }
        return conversionService.convert(value, targetType);
    }
    // ...
}
```

**Direction:** The resolvers now depend on `ConversionService`, which connects the **argument resolution pipeline** to the **type conversion infrastructure**. To make this work, we need to build:
- `Converter<S, T>` — the functional interface for individual converters
- `ConversionService` — the service interface for querying and performing conversions
- `SimpleConversionService` — the implementation with converter registry and built-in String→X converters
- `ConversionException` — runtime exception for conversion failures
- Update `SimpleHandlerAdapter` to create the `ConversionService` and pass it to both resolvers

Two key decisions at the integration point:

1. **Short-circuit for String targets.** When the parameter type is already `String`, we skip the conversion service entirely. This avoids unnecessary overhead for the common case and means we don't need to register a String→String identity converter. The real framework handles this inside `GenericConversionService.convert()` via an `isInstance` check.

2. **ConversionService injected directly, not through WebDataBinder.** In the real framework, the conversion chain is: `AbstractNamedValueMethodArgumentResolver.resolveArgument()` → `WebDataBinder.convertIfNecessary()` → `TypeConverterDelegate` → `ConversionService.convert()`. We skip the middle layers and inject `ConversionService` directly into the resolver. This is a deliberate simplification — `WebDataBinder` exists to combine type conversion + validation + data binding, but we only need conversion right now.

## 9.2 The Converter Functional Interface

**New file:** `src/main/java/com/simplespringmvc/convert/Converter.java`

```java
package com.simplespringmvc.convert;

@FunctionalInterface
public interface Converter<S, T> {

    T convert(S source);
}
```

The `@FunctionalInterface` annotation is deliberate — it enables lambda-based registration:

```java
addConverter(String.class, Integer.class, s -> Integer.valueOf(s.trim()));
```

In the real framework, `Converter<S, T>` is one of three converter types:
- `Converter<S, T>` — 1:1 conversion (what we implement)
- `ConverterFactory<S, R>` — 1:N, e.g., one factory covering String→Integer, String→Long, String→Double
- `GenericConverter` — N:N with full `TypeDescriptor` access for collection conversions

We only need `Converter<S, T>` because our conversions are all simple 1:1 mappings.

## 9.3 The ConversionService Interface

**New file:** `src/main/java/com/simplespringmvc/convert/ConversionService.java`

```java
package com.simplespringmvc.convert;

public interface ConversionService {

    boolean canConvert(Class<?> sourceType, Class<?> targetType);

    <T> T convert(Object source, Class<T> targetType);
}
```

Two methods — query then convert. The real interface has five methods with `TypeDescriptor` parameters for generics-aware conversion (e.g., converting `List<String>` to `List<Integer>`). Since we only convert scalar types, raw `Class<?>` is sufficient.

**New file:** `src/main/java/com/simplespringmvc/convert/ConversionException.java`

```java
package com.simplespringmvc.convert;

public class ConversionException extends RuntimeException {

    public ConversionException(String message) {
        super(message);
    }

    public ConversionException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

The real framework splits this into `ConverterNotFoundException` (no converter registered) and `ConversionFailedException` (converter threw). We use a single exception for simplicity.

## 9.4 The SimpleConversionService Implementation

**New file:** `src/main/java/com/simplespringmvc/convert/SimpleConversionService.java`

This is the core implementation, combining `GenericConversionService` (converter storage + lookup) and `DefaultConversionService` (built-in converter registration) from the real framework.

```java
package com.simplespringmvc.convert;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class SimpleConversionService implements ConversionService {

    private final Map<ConvertiblePair, Converter<?, ?>> converters = new ConcurrentHashMap<>();

    public SimpleConversionService() {
        addDefaultConverters();
    }

    public <S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<S, T> converter) {
        converters.put(new ConvertiblePair(sourceType, targetType), converter);
    }

    @Override
    public boolean canConvert(Class<?> sourceType, Class<?> targetType) {
        if (sourceType == targetType || targetType.isAssignableFrom(sourceType)) {
            return true;
        }
        Class<?> normalizedTarget = boxIfPrimitive(targetType);
        if (normalizedTarget.isEnum()) {
            return converters.containsKey(new ConvertiblePair(sourceType, Enum.class));
        }
        return converters.containsKey(new ConvertiblePair(sourceType, normalizedTarget));
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T convert(Object source, Class<T> targetType) {
        if (targetType.isInstance(source)) {
            return (T) source;
        }

        Class<?> normalizedTarget = boxIfPrimitive(targetType);
        Class<?> sourceType = source.getClass();
        Converter<Object, ?> converter;

        if (normalizedTarget.isEnum()) {
            converter = (Converter<Object, ?>) converters.get(
                    new ConvertiblePair(sourceType, Enum.class));
            if (converter != null) {
                @SuppressWarnings("rawtypes")
                Class<? extends Enum> enumType = (Class<? extends Enum>) targetType;
                return (T) Enum.valueOf(enumType, source.toString().trim());
            }
        } else {
            converter = (Converter<Object, ?>) converters.get(
                    new ConvertiblePair(sourceType, normalizedTarget));
        }

        if (converter == null) {
            throw new ConversionException(
                    "No converter found for " + sourceType.getName() + " → " + targetType.getName());
        }

        return (T) converter.convert(source);
    }

    private void addDefaultConverters() {
        addConverter(String.class, Integer.class, s -> Integer.valueOf(s.trim()));
        addConverter(String.class, Long.class, s -> Long.valueOf(s.trim()));
        addConverter(String.class, Double.class, s -> Double.valueOf(s.trim()));
        addConverter(String.class, Float.class, s -> Float.valueOf(s.trim()));
        addConverter(String.class, Short.class, s -> Short.valueOf(s.trim()));
        addConverter(String.class, Byte.class, s -> Byte.valueOf(s.trim()));
        addConverter(String.class, Boolean.class, s -> {
            String trimmed = s.trim();
            if ("true".equalsIgnoreCase(trimmed)) return Boolean.TRUE;
            if ("false".equalsIgnoreCase(trimmed)) return Boolean.FALSE;
            throw new ConversionException("Cannot convert '" + s + "' to Boolean");
        });
        addConverter(String.class, Character.class, s -> {
            if (s.length() != 1) throw new ConversionException("...");
            return s.charAt(0);
        });
        // Sentinel for enum — actual conversion done in convert() via Enum.valueOf()
        addConverter(String.class, Enum.class, s -> {
            throw new UnsupportedOperationException("Enum conversion handled in convert()");
        });
    }

    private Class<?> boxIfPrimitive(Class<?> type) {
        if (type == int.class) return Integer.class;
        if (type == long.class) return Long.class;
        if (type == double.class) return Double.class;
        if (type == float.class) return Float.class;
        if (type == boolean.class) return Boolean.class;
        if (type == short.class) return Short.class;
        if (type == byte.class) return Byte.class;
        if (type == char.class) return Character.class;
        return type;
    }

    private record ConvertiblePair(Class<?> sourceType, Class<?> targetType) {}
}
```

Three key design decisions in this implementation:

1. **`ConvertiblePair` record as map key.** Uses Java 17 records to get automatic `equals`/`hashCode` based on both source and target types. This mirrors `GenericConverter.ConvertiblePair` in the real framework (which is a regular class with manual equals/hashCode).

2. **`boxIfPrimitive()` for primitive type normalization.** Converters are registered by wrapper type (`Integer.class`), but method parameters can declare primitives (`int.class`). We normalize before lookup. The real framework handles this inside `TypeDescriptor` via `ClassUtils.resolvePrimitiveIfNecessary()`.

3. **Sentinel registration for enum types.** We register a single converter keyed by `(String.class, Enum.class)` as a sentinel — its `convert()` method is never called. Instead, `SimpleConversionService.convert()` detects enum targets and calls `Enum.valueOf(enumType, source)` directly with the concrete enum class. This mimics how the real `StringToEnumConverterFactory` works — one registration covers all enum types.

## 9.5 Wiring Into SimpleHandlerAdapter

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Create a `SimpleConversionService` and pass it to both `PathVariableArgumentResolver` and `RequestParamArgumentResolver`.

```java
public SimpleHandlerAdapter() {
    messageConverters.add(new JacksonMessageConverter());
    conversionService = new SimpleConversionService();                        // NEW

    argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));   // CHANGED
    argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));   // CHANGED
    argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));

    returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
    returnValueHandlers.addHandler(new StringReturnValueHandler());
}
```

The `ConversionService` is a shared dependency — both resolvers use the same instance with the same set of registered converters. In the real framework, this sharing happens through `WebDataBinder` → `TypeConverterDelegate` → `ConversionService`, but the effect is the same: one conversion service, many consumers.

## 9.6 Try It Yourself

<details>
<summary>Challenge: Implement the Converter interface and ConversionService</summary>

Starting from scratch, create:

1. A `@FunctionalInterface` called `Converter<S, T>` with a single `T convert(S source)` method
2. A `ConversionService` interface with `canConvert(Class<?>, Class<?>)` and `<T> T convert(Object, Class<T>)`
3. A `SimpleConversionService` that:
   - Stores converters in a `Map` keyed by (sourceType, targetType) pairs
   - Pre-registers converters for String→Integer, String→Long, String→Boolean, String→Double
   - Handles primitive type boxing (int.class → Integer.class)
   - Handles enum conversion via `Enum.valueOf()`

Test with:
```java
ConversionService cs = new SimpleConversionService();
assert cs.convert("42", int.class) == 42;
assert cs.convert("true", boolean.class) == true;
assert cs.convert("3.14", Double.class) == 3.14;
```

The solution is in `src/main/java/com/simplespringmvc/convert/`.

</details>

<details>
<summary>Challenge: Integrate ConversionService into the argument resolvers</summary>

Given the `SimpleConversionService`, update `RequestParamArgumentResolver` to:

1. Accept a `ConversionService` in its constructor
2. After extracting the String value from `request.getParameter()`, check the parameter's declared type
3. If it's `String`, return as-is. Otherwise, call `conversionService.convert(value, targetType)`

Then do the same for `PathVariableArgumentResolver`.

Hint: The conversion must happen AFTER extracting the value but BEFORE returning it. The target type comes from `parameter.getParameterType()`.

The solution is in:
- `src/main/java/com/simplespringmvc/adapter/RequestParamArgumentResolver.java`
- `src/main/java/com/simplespringmvc/adapter/PathVariableArgumentResolver.java`

</details>

## 9.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/convert/SimpleConversionServiceTest.java`

Tests all built-in converters plus the conversion service infrastructure:

```java
@Test
@DisplayName("should convert valid integer string")
void shouldConvertValidInteger() {
    assertThat(conversionService.convert("42", Integer.class)).isEqualTo(42);
}

@Test
@DisplayName("should convert to primitive int")
void shouldConvertToPrimitiveInt() {
    int result = conversionService.convert("42", int.class);
    assertThat(result).isEqualTo(42);
}

@Test
@DisplayName("should convert valid enum constant name")
void shouldConvertValidEnumName() {
    assertThat(conversionService.convert("RED", TestColor.class)).isEqualTo(TestColor.RED);
}

@Test
@DisplayName("should use custom converter when registered")
void shouldUseCustomConverter() {
    conversionService.addConverter(String.class, java.util.UUID.class,
            s -> java.util.UUID.fromString(s));
    UUID uuid = conversionService.convert("550e8400-e29b-41d4-a716-446655440000", UUID.class);
    assertThat(uuid.toString()).isEqualTo("550e8400-e29b-41d4-a716-446655440000");
}
```

Key test categories:
- `canConvert()` — checks for registered, unregistered, and identity conversions
- String → Integer/int — valid, negative, whitespace trimming, invalid input
- String → Long/long, Double/double, Float/float, Short/short, Byte/byte
- String → Boolean — true, false, case-insensitive, invalid
- String → Character — single char, multi-char error, empty error
- String → Enum — valid constant, invalid constant
- Identity conversion — same-type and assignable (Integer → Number)
- Custom converter — user-registered converter, no-converter error

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/TypeConversionIntegrationTest.java`

Tests the full stack with a real Tomcat server — HTTP request → DispatcherServlet → handler mapping → adapter → resolver → conversion → controller:

- `@RequestParam int page` — query string `?page=2` → integer `2`
- `@RequestParam long id` + `@RequestParam double ratio` + `@RequestParam boolean active` — multiple types in one request
- `@RequestParam Status status` — enum conversion from query string
- `@PathVariable Long id` — path variable `/users/42` → Long `42`
- Mixed `@PathVariable Long id` + `@RequestParam int page` + `@RequestParam boolean includeArchived`

### Modified Tests

The following existing test files were updated to pass `SimpleConversionService` to the resolver constructors (since they now require a `ConversionService`):
- `RequestParamArgumentResolverTest.java` — `new RequestParamArgumentResolver(new SimpleConversionService())`
- `PathVariableArgumentResolverTest.java` — `new PathVariableArgumentResolver(new SimpleConversionService())`
- `HandlerMethodArgumentResolverCompositeTest.java` — same change

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 9.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Why `ConversionService` is separate from `Converter`.** A `Converter<S, T>` knows how to convert one specific type pair. The `ConversionService` knows how to *find the right converter* for a given type pair. This separation is the Strategy pattern at two levels — each converter is a strategy for one conversion, and the service is a strategy for converter selection. In the real framework, this separation enables powerful features: the service can walk class hierarchies to find compatible converters (e.g., a `Number → String` converter also handles `Integer → String`), cache lookups, and compose converters for multi-hop conversions. Our flat map lookup is simpler but loses hierarchy walking — a trade-off that's fine for the String→primitive conversions we need.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Primitive boxing: a necessary bridge.** Java's type system has a split personality — `int.class` and `Integer.class` are different objects in reflection. When a controller declares `@RequestParam int page`, `Method.getParameters()[i].getType()` returns `int.class`, not `Integer.class`. But our converters are registered by wrapper type (`String → Integer`), not primitive type. The `boxIfPrimitive()` method bridges this gap. The real framework handles it the same way, inside `TypeDescriptor` via `ClassUtils.resolvePrimitiveIfNecessary()`. Without this normalization, `@RequestParam int` would throw "no converter found" even though `@RequestParam Integer` works — a confusing user experience.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Why `@FunctionalInterface` matters for converters.** Making `Converter<S, T>` a functional interface isn't just syntactic sugar — it changes how users interact with the conversion system. Instead of writing `class StringToIntConverter implements Converter<String, Integer> { ... }`, you can write `addConverter(String.class, Integer.class, s -> Integer.valueOf(s))`. This is important because most converters are trivial one-liners. The real framework's `Converter<S, T>` has been `@FunctionalInterface` since Spring 4.0 (2013), and it's one of the earliest uses of this pattern in the Spring codebase — predating the widespread adoption of lambda-based configuration.
> -----------------------------------------------------------

## 9.9 What We Enhanced

| Aspect | Before (ch05–ch08) | Current (ch09) | Real Framework |
|--------|---------------------|----------------|----------------|
| `RequestParamArgumentResolver` | Returns raw `String` from `request.getParameter()` — no-arg constructor | Accepts `ConversionService`, converts String to declared parameter type | `AbstractNamedValueMethodArgumentResolver.resolveArgument()` → `WebDataBinder.convertIfNecessary()` → `TypeConverterDelegate` → `ConversionService` |
| `PathVariableArgumentResolver` | Returns raw `String` from path variables map — no-arg constructor | Accepts `ConversionService`, converts String to declared parameter type | Same chain as above — conversion happens in the shared base class |
| `SimpleHandlerAdapter` | Creates resolvers without conversion support | Creates `SimpleConversionService` and passes to both resolvers as shared dependency | `RequestMappingHandlerAdapter.afterPropertiesSet()` initializes `ConfigurableWebBindingInitializer` with `ConversionService`, which is passed to every `WebDataBinder` |
| Type support | Only `String` parameters from `@RequestParam`/`@PathVariable` | `int`, `long`, `double`, `float`, `boolean`, `short`, `byte`, `char`, `Enum`, and custom types via `addConverter()` | ~40 built-in converters including collections, dates, UUIDs, patterns, currencies, plus `ConverterFactory` for range-based registration |
| Conversion infrastructure | None | `Converter<S,T>` + `ConversionService` + `SimpleConversionService` with `Map<ConvertiblePair, Converter>` | Three converter types (`Converter`, `ConverterFactory`, `GenericConverter`), `TypeDescriptor` for generics, class hierarchy walking, soft-reference caching, `ConverterRegistry` interface |

## 9.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Converter<S, T>` (1 method) | `Converter<S, T>` | `Converter.java:49` | Real has `andThen()` default method; `T` allows `@Nullable` |
| `ConversionService` (2 methods) | `ConversionService` | `ConversionService.java:42-75` | Real has 5 methods with `TypeDescriptor` overloads for generics-aware conversion |
| `SimpleConversionService` | `GenericConversionService` + `DefaultConversionService` | `GenericConversionService.java:73`, `DefaultConversionService.java:44` | Real uses two-level `Map<ConvertiblePair, ConvertersForPair>` with deque, class hierarchy walking, soft-reference cache, `NO_MATCH` sentinel, `NoOpConverter` fallback |
| `ConversionException` | `ConversionFailedException` + `ConverterNotFoundException` | `ConversionFailedException.java:31`, `ConverterNotFoundException.java:30` | Real framework has two separate exception types for "no converter" vs "converter failed" |
| `ConvertiblePair` record | `GenericConverter.ConvertiblePair` | `GenericConverter.java:80` | Real is a public class with manual `equals`/`hashCode`; we use Java 17 record |
| `boxIfPrimitive()` | `ClassUtils.resolvePrimitiveIfNecessary()` | `ClassUtils.java:257` | Real handles all 8 primitives and `void.class`; used by `TypeDescriptor` |
| `addDefaultConverters()` — 9 individual converters | `DefaultConversionService.addScalarConverters()` | `DefaultConversionService.java:118` | Real registers ~40 converters via `ConverterFactory` (one factory covers all Number subclasses) |
| `addConverter(Class, Class, Converter)` | `GenericConversionService.addConverter(Class, Class, Converter)` | `GenericConversionService.java:114` | Real wraps in `ConverterAdapter`, stores in deque (last-registered-first), invalidates cache |
| Conversion via resolvers: `conversionService.convert(value, targetType)` | Via `WebDataBinder.convertIfNecessary()` → `TypeConverterDelegate` → `ConversionService` | `AbstractNamedValueMethodArgumentResolver.java:132` | Real goes through WebDataBinder which also handles data binding and validation |

## 9.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/convert/Converter.java` [NEW]

```java
package com.simplespringmvc.convert;

/**
 * A converter that transforms a source object of type {@code S} to a target
 * of type {@code T}. This is the simplest converter contract — a single method
 * that takes one type and returns another.
 *
 * Maps to: {@code org.springframework.core.convert.converter.Converter<S, T>}
 *
 * The real interface:
 * <ul>
 *   <li>Has an {@code andThen(Converter)} default method for chaining</li>
 *   <li>Is one of three converter types: {@code Converter} (1:1), {@code ConverterFactory}
 *       (1:N, e.g., String→Number covering Integer, Long, etc.), and {@code GenericConverter}
 *       (N:N with full TypeDescriptor access)</li>
 *   <li>Uses {@code @Nullable} on the target type parameter to allow null returns</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code andThen()} chaining — single-step conversions only</li>
 *   <li>No {@code ConverterFactory} or {@code GenericConverter} — just direct 1:1 converters</li>
 *   <li>No null return support — conversions always produce a value or throw</li>
 * </ul>
 *
 * @param <S> the source type
 * @param <T> the target type
 */
@FunctionalInterface
public interface Converter<S, T> {

    /**
     * Convert the source object of type {@code S} to target type {@code T}.
     *
     * @param source the source object to convert (never null)
     * @return the converted object
     * @throws IllegalArgumentException if the source cannot be converted
     */
    T convert(S source);
}
```

#### File: `src/main/java/com/simplespringmvc/convert/ConversionService.java` [NEW]

```java
package com.simplespringmvc.convert;

/**
 * A service interface for type conversion. This is the entry point into the
 * conversion system — callers check {@code canConvert()} then call {@code convert()}.
 *
 * Maps to: {@code org.springframework.core.convert.ConversionService}
 *
 * The real interface has five methods with two levels of type information:
 * <ul>
 *   <li>{@code canConvert(Class, Class)} — simple class-based check</li>
 *   <li>{@code canConvert(TypeDescriptor, TypeDescriptor)} — generics-aware check</li>
 *   <li>{@code convert(Object, Class)} — simple conversion</li>
 *   <li>{@code convert(Object, TypeDescriptor)} — default method (since 6.1)</li>
 *   <li>{@code convert(Object, TypeDescriptor, TypeDescriptor)} — full generics-aware conversion</li>
 * </ul>
 *
 * {@code TypeDescriptor} carries generic type info (e.g., {@code List<String>} vs {@code List<Integer>})
 * which is needed for collection conversion. Since we only convert scalar types
 * (String→int, String→Long, etc.), raw {@code Class<?>} is sufficient.
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code TypeDescriptor} — only raw {@code Class<?>} parameters</li>
 *   <li>No generics-aware conversion — no collection element conversion</li>
 *   <li>Two methods instead of five</li>
 * </ul>
 */
public interface ConversionService {

    /**
     * Whether a conversion from {@code sourceType} to {@code targetType} is possible.
     *
     * Maps to: {@code ConversionService.canConvert(Class, Class)} (line 42)
     *
     * @param sourceType the source type to convert from
     * @param targetType the target type to convert to
     * @return true if a converter is registered for this source/target pair
     */
    boolean canConvert(Class<?> sourceType, Class<?> targetType);

    /**
     * Convert the given source value to the specified target type.
     *
     * Maps to: {@code ConversionService.convert(Object, Class)} (line 57)
     *
     * @param source     the source value to convert (never null)
     * @param targetType the target type to convert to
     * @param <T>        the target type
     * @return the converted value
     * @throws ConversionException if conversion fails
     */
    <T> T convert(Object source, Class<T> targetType);
}
```

#### File: `src/main/java/com/simplespringmvc/convert/ConversionException.java` [NEW]

```java
package com.simplespringmvc.convert;

/**
 * Exception thrown when a type conversion fails.
 *
 * Maps to: {@code org.springframework.core.convert.ConversionFailedException}
 * and {@code org.springframework.core.convert.ConverterNotFoundException}
 *
 * The real framework has two exception types:
 * <ul>
 *   <li>{@code ConverterNotFoundException} — no converter registered for the type pair</li>
 *   <li>{@code ConversionFailedException} — converter found but conversion failed</li>
 * </ul>
 *
 * We collapse both into a single exception for simplicity.
 */
public class ConversionException extends RuntimeException {

    public ConversionException(String message) {
        super(message);
    }

    public ConversionException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/convert/SimpleConversionService.java` [NEW]

```java
package com.simplespringmvc.convert;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * A simple implementation of {@link ConversionService} that stores converters
 * in a map keyed by (sourceType, targetType) pairs and pre-registers built-in
 * converters for common String-to-X conversions.
 *
 * Maps to: {@code org.springframework.core.convert.support.GenericConversionService}
 * (converter storage and lookup) combined with
 * {@code org.springframework.core.convert.support.DefaultConversionService}
 * (built-in converter registration).
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   GenericConversionService:
 *     - Stores converters in a two-level structure: Map&lt;ConvertiblePair, ConvertersForPair&gt;
 *       where ConvertersForPair holds a Deque&lt;GenericConverter&gt;
 *     - Lookup walks the full class hierarchy (superclasses + interfaces) of both
 *       source and target to find compatible converters
 *     - Caches lookup results in a ConcurrentReferenceHashMap (soft references)
 *     - Uses a sentinel NO_MATCH value to cache negative lookups
 *     - Falls back to a NoOpConverter when source is assignable to target
 *
 *   DefaultConversionService:
 *     - Registers ~40 converters via addDefaultConverters()
 *     - Uses ConverterFactory for range-based conversions (e.g., StringToNumberConverterFactory
 *       covers Integer, Long, Double, Float, BigDecimal, etc. with one registration)
 *     - Provides a lazily-initialized shared singleton via getSharedInstance()
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Flat map keyed by {@code ConvertiblePair} — no class hierarchy walking</li>
 *   <li>No {@code ConverterFactory} — each target type registered individually</li>
 *   <li>No caching — map lookup is already O(1)</li>
 *   <li>No sentinel caching for misses</li>
 *   <li>No assignability-based fallback (no NoOpConverter)</li>
 *   <li>Built-in converters cover: String→Integer/Long/Boolean/Double/Float/Short/Byte and String→Enum</li>
 * </ul>
 */
public class SimpleConversionService implements ConversionService {

    /**
     * Converter registry keyed by (sourceType, targetType).
     *
     * Maps to: {@code GenericConversionService.Converters.converters}
     * (a {@code Map<ConvertiblePair, ConvertersForPair>})
     *
     * The real version uses a ConcurrentHashMap of ConvertiblePair → ConvertersForPair
     * where ConvertersForPair holds a Deque of converters (last-registered wins).
     * We simplify to a single converter per type pair.
     */
    private final Map<ConvertiblePair, Converter<?, ?>> converters = new ConcurrentHashMap<>();

    /**
     * Creates a ConversionService pre-loaded with default String→X converters.
     *
     * Maps to: {@code DefaultConversionService.addDefaultConverters()} →
     * {@code addScalarConverters()} (line 118)
     *
     * The real version registers converters via:
     * - {@code StringToNumberConverterFactory} — one factory covering all Number subclasses
     * - {@code StringToBooleanConverter} — handles "true"/"false", "on"/"off", "yes"/"no", "1"/"0"
     * - {@code StringToEnumConverterFactory} — one factory covering all Enum types
     *
     * We register individual converters for each numeric type. Our boolean converter
     * only handles "true"/"false" (case-insensitive). Our enum converter handles
     * all enum types via a single registration with special handling in convert().
     */
    public SimpleConversionService() {
        addDefaultConverters();
    }

    // ─── Registration ────────────────────────────────────────────

    /**
     * Register a converter for a specific source→target type pair.
     *
     * Maps to: {@code GenericConversionService.addConverter(Class, Class, Converter)}
     * (line 114)
     *
     * The real version wraps the Converter in a ConverterAdapter (adapting to
     * GenericConverter), stores in ConvertersForPair deque (addFirst — last wins),
     * and invalidates the converter cache.
     *
     * We store directly in the flat map — last registration wins by overwriting.
     *
     * @param sourceType the source type
     * @param targetType the target type
     * @param converter  the converter to register
     * @param <S>        source type
     * @param <T>        target type
     */
    public <S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<S, T> converter) {
        converters.put(new ConvertiblePair(sourceType, targetType), converter);
    }

    // ─── ConversionService implementation ────────────────────────

    /**
     * Check if a converter exists for the given source→target pair.
     *
     * Maps to: {@code GenericConversionService.canConvert(Class, Class)} (line 160)
     *
     * The real version:
     * <pre>
     *   1. Wraps classes in TypeDescriptors
     *   2. Calls getConverter(sourceType, targetType)
     *   3. getConverter() checks cache, then calls converters.find()
     *   4. find() walks the full class hierarchy of both types
     *   5. Returns true if any converter matches
     * </pre>
     *
     * We do a direct map lookup — no hierarchy walking. This means we must
     * register converters for exact types (e.g., String→int AND String→Integer
     * must both be registered, though we handle boxing automatically).
     */
    @Override
    public boolean canConvert(Class<?> sourceType, Class<?> targetType) {
        if (sourceType == targetType || targetType.isAssignableFrom(sourceType)) {
            return true;
        }
        Class<?> normalizedTarget = boxIfPrimitive(targetType);
        // Special case: enum types — we have a single converter for all enums
        if (normalizedTarget.isEnum()) {
            return converters.containsKey(new ConvertiblePair(sourceType, Enum.class));
        }
        return converters.containsKey(new ConvertiblePair(sourceType, normalizedTarget));
    }

    /**
     * Convert the source value to the target type.
     *
     * Maps to: {@code GenericConversionService.convert(Object, Class)} (line 186)
     *
     * The real version:
     * <pre>
     *   1. Get TypeDescriptor for source and target
     *   2. Call getConverter() (cache → find → hierarchy walk)
     *   3. If converter found: ConversionUtils.invokeConverter()
     *   4. If no converter: handleConverterNotFound() — returns source if assignable,
     *      returns null for null source, or throws ConverterNotFoundException
     * </pre>
     *
     * We do the assignability check first (skip conversion if types match),
     * then map lookup, then invoke.
     */
    @Override
    @SuppressWarnings("unchecked")
    public <T> T convert(Object source, Class<T> targetType) {
        // No conversion needed if source is already the target type
        if (targetType.isInstance(source)) {
            return (T) source;
        }

        Class<?> normalizedTarget = boxIfPrimitive(targetType);
        Class<?> sourceType = source.getClass();
        Converter<Object, ?> converter;

        // Special handling for enum types — look up by Enum.class
        if (normalizedTarget.isEnum()) {
            converter = (Converter<Object, ?>) converters.get(new ConvertiblePair(sourceType, Enum.class));
            if (converter != null) {
                try {
                    @SuppressWarnings("rawtypes")
                    Class<? extends Enum> enumType = (Class<? extends Enum>) targetType;
                    return (T) Enum.valueOf(enumType, source.toString().trim());
                } catch (IllegalArgumentException e) {
                    throw new ConversionException(
                            "Cannot convert '" + source + "' to enum type " + targetType.getName(), e);
                }
            }
        } else {
            converter = (Converter<Object, ?>) converters.get(new ConvertiblePair(sourceType, normalizedTarget));
        }

        if (converter == null) {
            throw new ConversionException(
                    "No converter found for " + sourceType.getName() + " → " + targetType.getName());
        }

        try {
            return (T) converter.convert(source);
        } catch (ConversionException e) {
            throw e;
        } catch (Exception e) {
            throw new ConversionException(
                    "Failed to convert '" + source + "' from " + sourceType.getName()
                            + " to " + targetType.getName(), e);
        }
    }

    // ─── Built-in converters ─────────────────────────────────────

    /**
     * Register all default String→X converters.
     *
     * Maps to: {@code DefaultConversionService.addScalarConverters()} (line 118)
     *
     * The real framework uses:
     * - {@code StringToNumberConverterFactory} — one ConverterFactory that creates
     *   converters for any Number subclass via {@code NumberUtils.parseNumber()}
     * - {@code StringToBooleanConverter} — handles "true"/"false", "on"/"off",
     *   "yes"/"no", "1"/"0" (case-insensitive)
     * - {@code StringToCharacterConverter} — takes first character
     * - {@code StringToEnumConverterFactory} — one factory for all enum types
     *
     * We register individual converters for each numeric type and handle
     * boolean with true/false only. The lambda-based registration is possible
     * because Converter is a @FunctionalInterface.
     */
    private void addDefaultConverters() {
        // String → Integer
        // Real: StringToNumberConverterFactory → NumberUtils.parseNumber(source, Integer.class)
        addConverter(String.class, Integer.class, s -> {
            try {
                return Integer.valueOf(s.trim());
            } catch (NumberFormatException e) {
                throw new ConversionException("Cannot convert '" + s + "' to Integer", e);
            }
        });

        // String → Long
        // Real: StringToNumberConverterFactory → NumberUtils.parseNumber(source, Long.class)
        addConverter(String.class, Long.class, s -> {
            try {
                return Long.valueOf(s.trim());
            } catch (NumberFormatException e) {
                throw new ConversionException("Cannot convert '" + s + "' to Long", e);
            }
        });

        // String → Double
        // Real: StringToNumberConverterFactory → NumberUtils.parseNumber(source, Double.class)
        addConverter(String.class, Double.class, s -> {
            try {
                return Double.valueOf(s.trim());
            } catch (NumberFormatException e) {
                throw new ConversionException("Cannot convert '" + s + "' to Double", e);
            }
        });

        // String → Float
        // Real: StringToNumberConverterFactory → NumberUtils.parseNumber(source, Float.class)
        addConverter(String.class, Float.class, s -> {
            try {
                return Float.valueOf(s.trim());
            } catch (NumberFormatException e) {
                throw new ConversionException("Cannot convert '" + s + "' to Float", e);
            }
        });

        // String → Short
        // Real: StringToNumberConverterFactory → NumberUtils.parseNumber(source, Short.class)
        addConverter(String.class, Short.class, s -> {
            try {
                return Short.valueOf(s.trim());
            } catch (NumberFormatException e) {
                throw new ConversionException("Cannot convert '" + s + "' to Short", e);
            }
        });

        // String → Byte
        // Real: StringToNumberConverterFactory → NumberUtils.parseNumber(source, Byte.class)
        addConverter(String.class, Byte.class, s -> {
            try {
                return Byte.valueOf(s.trim());
            } catch (NumberFormatException e) {
                throw new ConversionException("Cannot convert '" + s + "' to Byte", e);
            }
        });

        // String → Boolean
        // Real: StringToBooleanConverter — handles "true"/"false", "on"/"off", "yes"/"no", "1"/"0"
        // We only handle "true"/"false" (case-insensitive) for simplicity.
        addConverter(String.class, Boolean.class, s -> {
            String trimmed = s.trim();
            if ("true".equalsIgnoreCase(trimmed)) {
                return Boolean.TRUE;
            } else if ("false".equalsIgnoreCase(trimmed)) {
                return Boolean.FALSE;
            }
            throw new ConversionException(
                    "Cannot convert '" + s + "' to Boolean — expected 'true' or 'false'");
        });

        // String → Character
        // Real: StringToCharacterConverter — takes first character, throws if length != 1
        addConverter(String.class, Character.class, s -> {
            if (s.isEmpty()) {
                throw new ConversionException("Cannot convert empty String to Character");
            }
            if (s.length() > 1) {
                throw new ConversionException(
                        "Cannot convert String '" + s + "' to Character — must be exactly 1 character");
            }
            return s.charAt(0);
        });

        // String → Enum (sentinel registration — convert() has special enum handling)
        // Real: StringToEnumConverterFactory creates a converter per enum type via
        //   Enum.valueOf(enumType, source.trim())
        // We register a placeholder converter keyed by Enum.class; the actual
        // Enum.valueOf() call is done in convert() which knows the concrete enum type.
        addConverter(String.class, Enum.class, s -> {
            throw new UnsupportedOperationException(
                    "Enum conversion is handled specially in convert() — this should not be called directly");
        });
    }

    // ─── Utility ─────────────────────────────────────────────────

    /**
     * Box a primitive type to its wrapper class. Converters are registered
     * by wrapper type (e.g., Integer.class), so we need to normalize primitive
     * types (e.g., int.class → Integer.class) before lookup.
     *
     * Maps to: The real framework handles this inside TypeDescriptor which
     * always stores the boxed type via {@code ClassUtils.resolvePrimitiveIfNecessary()}.
     */
    private Class<?> boxIfPrimitive(Class<?> type) {
        if (type == int.class) return Integer.class;
        if (type == long.class) return Long.class;
        if (type == double.class) return Double.class;
        if (type == float.class) return Float.class;
        if (type == boolean.class) return Boolean.class;
        if (type == short.class) return Short.class;
        if (type == byte.class) return Byte.class;
        if (type == char.class) return Character.class;
        return type;
    }

    // ─── ConvertiblePair ─────────────────────────────────────────

    /**
     * A pair of (sourceType, targetType) used as a map key for converter lookup.
     *
     * Maps to: {@code GenericConverter.ConvertiblePair} (line 80 in GenericConverter.java)
     * The real version is a public nested class in GenericConverter with the same
     * structure: two Class fields with equals/hashCode based on both.
     */
    private record ConvertiblePair(Class<?> sourceType, Class<?> targetType) {}
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/RequestParamArgumentResolver.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves method arguments annotated with {@link RequestParam} by extracting
 * values from the HTTP request's query string (or form data).
 *
 * Maps to: {@code org.springframework.web.method.annotation.RequestParamMethodArgumentResolver}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @RequestParam}</li>
 *   <li>{@code resolveArgument()} reads the annotation's value to get the parameter name</li>
 *   <li>Falls back to the Java parameter name if no explicit name is given</li>
 *   <li>Calls {@code request.getParameter(name)} to get the String value</li>
 *   <li>Converts the String to the parameter's declared type via {@link ConversionService}</li>
 * </ol>
 *
 * <h3>Parameter name resolution:</h3>
 * When {@code @RequestParam} has no explicit value (e.g., {@code @RequestParam String name}),
 * we fall back to the method parameter name from the Java reflection API. This requires
 * the {@code -parameters} compiler flag, which is already configured in build.gradle.
 * The real framework uses a {@code ParameterNameDiscoverer} chain that can also read
 * bytecode debug info via ASM — we rely solely on the JDK's built-in support.
 *
 * <h3>Type conversion:</h3>
 * Request parameters arrive as Strings from the Servlet API. When the method parameter
 * declares a non-String type (e.g., {@code @RequestParam int page}), the resolver
 * uses the {@link ConversionService} to convert the String to the target type.
 * In the real framework, this conversion happens inside
 * {@code AbstractNamedValueMethodArgumentResolver.resolveArgument()} which calls
 * {@code WebDataBinder.convertIfNecessary()} → {@code TypeConverterDelegate} →
 * {@code ConversionService.convert()}.
 *
 * Simplifications:
 * <ul>
 *   <li>No {@code required} flag or default value handling</li>
 *   <li>No multipart file support</li>
 *   <li>No "default resolution mode" for unannotated simple types</li>
 *   <li>Doesn't extend AbstractNamedValueMethodArgumentResolver (no shared name/default/required logic)</li>
 *   <li>Converts directly via ConversionService instead of via WebDataBinder</li>
 * </ul>
 */
public class RequestParamArgumentResolver implements HandlerMethodArgumentResolver {

    private final ConversionService conversionService;

    /**
     * Create a resolver with the given ConversionService for type conversion.
     *
     * In the real framework, type conversion is handled by WebDataBinder which
     * is created by WebDataBinderFactory. The binder internally uses a
     * TypeConverterDelegate that delegates to ConversionService.
     * We inject ConversionService directly for simplicity.
     */
    public RequestParamArgumentResolver(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    /**
     * Supports parameters annotated with {@code @RequestParam}.
     *
     * Maps to: {@code RequestParamMethodArgumentResolver.supportsParameter()} (line 128)
     * The real version also supports multipart types and (in default mode) unannotated
     * simple types. We only handle the explicit annotation case.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestParam.class);
    }

    /**
     * Extract the query parameter value from the request and convert to the
     * parameter's declared type.
     *
     * Maps to: {@code AbstractNamedValueMethodArgumentResolver.resolveArgument()} (line 99)
     * which calls {@code resolveName()} to get the String value, then
     * {@code binder.convertIfNecessary()} to convert it to the target type.
     *
     * The real conversion chain:
     * <pre>
     *   resolveArgument()
     *     → resolveName() → request.getParameter(name)       // get String
     *     → binder.convertIfNecessary(value, paramType)       // convert
     *       → TypeConverterDelegate.convertIfNecessary()
     *         → conversionService.convert(value, targetType)
     * </pre>
     *
     * We collapse this to: get String → conversionService.convert().
     *
     * @throws IllegalStateException if the parameter is missing from the request
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String parameterName = getParameterName(parameter);
        String value = request.getParameter(parameterName);

        if (value == null) {
            throw new IllegalStateException(
                    "Missing required request parameter '" + parameterName
                            + "' for method parameter type " + parameter.getParameterType().getSimpleName());
        }

        Class<?> targetType = parameter.getParameterType();

        // No conversion needed for String parameters
        if (targetType == String.class) {
            return value;
        }

        // Convert String to the declared parameter type
        return conversionService.convert(value, targetType);
    }

    /**
     * Determine the request parameter name: use the annotation's value() if
     * specified, otherwise fall back to the method parameter name.
     *
     * Maps to: {@code AbstractNamedValueMethodArgumentResolver.updateNamedValueInfo()} (line 168)
     * The real version has a full NamedValueInfo chain with required/defaultValue support.
     */
    private String getParameterName(MethodParameter parameter) {
        RequestParam annotation = parameter.getParameterAnnotation(RequestParam.class);
        String name = (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
        return name;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/PathVariableArgumentResolver.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.PathVariable;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.http.HttpServletRequest;

import java.util.Map;

/**
 * Resolves method arguments annotated with {@link PathVariable} by reading
 * extracted URI template variable values from a request attribute.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.PathVariableMethodArgumentResolver}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@link SimpleHandlerMapping#lookupHandler(HttpServletRequest)} matches the request
 *       path against a pattern like {@code /users/{id}} and stores the extracted variables
 *       ({@code {"id" → "42"}}) as a request attribute</li>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @PathVariable}</li>
 *   <li>{@code resolveArgument()} reads the extracted variables map from the request
 *       attribute and looks up the value by variable name</li>
 *   <li>Converts the String value to the parameter's declared type via {@link ConversionService}</li>
 * </ol>
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   PathVariableMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver
 *   1. supportsParameter() checks for @PathVariable annotation
 *   2. createNamedValueInfo() extracts name, required, defaultValue from the annotation
 *   3. resolveName() reads the Map from request attribute HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE
 *   4. Base class handles type conversion via WebDataBinder.convertIfNecessary()
 *   5. handleResolvedValue() stores the type-converted value in View.PATH_VARIABLES attribute
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Doesn't extend AbstractNamedValueMethodArgumentResolver — standalone resolver</li>
 *   <li>No {@code required} flag handling</li>
 *   <li>No {@code View.PATH_VARIABLES} attribute storage</li>
 *   <li>No support for Map parameters (all path vars as Map)</li>
 *   <li>Converts directly via ConversionService instead of via WebDataBinder</li>
 * </ul>
 */
public class PathVariableArgumentResolver implements HandlerMethodArgumentResolver {

    private final ConversionService conversionService;

    /**
     * Create a resolver with the given ConversionService for type conversion.
     *
     * In the real framework, type conversion is handled by WebDataBinder which
     * is created by WebDataBinderFactory. The binder internally uses a
     * TypeConverterDelegate that delegates to ConversionService.
     * We inject ConversionService directly for simplicity.
     */
    public PathVariableArgumentResolver(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    /**
     * Supports parameters annotated with {@code @PathVariable}.
     *
     * Maps to: {@code PathVariableMethodArgumentResolver.supportsParameter()} (line 73)
     * The real version also checks if the parameter type is Map (without a specific
     * variable name). We only handle the simple case.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(PathVariable.class);
    }

    /**
     * Extract the path variable value from the request attributes and convert
     * to the parameter's declared type.
     *
     * Maps to: {@code AbstractNamedValueMethodArgumentResolver.resolveArgument()} (line 99)
     * which calls {@code resolveName()} → {@code binder.convertIfNecessary()}.
     *
     * The real conversion chain:
     * <pre>
     *   resolveArgument()
     *     → resolveName() → uriTemplateVars.get(name)         // get String
     *     → binder.convertIfNecessary(value, paramType)        // convert
     *       → TypeConverterDelegate.convertIfNecessary()
     *         → conversionService.convert(value, targetType)
     * </pre>
     *
     * @throws IllegalStateException if the path variable is not found
     */
    @Override
    @SuppressWarnings("unchecked")
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        String variableName = getVariableName(parameter);

        // Read the path variables map stored by SimpleHandlerMapping during lookup.
        // Maps to: PathVariableMethodArgumentResolver.resolveName() (line 93):
        //   Map<String, String> uriTemplateVars = (Map<String, String>) request.getAttribute(
        //       HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
        Map<String, String> pathVariables = (Map<String, String>) request.getAttribute(
                SimpleHandlerMapping.PATH_VARIABLES_ATTRIBUTE);

        if (pathVariables == null) {
            throw new IllegalStateException(
                    "No path variables found in request. Was the handler resolved by SimpleHandlerMapping?");
        }

        String value = pathVariables.get(variableName);
        if (value == null) {
            // Maps to: PathVariableMethodArgumentResolver.handleMissingValue() (line 100)
            // which throws MissingPathVariableException
            throw new IllegalStateException(
                    "Missing path variable '" + variableName + "' for method parameter type "
                            + parameter.getParameterType().getSimpleName()
                            + ". Available path variables: " + pathVariables.keySet());
        }

        Class<?> targetType = parameter.getParameterType();

        // No conversion needed for String parameters
        if (targetType == String.class) {
            return value;
        }

        // Convert String to the declared parameter type
        return conversionService.convert(value, targetType);
    }

    /**
     * Determine the path variable name: use the annotation's value() if
     * specified, otherwise fall back to the method parameter name.
     *
     * Maps to: {@code PathVariableMethodArgumentResolver.createNamedValueInfo()} (line 85)
     */
    private String getVariableName(MethodParameter parameter) {
        PathVariable annotation = parameter.getParameterAnnotation(PathVariable.class);
        String name = (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
        return name;
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
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;

/**
 * The default HandlerAdapter that knows how to invoke {@link HandlerMethod}
 * instances via reflection, resolving method arguments from the request first
 * and processing the return value through the return value handler chain.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 * which extends {@code AbstractHandlerMethodAdapter}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handleInternal() creates a ServletInvocableHandlerMethod
 *   3. Sets argument resolvers on the invocable method
 *   4. Sets return value handlers on the invocable method
 *   5. Calls invocableMethod.invokeAndHandle()
 *   6. invokeAndHandle() resolves arguments, invokes via reflection,
 *      handles the return value
 * </pre>
 *
 * We collapse that chain:
 * <pre>
 *   1. supports() checks if handler is a HandlerMethod
 *   2. handle() resolves arguments via the resolver chain
 *   3. Invokes the method via reflection with the resolved arguments
 *   4. Delegates return value to the return value handler chain
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No InvocableHandlerMethod wrapper — we call Method.invoke() directly</li>
 *   <li>No ModelAndView return — response is written directly</li>
 *   <li>No WebDataBinderFactory — ch15 adds data binding</li>
 *   <li>No ModelFactory — ch13 adds model support</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();
    private final List<HttpMessageConverter> messageConverters = new ArrayList<>();
    private final ConversionService conversionService;

    public SimpleHandlerAdapter() {
        // Initialize shared infrastructure that resolvers and handlers depend on.
        // Maps to: RequestMappingHandlerAdapter.afterPropertiesSet() (line 344)
        // Real version initializes converters, ConversionService, then builds
        // resolver and handler lists.
        messageConverters.add(new JacksonMessageConverter());
        conversionService = new SimpleConversionService();

        // Register the default argument resolvers.
        // Maps to: RequestMappingHandlerAdapter.getDefaultArgumentResolvers() (line 644)
        // Real version registers ~30 resolvers in a specific order.
        // ORDER MATTERS: resolvers are checked in registration order. @PathVariable
        // comes before @RequestParam to match the real framework's ordering.
        // @RequestBody comes after @RequestParam — same as real framework line 663.
        // The ConversionService is shared by both @PathVariable and @RequestParam
        // resolvers — in the real framework this sharing happens through WebDataBinder.
        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));

        // Register the default return value handlers.
        // Maps to: RequestMappingHandlerAdapter.getDefaultReturnValueHandlers() (line 730)
        // ORDER MATTERS: @ResponseBody handler must come BEFORE the String fallback,
        // because the composite uses first-match. This is the same priority scheme
        // the real framework uses — annotation-based handlers before catch-all handlers.
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    /**
     * Create an adapter with custom resolvers added BEFORE the defaults.
     * This mirrors how the real framework's setCustomArgumentResolvers() works.
     */
    public SimpleHandlerAdapter(List<HandlerMethodArgumentResolver> customResolvers) {
        messageConverters.add(new JacksonMessageConverter());
        conversionService = new SimpleConversionService();
        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(messageConverters));
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    /**
     * This adapter supports only HandlerMethod instances — the type produced
     * by our SimpleHandlerMapping when scanning @RequestMapping methods.
     *
     * Maps to: {@code AbstractHandlerMethodAdapter.supports()} (line 44)
     * which checks {@code handler instanceof HandlerMethod}
     */
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    /**
     * Resolve arguments, invoke the handler method, and write the result to the response.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.handleInternal()} (line 831)
     * → {@code invokeHandlerMethod()} (line 885)
     * → {@code ServletInvocableHandlerMethod.invokeAndHandle()} (line 114)
     * → {@code InvocableHandlerMethod.invokeForRequest()} (line 171)
     *
     * The real chain creates an InvocableHandlerMethod, configures argument
     * resolvers and return value handlers, then invokes. We do a simplified
     * version that still follows the same resolve → invoke → handle-return flow.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the HandlerMethod to invoke
     */
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // Step 1: Resolve method arguments from the request (ch05).
        // Maps to: InvocableHandlerMethod.getMethodArgumentValues() (line 186)
        Object[] args = resolveArguments(handlerMethod, request);

        // Step 2: Invoke the controller method via reflection.
        // Maps to: InvocableHandlerMethod.doInvoke() (line 243)
        Object result = invokeHandlerMethod(handlerMethod, args);

        // Step 3: Process the return value through the handler chain.
        // Maps to: ServletInvocableHandlerMethod.invokeAndHandle() (line 114)
        // → HandlerMethodReturnValueHandlerComposite.handleReturnValue()
        // The composite iterates handlers: @ResponseBody → JSON, else → text/plain.
        returnValueHandlers.handleReturnValue(result, handlerMethod, request, response);
    }

    /**
     * Resolve each method parameter into an argument value using the resolver chain.
     *
     * Maps to: {@code InvocableHandlerMethod.getMethodArgumentValues()} (line 186)
     *
     * The real version:
     * <pre>
     *   for each parameter:
     *     if resolvers.supportsParameter(param):
     *       args[i] = resolvers.resolveArgument(param, mavContainer, webRequest, binderFactory)
     *     else:
     *       throw IllegalStateException("no suitable resolver")
     * </pre>
     *
     * We follow the same pattern but with our simplified resolver interface.
     * Parameters without a matching resolver get null — this allows no-arg
     * methods and methods with a mix of resolved and unresolved params to work.
     * Future chapters will tighten this as more resolvers are added.
     */
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

    /**
     * Invoke the handler method with the resolved arguments and return the result.
     *
     * Maps to: {@code InvocableHandlerMethod.doInvoke()} (line 243)
     * The real version uses getBridgedMethod() for generic type bridge methods,
     * handles Kotlin suspend functions, and formats detailed error messages.
     */
    private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
        } catch (InvocationTargetException ex) {
            // Unwrap the real exception from the reflection wrapper.
            // Maps to: InvocableHandlerMethod.doInvoke() (line 260-275)
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    /**
     * Provides access to the argument resolver composite for testing.
     */
    public HandlerMethodArgumentResolverComposite getArgumentResolvers() {
        return argumentResolvers;
    }

    /**
     * Provides access to the return value handler composite for testing.
     */
    public HandlerMethodReturnValueHandlerComposite getReturnValueHandlers() {
        return returnValueHandlers;
    }

    /**
     * Provides access to the message converters for testing.
     */
    public List<HttpMessageConverter> getMessageConverters() {
        return messageConverters;
    }

    /**
     * Provides access to the conversion service for testing.
     */
    public ConversionService getConversionService() {
        return conversionService;
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/convert/SimpleConversionServiceTest.java` [NEW]

See section 9.7 for test descriptions. Full source in `src/test/java/com/simplespringmvc/convert/SimpleConversionServiceTest.java`.

#### File: `src/test/java/com/simplespringmvc/integration/TypeConversionIntegrationTest.java` [NEW]

See section 9.7 for test descriptions. Full source in `src/test/java/com/simplespringmvc/integration/TypeConversionIntegrationTest.java`.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`Converter<S, T>`** | A `@FunctionalInterface` that converts one type to another — enables lambda registration |
| **`ConversionService`** | The entry point for type conversion — `canConvert()` checks, `convert()` performs |
| **`SimpleConversionService`** | Implementation with `Map<ConvertiblePair, Converter>` storage, pre-registered String→X converters, and primitive boxing |
| **`ConvertiblePair`** | A (sourceType, targetType) record used as the map key — mirrors `GenericConverter.ConvertiblePair` |
| **`boxIfPrimitive()`** | Normalizes `int.class` → `Integer.class` so converters registered by wrapper type also handle primitives |
| **Enum sentinel** | One converter registered for `(String, Enum.class)` as a marker; actual `Enum.valueOf()` happens in `convert()` with the concrete type |

**Next: Chapter 10 — Composed Annotations** — Support `@GetMapping`, `@PostMapping`, and `@RestController` through meta-annotation detection — annotations on annotations. This removes boilerplate from `@RequestMapping(path = "/users", method = "GET")` to just `@GetMapping("/users")`.
