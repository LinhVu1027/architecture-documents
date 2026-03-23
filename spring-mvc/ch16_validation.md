# Chapter 16: Validation

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `@ModelAttribute` binds request parameters to POJOs, but accepts any values — a negative age, a null name, or an empty email all pass through silently | No way to enforce business rules on bound objects; invalid data reaches the controller and must be checked manually | Add `@Valid` support that runs Jakarta Bean Validation constraints (`@NotNull`, `@Min`, `@Size`, etc.) after data binding, throwing `MethodArgumentNotValidException` with structured error details |

---

## 16.1 The Integration Point

The integration point is `ModelAttributeArgumentResolver.resolveArgument()` — the exact place where data binding finishes and the freshly-bound POJO is about to be returned to the handler adapter. This is where validation inserts itself: *after* binding succeeds, *before* the handler method sees the object.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolver.java`
**Change:** Add a `validateIfApplicable()` call between `dataBinder.bind()` and `return target`

```java
@Override
public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
    Class<?> targetType = parameter.getParameterType();
    Object target = dataBinder.bind(targetType, request);

    // NEW: Step 6 — validateIfApplicable
    validateIfApplicable(target, parameter);

    return target;
}
```

The resolver also gains a new field:

```java
private final SimpleValidatorAdapter validatorAdapter;

public ModelAttributeArgumentResolver(ConversionService conversionService) {
    this.dataBinder = new SimpleDataBinder(conversionService);
    this.validatorAdapter = new SimpleValidatorAdapter();  // NEW
}
```

Two key decisions here:

1. **Validation runs inside the argument resolver, not in the adapter or dispatcher.** In the real framework, `ModelAttributeMethodProcessor.resolveArgument()` calls `validateIfApplicable()` at line 246 — keeping validation close to binding means the two are always in sync. If we moved validation to the adapter, every resolver that populates an object would need its own validation hook.

2. **Validation is opt-in via `@Valid`.** A POJO can have constraint annotations (`@NotNull`, `@Size`) but they're inert until the controller method parameter also carries `@Valid`. This decouples the domain model (constraints) from the web layer (when to validate).

This connects **data binding** to **Jakarta Bean Validation**. To make it work, we need to build:
- `FieldError` — represents one constraint violation on one field
- `BindingResult` — collects all `FieldError`s for a target object
- `SimpleValidatorAdapter` — bridges Jakarta `Validator` to our `BindingResult`
- `MethodArgumentNotValidException` — thrown when validation fails

---

## 16.2 FieldError — One Violation, One Field

A `FieldError` captures exactly one thing that went wrong on one property. We use a Java record for immutability.

**New file:** `src/main/java/com/simplespringmvc/validation/FieldError.java`

```java
package com.simplespringmvc.validation;

public record FieldError(
        String field,
        Object rejectedValue,
        String message,
        String code
) {
    public FieldError(String field, Object rejectedValue, String message) {
        this(field, rejectedValue, message, "Invalid");
    }

    @Override
    public String toString() {
        return "Field '" + field + "': rejected value [" + rejectedValue + "] — " + message;
    }
}
```

The `code` is the constraint annotation's simple name (`"NotNull"`, `"Size"`, `"Min"`). The real framework's `FieldError` extends `ObjectError` → `DefaultMessageSourceResolvable` for i18n, but a record captures the same essential data.

---

## 16.3 BindingResult — The Error Collector

`BindingResult` holds the target object and a list of `FieldError`s. After validation, the caller checks `hasErrors()`.

**New file:** `src/main/java/com/simplespringmvc/validation/BindingResult.java`

```java
package com.simplespringmvc.validation;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class BindingResult {

    private final Object target;
    private final String objectName;
    private final List<FieldError> fieldErrors = new ArrayList<>();

    public BindingResult(Object target, String objectName) {
        this.target = target;
        this.objectName = objectName;
    }

    public Object getTarget() { return target; }
    public String getObjectName() { return objectName; }

    public void addFieldError(FieldError error) {
        fieldErrors.add(error);
    }

    public boolean hasErrors() { return !fieldErrors.isEmpty(); }
    public int getErrorCount() { return fieldErrors.size(); }

    public List<FieldError> getFieldErrors() {
        return Collections.unmodifiableList(fieldErrors);
    }
}
```

The real framework has a deep hierarchy: `Errors` → `BindingResult` → `AbstractBindingResult` → `AbstractPropertyBindingResult` → `BeanPropertyBindingResult`. It also supports global (object-level) errors, nested paths, and hierarchical message code resolution. Our single class captures the core: collect field errors, check if any exist, expose them as an unmodifiable list.

---

## 16.4 SimpleValidatorAdapter — Bridging Jakarta and Spring

The adapter bootstraps a Jakarta `Validator` from Hibernate Validator (already on the classpath) and converts `ConstraintViolation`s into `FieldError`s.

**New file:** `src/main/java/com/simplespringmvc/validation/SimpleValidatorAdapter.java`

```java
package com.simplespringmvc.validation;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.Path;
import jakarta.validation.Validation;
import jakarta.validation.Validator;
import jakarta.validation.ValidatorFactory;
import java.util.Set;

public class SimpleValidatorAdapter {

    private final Validator validator;

    public SimpleValidatorAdapter() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        this.validator = factory.getValidator();
    }

    public void validate(Object target, BindingResult bindingResult) {
        Set<ConstraintViolation<Object>> violations = validator.validate(target);
        for (ConstraintViolation<Object> violation : violations) {
            String field = determineField(violation);
            Object rejectedValue = violation.getInvalidValue();
            String message = violation.getMessage();
            String code = determineErrorCode(violation);
            bindingResult.addFieldError(new FieldError(field, rejectedValue, message, code));
        }
    }

    private String determineField(ConstraintViolation<?> violation) {
        String field = "";
        for (Path.Node node : violation.getPropertyPath()) {
            field = node.getName();
        }
        return field != null ? field : "";
    }

    private String determineErrorCode(ConstraintViolation<?> violation) {
        return violation.getConstraintDescriptor()
                .getAnnotation().annotationType().getSimpleName();
    }
}
```

Three methods mirror the real `SpringValidatorAdapter`:
- `validate()` → calls `targetValidator.validate()` then `processConstraintViolations()`
- `determineField()` → extracts the property name from the JSR-303 `Path` (real version handles indexed paths like `addresses[0].street`)
- `determineErrorCode()` → returns the annotation's simple name (`"NotNull"`, `"Size"`)

---

## 16.5 MethodArgumentNotValidException — Signaling Failure

This checked exception carries the `MethodParameter` that failed and the `BindingResult` with all errors.

**New file:** `src/main/java/com/simplespringmvc/validation/MethodArgumentNotValidException.java`

```java
package com.simplespringmvc.validation;

import com.simplespringmvc.mapping.MethodParameter;
import java.util.stream.Collectors;

public class MethodArgumentNotValidException extends Exception {

    private final MethodParameter parameter;
    private final BindingResult bindingResult;

    public MethodArgumentNotValidException(MethodParameter parameter, BindingResult bindingResult) {
        super(buildMessage(parameter, bindingResult));
        this.parameter = parameter;
        this.bindingResult = bindingResult;
    }

    public MethodParameter getParameter() { return parameter; }
    public BindingResult getBindingResult() { return bindingResult; }

    private static String buildMessage(MethodParameter parameter, BindingResult bindingResult) {
        String errors = bindingResult.getFieldErrors().stream()
                .map(fe -> fe.field() + ": " + fe.message())
                .collect(Collectors.joining(", ", "[", "]"));
        return "Validation failed for argument [" + parameter.getParameterIndex()
                + "] in " + parameter.getMethod()
                + " with " + bindingResult.getErrorCount() + " error(s): " + errors;
    }
}
```

The real `MethodArgumentNotValidException` extends `BindException` and implements `ErrorResponse` for RFC 7807 `ProblemDetail` support. It always reports HTTP 400.

---

## 16.6 The Complete Integration — validateIfApplicable()

Back at the integration point, the `validateIfApplicable()` method ties everything together:

```java
private void validateIfApplicable(Object target, MethodParameter parameter)
        throws MethodArgumentNotValidException {
    if (!parameter.hasParameterAnnotation(Valid.class)) {
        return;  // No @Valid → skip validation entirely
    }

    String objectName = uncapitalize(target.getClass().getSimpleName());
    BindingResult bindingResult = new BindingResult(target, objectName);
    validatorAdapter.validate(target, bindingResult);

    if (bindingResult.hasErrors()) {
        throw new MethodArgumentNotValidException(parameter, bindingResult);
    }
}
```

This maps directly to the real framework's flow in `ModelAttributeMethodProcessor`:
1. `validateIfApplicable(binder, parameter)` at line 246 — iterates annotations, detects `@Valid`
2. `binder.validate(hints)` → `DataBinder.validate()` at line 1371 — runs all registered validators
3. `isBindExceptionRequired(parameter)` at line 259 — checks if next param is `Errors`/`BindingResult`
4. If required, `throw new MethodArgumentNotValidException(parameter, binder.getBindingResult())`

Our simplification: we always throw (no `BindingResult` parameter suppression), and we only detect `@jakarta.validation.Valid` (not `@Validated` with groups).

---

## 16.7 Try It Yourself

<details>
<summary>Challenge 1: Implement the FieldError record</summary>

Given that you need to represent one validation error with a field name, rejected value, message, and constraint code, implement it as a Java record:

```java
public record FieldError(
        String field,
        Object rejectedValue,
        String message,
        String code
) {
    public FieldError(String field, Object rejectedValue, String message) {
        this(field, rejectedValue, message, "Invalid");
    }

    @Override
    public String toString() {
        return "Field '" + field + "': rejected value [" + rejectedValue + "] — " + message;
    }
}
```

</details>

<details>
<summary>Challenge 2: Wire validation into the resolver</summary>

The key challenge is: where exactly does validation go in `resolveArgument()`? Consider:
- It must run *after* binding (you need a populated object to validate)
- It must run *before* returning (the handler method should never see invalid data)
- It should be opt-in (only when `@Valid` is present)

```java
@Override
public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
    Class<?> targetType = parameter.getParameterType();
    Object target = dataBinder.bind(targetType, request);
    validateIfApplicable(target, parameter);
    return target;
}

private void validateIfApplicable(Object target, MethodParameter parameter)
        throws MethodArgumentNotValidException {
    if (!parameter.hasParameterAnnotation(Valid.class)) {
        return;
    }
    String objectName = uncapitalize(target.getClass().getSimpleName());
    BindingResult bindingResult = new BindingResult(target, objectName);
    validatorAdapter.validate(target, bindingResult);
    if (bindingResult.hasErrors()) {
        throw new MethodArgumentNotValidException(parameter, bindingResult);
    }
}
```

</details>

<details>
<summary>Challenge 3: Build a @ControllerAdvice that returns structured JSON errors</summary>

Combine validation with exception handling (ch12) to produce an API-friendly error response:

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseBody
    public Map<String, Object> handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> Map.of(
                        "field", fe.field(),
                        "message", fe.message(),
                        "code", fe.code()))
                .toList();
        return Map.of("status", 400, "errors", errors);
    }
}
```

</details>

---

## 16.8 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/validation/BindingResultTest.java`

```java
@Test
void shouldReportNoErrors_WhenNewlyCreated()
@Test
void shouldCollectFieldErrors_WhenErrorsAdded()
@Test
void shouldReturnUnmodifiableList_WhenGetFieldErrorsCalled()
@Test
void shouldPreserveTargetAndObjectName()
@Test
void shouldIncludeErrorsInToString_WhenErrorsExist()
```

**New file:** `src/test/java/com/simplespringmvc/validation/SimpleValidatorAdapterTest.java`

```java
@Test
void shouldReportNoErrors_WhenAllConstraintsSatisfied()
@Test
void shouldReportFieldError_WhenNotNullViolated()
@Test
void shouldReportFieldError_WhenMinViolated()
@Test
void shouldReportFieldError_WhenSizeViolated()
@Test
void shouldReportMultipleErrors_WhenMultipleViolations()
@Test
void shouldReportNoErrors_WhenNoConstraintsOnPojo()
@Test
void shouldSetCorrectErrorCode_WhenConstraintDetected()
```

**New file:** `src/test/java/com/simplespringmvc/validation/MethodArgumentNotValidExceptionTest.java`

```java
@Test
void shouldCarryParameterAndBindingResult()
@Test
void shouldBuildDescriptiveMessage_WhenErrorsPresent()
@Test
void shouldBeCheckedException()
```

**Modified file:** `src/test/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolverTest.java`

New validation tests added:
```java
@Test
void shouldPassValidation_WhenValidAnnotationPresentAndConstraintsSatisfied()
@Test
void shouldThrowException_WhenValidAnnotationPresentAndConstraintViolated()
@Test
void shouldThrowException_WhenMinConstraintViolated()
@Test
void shouldSkipValidation_WhenValidAnnotationAbsent()
```

### Integration Test

**New file:** `src/test/java/com/simplespringmvc/integration/ValidationIntegrationTest.java`

Tests the full pipeline: handler mapping → adapter → `@Valid` + `@ModelAttribute` → validation → exception handling via `@ControllerAdvice`:

```java
@Test
void shouldProcessRequest_WhenAllConstraintsSatisfied()
@Test
void shouldThrowValidationException_WhenNotNullViolated()
@Test
void shouldThrowValidationException_WhenMinViolated()
@Test
void shouldThrowValidationException_WhenSizeViolated()
@Test
void shouldReportMultipleErrors_WhenMultipleViolations()
@Test
void shouldNotValidate_WhenValidAnnotationAbsent()
@Test
void shouldReturnStructuredError_WhenExceptionHandlerCatchesValidation()
```

**Run:** `./gradlew test` — expected: all 48+ tests pass (including all prior features' tests)

---

## 16.9 Why This Works

> ★ **Insight** -------------------------------------------
> **Validation as a post-binding decorator, not a binding replacement.** The key design
> insight is that validation doesn't change how data enters the object — it checks the
> result *after* binding is complete. This separation means:
> - Binding errors (type mismatch) and validation errors (business rule violation) are
>   orthogonal concerns
> - The same constraint annotations work regardless of how the object was populated
>   (form params, JSON body, programmatic)
> - You can skip validation entirely by omitting `@Valid`, useful for partial updates
>
> The real framework takes this further: `SpringValidatorAdapter.processConstraintViolations()`
> skips fields that already have binding failures, preventing validation errors from
> overwriting type mismatch errors. Our version assumes binding succeeded (since
> `DataBindingException` is fail-fast), so we don't need this check.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The `@Valid` trigger pattern — annotation-driven activation.** Constraint annotations
> on the POJO (`@NotNull`, `@Size`) are metadata, but they don't *do* anything until
> `@Valid` appears on the method parameter. This two-level activation means:
> - A POJO can be reused in validated and unvalidated contexts
> - The controller author explicitly opts into validation per endpoint
> - The real framework extends this with `@Validated(Group.class)` for validation groups,
>   letting different endpoints validate different subsets of constraints on the same POJO
>
> When `@Valid` is NOT the right trigger: for programmatic validation (e.g., in a service
> layer), call `Validator.validate(object)` directly instead of relying on the web layer.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **MethodArgumentNotValidException as a structured error carrier.** Making this a checked
> `Exception` (not `RuntimeException`) is deliberate — it forces explicit handling. The
> exception carries the full `BindingResult`, so a `@ControllerAdvice` handler can extract
> every field error, format them as JSON, and return a structured API response. In the real
> framework, this exception also implements `ErrorResponse` with `ProblemDetail` for
> RFC 7807 standardized error responses.
>
> This combines with Feature 12 (Exception Handling): the existing `@ExceptionHandler` +
> `@ControllerAdvice` infrastructure catches validation exceptions automatically, with zero
> changes to the exception resolver.
> -----------------------------------------------------------

---

## 16.10 What We Enhanced

| Aspect | Before (ch15) | Current (ch16) | Real Framework |
|--------|---------------|----------------|----------------|
| **Data validation** | None — any value passed through silently after binding | `@Valid` triggers Jakarta Bean Validation; constraint violations throw `MethodArgumentNotValidException` | `ModelAttributeMethodProcessor.validateIfApplicable()` at line 246 with `ValidationAnnotationUtils` supporting `@Valid`, `@Validated`, and custom annotations |
| **Error collection** | `DataBindingException` (fail-fast, single error) | `BindingResult` collects all validation errors into `FieldError` list | `BeanPropertyBindingResult` extends `AbstractPropertyBindingResult` with nested paths, global errors, message code resolution, and model integration |
| **Constraint-to-error translation** | N/A | `SimpleValidatorAdapter` extracts field name, rejected value, message, and error code from each `ConstraintViolation` | `SpringValidatorAdapter.processConstraintViolations()` at line 133 with binding-failure skip, hierarchical error codes, constraint attribute extraction, and `ViolationFieldError` wrapping |
| **Exception propagation** | Binding errors as `RuntimeException` | `MethodArgumentNotValidException` (checked) carrying `MethodParameter` + `BindingResult` | `MethodArgumentNotValidException` extends `BindException` implements `ErrorResponse` with `ProblemDetail` (RFC 7807), HTTP 400 status |
| **`@Valid` detection** | N/A | Direct check for `jakarta.validation.Valid` on the parameter | `ValidationAnnotationUtils.determineValidationHints()` detects `@Valid`, `@Validated`, meta-annotations, and any annotation starting with "Valid" |

---

## 16.11 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `FieldError` (record) | `FieldError` | `FieldError.java:38` | Real extends `ObjectError` → `DefaultMessageSourceResolvable` with hierarchical error codes, i18n arguments, and binding-failure flag |
| `BindingResult` (class) | `BindingResult` (interface) + `BeanPropertyBindingResult` | `BindingResult.java:36`, `BeanPropertyBindingResult.java:43` | Real is a deep interface hierarchy with nested path support, model map integration, message code resolution, and suppressed field tracking |
| `SimpleValidatorAdapter` | `SpringValidatorAdapter` | `SpringValidatorAdapter.java:82` | Real also implements `jakarta.validation.Validator`, supports validation groups via `SmartValidator`, skips fields with binding failures, wraps `ConstraintViolation` in inner `ViolationFieldError` |
| `MethodArgumentNotValidException` | `MethodArgumentNotValidException` | `MethodArgumentNotValidException.java:42` | Real extends `BindException` (inheriting all `Errors` methods) and implements `ErrorResponse` for RFC 7807 `ProblemDetail` |
| `validateIfApplicable()` | `ModelAttributeMethodProcessor.validateIfApplicable()` | `ModelAttributeMethodProcessor.java:246` | Real uses `ValidationAnnotationUtils.determineValidationHints()` to detect `@Valid`, `@Validated(groups)`, and meta-annotated annotations; checks `isBindExceptionRequired()` for next-parameter suppression |
| `uncapitalize()` for object name | `ModelFactory.getNameForParameter()` → `Conventions.getVariableNameForParameter()` | `ModelFactory.java:174` | Real uses `Conventions` class that handles arrays, collections, and generic type resolution for naming |

---

## 16.12 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/validation/FieldError.java` [NEW]

```java
package com.simplespringmvc.validation;

/**
 * Represents a field-level validation error — one constraint violation on
 * one property of the validated object.
 *
 * Maps to: {@code org.springframework.validation.FieldError}
 *
 * The real FieldError extends ObjectError (which extends DefaultMessageSourceResolvable):
 * <ul>
 *   <li>Supports hierarchical error codes for i18n message resolution
 *       (e.g., "NotNull.userForm.name", "NotNull.name", "NotNull")</li>
 *   <li>Carries error arguments for parameterized messages
 *       (e.g., "{min}" and "{max}" for @Size)</li>
 *   <li>Distinguishes binding failures (type mismatch) from validation failures
 *       via the {@code bindingFailure} flag</li>
 *   <li>Wraps the original {@code ConstraintViolation} as the "source" for unwrapping</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Flat structure — no ObjectError/DefaultMessageSourceResolvable hierarchy</li>
 *   <li>Single message string — no i18n code resolution</li>
 *   <li>No binding failure distinction — we only create FieldErrors for validation</li>
 *   <li>No ConstraintViolation wrapping</li>
 * </ul>
 *
 * @param field         the property name that failed validation (e.g., "name", "age")
 * @param rejectedValue the actual value that was rejected (can be null)
 * @param message       the human-readable error message from the constraint validator
 * @param code          the constraint annotation's simple name (e.g., "NotNull", "Size", "Min")
 */
public record FieldError(
        String field,
        Object rejectedValue,
        String message,
        String code
) {

    /**
     * Convenience constructor without explicit code — derives the code from
     * the message prefix or uses "Invalid" as default.
     */
    public FieldError(String field, Object rejectedValue, String message) {
        this(field, rejectedValue, message, "Invalid");
    }

    @Override
    public String toString() {
        return "Field '" + field + "': rejected value [" + rejectedValue + "] — " + message;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/validation/BindingResult.java` [NEW]

```java
package com.simplespringmvc.validation;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Collects validation errors for a single target object. After data binding
 * populates a POJO, the validator writes errors here; the caller then checks
 * {@link #hasErrors()} to decide whether to proceed or throw.
 *
 * Maps to: {@code org.springframework.validation.BindingResult} (interface)
 * and {@code org.springframework.validation.BeanPropertyBindingResult} (implementation)
 *
 * <h3>The real framework's error model:</h3>
 * <pre>
 *   Errors (interface)
 *     └── BindingResult (interface, extends Errors)
 *           └── AbstractBindingResult
 *                 └── AbstractPropertyBindingResult
 *                       └── BeanPropertyBindingResult
 * </pre>
 *
 * The real BindingResult:
 * <ul>
 *   <li>Holds both global (object-level) errors and field-level errors</li>
 *   <li>Supports nested paths for sub-object validation (e.g., "address.city")</li>
 *   <li>Provides a model map containing both the target and itself for view rendering</li>
 *   <li>Resolves hierarchical message codes via MessageCodesResolver</li>
 *   <li>Carries the PropertyEditorRegistry for custom editors</li>
 *   <li>Tracks suppressed fields (filtered by allowedFields/disallowedFields)</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Single class — no interface/abstract hierarchy</li>
 *   <li>Field errors only — no global (object-level) errors</li>
 *   <li>No nested path support</li>
 *   <li>No message code resolution — uses plain error messages</li>
 *   <li>No model integration</li>
 * </ul>
 */
public class BindingResult {

    private final Object target;
    private final String objectName;
    private final List<FieldError> fieldErrors = new ArrayList<>();

    /**
     * Create a BindingResult for the given target object.
     *
     * Maps to: {@code new BeanPropertyBindingResult(target, objectName)}
     *
     * @param target     the validated object
     * @param objectName the logical name (used in error messages)
     */
    public BindingResult(Object target, String objectName) {
        this.target = target;
        this.objectName = objectName;
    }

    /**
     * The validated target object.
     *
     * Maps to: {@code BindingResult.getTarget()}
     */
    public Object getTarget() {
        return target;
    }

    /**
     * The logical name of the target object (e.g., "userForm").
     *
     * Maps to: {@code Errors.getObjectName()}
     */
    public String getObjectName() {
        return objectName;
    }

    /**
     * Add a field error to this result.
     *
     * Maps to: {@code BindingResult.addError(ObjectError)} — the real version
     * accepts any ObjectError (including FieldError subclass). We only support FieldError.
     */
    public void addFieldError(FieldError error) {
        fieldErrors.add(error);
    }

    /**
     * Whether this result contains any errors.
     *
     * Maps to: {@code Errors.hasErrors()}
     */
    public boolean hasErrors() {
        return !fieldErrors.isEmpty();
    }

    /**
     * The total number of errors.
     *
     * Maps to: {@code Errors.getErrorCount()}
     */
    public int getErrorCount() {
        return fieldErrors.size();
    }

    /**
     * All field errors, in the order they were added.
     *
     * Maps to: {@code Errors.getFieldErrors()}
     */
    public List<FieldError> getFieldErrors() {
        return Collections.unmodifiableList(fieldErrors);
    }

    @Override
    public String toString() {
        return "BindingResult for '" + objectName + "': " + fieldErrors.size() + " error(s)" +
                (hasErrors() ? " — " + fieldErrors : "");
    }
}
```

#### File: `src/main/java/com/simplespringmvc/validation/MethodArgumentNotValidException.java` [NEW]

```java
package com.simplespringmvc.validation;

import com.simplespringmvc.mapping.MethodParameter;

import java.util.stream.Collectors;

/**
 * Thrown when validation on a {@code @Valid}-annotated method argument fails
 * and the controller method does not declare a {@code BindingResult} parameter
 * to receive the errors.
 *
 * Maps to: {@code org.springframework.web.bind.MethodArgumentNotValidException}
 *
 * <h3>The real exception:</h3>
 * <pre>
 *   MethodArgumentNotValidException extends BindException implements ErrorResponse
 *     - Carries MethodParameter + BindingResult
 *     - Implements ErrorResponse for RFC 7807 ProblemDetail support
 *     - getStatusCode() → 400 BAD_REQUEST
 *     - getBody() → ProblemDetail with "Invalid request content" detail
 *     - getDetailMessageArguments() → [globalErrors, fieldErrors] as resolved strings
 * </pre>
 *
 * <h3>When is it thrown?</h3>
 * In the real framework, {@code ModelAttributeMethodProcessor.resolveArgument()} checks
 * {@code isBindExceptionRequired()} — if the next parameter is NOT an Errors/BindingResult,
 * the exception is thrown. If the next parameter IS Errors/BindingResult, errors are
 * silently passed through so the controller can handle them.
 *
 * Simplifications:
 * <ul>
 *   <li>Always thrown — no BindingResult-parameter-suppression check</li>
 *   <li>No RFC 7807 ProblemDetail support</li>
 *   <li>No BindException superclass chain</li>
 *   <li>Direct access to BindingResult rather than inheriting Errors methods</li>
 * </ul>
 */
public class MethodArgumentNotValidException extends Exception {

    private final MethodParameter parameter;
    private final BindingResult bindingResult;

    /**
     * Create a new exception for a failed validation.
     *
     * Maps to: {@code new MethodArgumentNotValidException(MethodParameter, BindingResult)}
     *
     * @param parameter     the method parameter that failed validation
     * @param bindingResult the result containing the validation errors
     */
    public MethodArgumentNotValidException(MethodParameter parameter, BindingResult bindingResult) {
        super(buildMessage(parameter, bindingResult));
        this.parameter = parameter;
        this.bindingResult = bindingResult;
    }

    /**
     * The method parameter that triggered validation failure.
     */
    public MethodParameter getParameter() {
        return parameter;
    }

    /**
     * The binding result containing all validation errors.
     *
     * Maps to: {@code MethodArgumentNotValidException.getBindingResult()}
     * (inherited from BindException in the real framework)
     */
    public BindingResult getBindingResult() {
        return bindingResult;
    }

    /**
     * Build a human-readable message listing all validation errors.
     *
     * Maps to: {@code MethodArgumentNotValidException.getMessage()} (line 109)
     * Real version: "Validation failed for argument [N] in public ReturnType method(...)
     * with N errors: [field1: msg1, field2: msg2]"
     */
    private static String buildMessage(MethodParameter parameter, BindingResult bindingResult) {
        String errors = bindingResult.getFieldErrors().stream()
                .map(fe -> fe.field() + ": " + fe.message())
                .collect(Collectors.joining(", ", "[", "]"));

        return "Validation failed for argument [" + parameter.getParameterIndex()
                + "] in " + parameter.getMethod()
                + " with " + bindingResult.getErrorCount() + " error(s): " + errors;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/validation/SimpleValidatorAdapter.java` [NEW]

```java
package com.simplespringmvc.validation;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.Path;
import jakarta.validation.Validation;
import jakarta.validation.Validator;
import jakarta.validation.ValidatorFactory;

import java.util.Set;

/**
 * Bridges Jakarta Bean Validation (JSR 380) with our simplified validation infrastructure.
 * Takes a target object, runs the Jakarta {@link Validator}, and converts
 * {@link ConstraintViolation}s into {@link FieldError}s in a {@link BindingResult}.
 *
 * Maps to: {@code org.springframework.validation.beanvalidation.SpringValidatorAdapter}
 *
 * <h3>How the real adapter works:</h3>
 * <pre>
 *   SpringValidatorAdapter implements SmartValidator, jakarta.validation.Validator
 *     1. validate(target, errors) → targetValidator.validate(target)
 *     2. processConstraintViolations(violations, errors):
 *        For each violation:
 *          a. determineField(violation) → Spring field name (handles indexed paths)
 *          b. Check if field already has binding failure → skip if so
 *          c. determineErrorCode(cd) → annotation simple name (e.g., "NotNull")
 *          d. getArgumentsForConstraint() → [fieldName, annotation attributes...]
 *          e. Create ViolationFieldError or ViolationObjectError
 *          f. Add to BindingResult via addError()
 * </pre>
 *
 * <h3>Key design decisions in the real framework:</h3>
 * <ul>
 *   <li>Skips fields that already have binding failures — validation errors don't
 *       overwrite type mismatch errors (this is why binding runs first)</li>
 *   <li>Wraps the ConstraintViolation as the error's "source" for later unwrapping</li>
 *   <li>Uses hierarchical error codes for i18n (e.g., "NotNull.userForm.name")</li>
 *   <li>Extracts constraint attributes as error arguments for message interpolation</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Creates the ValidatorFactory once at construction (real version uses LocalValidatorFactoryBean)</li>
 *   <li>No binding-failure skip — we assume binding succeeded before validation runs</li>
 *   <li>No hierarchical error codes — uses the annotation simple name directly</li>
 *   <li>No error argument extraction — uses the interpolated message as-is</li>
 *   <li>No ConstraintViolation wrapping in the FieldError</li>
 * </ul>
 */
public class SimpleValidatorAdapter {

    private final Validator validator;

    /**
     * Create the adapter, bootstrapping a Jakarta Validator from the default provider
     * (Hibernate Validator, which is on the classpath).
     *
     * Maps to: {@code LocalValidatorFactoryBean.afterPropertiesSet()} which builds
     * the ValidatorFactory via {@code Validation.buildDefaultValidatorFactory()}.
     */
    public SimpleValidatorAdapter() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        this.validator = factory.getValidator();
    }

    /**
     * Validate the given target object and populate the BindingResult with any errors.
     *
     * Maps to: {@code SpringValidatorAdapter.validate(Object, Errors)} (line 104)
     * → {@code processConstraintViolations(violations, errors)} (line 133)
     *
     * @param target        the object to validate (already populated via data binding)
     * @param bindingResult the result to populate with validation errors
     */
    public void validate(Object target, BindingResult bindingResult) {
        Set<ConstraintViolation<Object>> violations = validator.validate(target);
        processConstraintViolations(violations, bindingResult);
    }

    /**
     * Convert each Jakarta ConstraintViolation into a simplified FieldError and
     * add it to the BindingResult.
     *
     * Maps to: {@code SpringValidatorAdapter.processConstraintViolations()} (line 133)
     *
     * The real version:
     * <pre>
     *   for each violation:
     *     field = determineField(violation)        // property path → Spring field name
     *     if field already has bindingFailure → skip
     *     code = determineErrorCode(descriptor)    // annotation simple name
     *     args = getArgumentsForConstraint(...)    // field name + annotation attrs
     *     if (field is empty) → ViolationObjectError (class-level constraint)
     *     else → ViolationFieldError (property-level constraint)
     *     errors.addError(error)
     * </pre>
     *
     * We simplify: extract field name from property path, use the annotation name
     * as the error code, and use the already-interpolated message.
     */
    private void processConstraintViolations(Set<ConstraintViolation<Object>> violations,
                                              BindingResult bindingResult) {
        for (ConstraintViolation<Object> violation : violations) {
            String field = determineField(violation);
            Object rejectedValue = violation.getInvalidValue();
            String message = violation.getMessage();
            String code = determineErrorCode(violation);

            bindingResult.addFieldError(new FieldError(field, rejectedValue, message, code));
        }
    }

    /**
     * Extract the field name from the violation's property path.
     *
     * Maps to: {@code SpringValidatorAdapter.determineField(ConstraintViolation)} (line 195)
     *
     * The real version handles indexed paths like "addresses[0].street" by converting
     * the JSR-303 {@code Path} to a Spring property path string. We just take the
     * last node's name, which works for simple (non-nested) properties.
     */
    private String determineField(ConstraintViolation<?> violation) {
        Path propertyPath = violation.getPropertyPath();
        String field = "";
        for (Path.Node node : propertyPath) {
            field = node.getName();
        }
        return field != null ? field : "";
    }

    /**
     * Derive the error code from the constraint annotation's simple class name.
     *
     * Maps to: {@code SpringValidatorAdapter.determineErrorCode(ConstraintDescriptor)} (line 229)
     * Real version: {@code descriptor.getAnnotation().annotationType().getSimpleName()}
     *
     * Examples: {@code @NotNull} → "NotNull", {@code @Size} → "Size", {@code @Min} → "Min"
     */
    private String determineErrorCode(ConstraintViolation<?> violation) {
        return violation.getConstraintDescriptor()
                .getAnnotation()
                .annotationType()
                .getSimpleName();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolver.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.binding.SimpleDataBinder;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.validation.BindingResult;
import com.simplespringmvc.validation.MethodArgumentNotValidException;
import com.simplespringmvc.validation.SimpleValidatorAdapter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;

/**
 * Resolves method arguments annotated with {@link ModelAttribute} by creating
 * a new instance of the parameter type, binding request parameters to it,
 * and optionally validating if {@code @Valid} is present.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor}
 * which extends {@code org.springframework.web.method.annotation.ModelAttributeMethodProcessor}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @ModelAttribute}</li>
 *   <li>{@code resolveArgument()} delegates to {@link SimpleDataBinder} to create
 *       and populate the target POJO from request parameters</li>
 *   <li>If the parameter has {@code @Valid}, runs Jakarta Bean Validation</li>
 *   <li>If validation errors exist, throws {@link MethodArgumentNotValidException}</li>
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
 *     6. validateIfApplicable(binder, parameter)
 *        → Iterates parameter annotations
 *        → Uses ValidationAnnotationUtils.determineValidationHints()
 *        → Calls binder.validate(hints) → DataBinder.validate()
 *        → Delegates to SpringValidatorAdapter → jakarta.validation.Validator
 *     7. Check bindingResult.hasErrors()
 *        → If next param is Errors/BindingResult → pass through
 *        → Otherwise → throw MethodArgumentNotValidException
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
 *   <li>No BindingResult parameter support — always throws on validation failure</li>
 *   <li>No attribute name resolution — always creates a new instance</li>
 *   <li>No validation groups ({@code @Validated(Group.class)}) — only {@code @Valid}</li>
 * </ul>
 */
public class ModelAttributeArgumentResolver implements HandlerMethodArgumentResolver {

    private final SimpleDataBinder dataBinder;
    private final SimpleValidatorAdapter validatorAdapter;

    public ModelAttributeArgumentResolver(ConversionService conversionService) {
        this.dataBinder = new SimpleDataBinder(conversionService);
        this.validatorAdapter = new SimpleValidatorAdapter();
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
     * Create an instance of the parameter type, bind request parameters to it,
     * and validate if {@code @Valid} is present.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.resolveArgument()} (line 106)
     *
     * The real flow:
     * <pre>
     *   1. name = ModelFactory.getNameForParameter(parameter)
     *   2. if (mavContainer.containsAttribute(name)) → get from model
     *   3. else → createAttribute() or DataBinder.construct()
     *   4. binder = binderFactory.createBinder(request, attribute, name)
     *   5. binder.bind(servletRequest)       // property binding
     *   6. validateIfApplicable(binder, parameter)  // @Valid → jakarta.validation
     *   7. check bindingResult.hasErrors()   → throw MethodArgumentNotValidException
     *   8. add attribute + bindingResult to model
     * </pre>
     *
     * We collapse this to: create → bind → validate → return (or throw).
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();
        Object target = dataBinder.bind(targetType, request);

        // Step 6: validateIfApplicable — check for @Valid on the parameter
        validateIfApplicable(target, parameter);

        return target;
    }

    /**
     * If the parameter is annotated with {@code @Valid}, run Jakarta Bean Validation
     * and throw {@link MethodArgumentNotValidException} if violations are found.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.validateIfApplicable()} (line 246)
     *
     * <h3>Real framework flow:</h3>
     * <pre>
     *   for (Annotation ann : parameter.getParameterAnnotations()) {
     *       Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
     *       if (validationHints != null) {
     *           binder.validate(validationHints);  // → DataBinder.validate()
     *           break;                             // only first match
     *       }
     *   }
     * </pre>
     *
     * {@code ValidationAnnotationUtils.determineValidationHints()} detects:
     * <ol>
     *   <li>{@code @Validated} — returns validation groups from value()</li>
     *   <li>{@code @jakarta.validation.Valid} — returns empty array (no groups)</li>
     *   <li>Any annotation meta-annotated with {@code @Validated}</li>
     *   <li>Any annotation whose simple name starts with "Valid"</li>
     * </ol>
     *
     * We simplify to: just check for {@code @jakarta.validation.Valid}.
     *
     * <h3>The "isBindExceptionRequired" check:</h3>
     * After validation, the real framework checks if the NEXT parameter in the method
     * signature is of type {@code Errors}/{@code BindingResult}. If so, errors are
     * silently passed through (the controller handles them). If not, the exception
     * is thrown. We always throw — no BindingResult parameter support.
     */
    private void validateIfApplicable(Object target, MethodParameter parameter) throws MethodArgumentNotValidException {
        if (!parameter.hasParameterAnnotation(Valid.class)) {
            return;
        }

        // Derive object name from type (e.g., UserForm → "userForm")
        // Maps to: ModelFactory.getNameForParameter() which uses
        // Conventions.getVariableNameForParameter() → uncapitalize(ClassName)
        String objectName = uncapitalize(target.getClass().getSimpleName());

        BindingResult bindingResult = new BindingResult(target, objectName);
        validatorAdapter.validate(target, bindingResult);

        if (bindingResult.hasErrors()) {
            throw new MethodArgumentNotValidException(parameter, bindingResult);
        }
    }

    /**
     * Uncapitalize the first character: "UserForm" → "userForm".
     */
    private static String uncapitalize(String name) {
        if (name == null || name.isEmpty()) {
            return name;
        }
        return Character.toLowerCase(name.charAt(0)) + name.substring(1);
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/validation/BindingResultTest.java` [NEW]

```java
package com.simplespringmvc.validation;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link BindingResult}.
 *
 * Verifies error collection, querying, and the immutable view contract.
 */
class BindingResultTest {

    @Test
    void shouldReportNoErrors_WhenNewlyCreated() {
        BindingResult result = new BindingResult(new Object(), "target");

        assertThat(result.hasErrors()).isFalse();
        assertThat(result.getErrorCount()).isZero();
        assertThat(result.getFieldErrors()).isEmpty();
    }

    @Test
    void shouldCollectFieldErrors_WhenErrorsAdded() {
        BindingResult result = new BindingResult(new Object(), "userForm");

        result.addFieldError(new FieldError("name", null, "must not be null", "NotNull"));
        result.addFieldError(new FieldError("age", -1, "must be greater than 0", "Min"));

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.getErrorCount()).isEqualTo(2);
        assertThat(result.getFieldErrors()).hasSize(2);
        assertThat(result.getFieldErrors().get(0).field()).isEqualTo("name");
        assertThat(result.getFieldErrors().get(1).field()).isEqualTo("age");
    }

    @Test
    void shouldReturnUnmodifiableList_WhenGetFieldErrorsCalled() {
        BindingResult result = new BindingResult(new Object(), "form");
        result.addFieldError(new FieldError("x", null, "error", "E"));

        assertThatThrownBy(() -> result.getFieldErrors().add(
                new FieldError("y", null, "hack", "H")))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void shouldPreserveTargetAndObjectName() {
        Object target = "hello";
        BindingResult result = new BindingResult(target, "greeting");

        assertThat(result.getTarget()).isSameAs(target);
        assertThat(result.getObjectName()).isEqualTo("greeting");
    }

    @Test
    void shouldIncludeErrorsInToString_WhenErrorsExist() {
        BindingResult result = new BindingResult(new Object(), "form");
        result.addFieldError(new FieldError("email", "", "must not be empty", "NotEmpty"));

        String str = result.toString();
        assertThat(str).contains("form");
        assertThat(str).contains("1 error(s)");
        assertThat(str).contains("email");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/validation/SimpleValidatorAdapterTest.java` [NEW]

```java
package com.simplespringmvc.validation;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link SimpleValidatorAdapter}.
 *
 * Verifies that Jakarta Bean Validation constraints are correctly translated
 * into our simplified FieldError/BindingResult model.
 */
class SimpleValidatorAdapterTest {

    private SimpleValidatorAdapter adapter;

    @BeforeEach
    void setUp() {
        adapter = new SimpleValidatorAdapter();
    }

    // ─── Test POJOs with Jakarta constraints ────────────────────

    public static class UserForm {
        @NotNull
        private String name;

        @Min(0)
        private int age;

        @Size(min = 1, max = 100)
        private String email;

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    public static class NoConstraintsPojo {
        private String value;
        public String getValue() { return value; }
        public void setValue(String value) { this.value = value; }
    }

    // ─── Tests ───────────────────────────────────────────────────

    @Test
    void shouldReportNoErrors_WhenAllConstraintsSatisfied() {
        UserForm form = new UserForm();
        form.setName("Alice");
        form.setAge(25);
        form.setEmail("alice@test.com");

        BindingResult result = new BindingResult(form, "userForm");
        adapter.validate(form, result);

        assertThat(result.hasErrors()).isFalse();
    }

    @Test
    void shouldReportFieldError_WhenNotNullViolated() {
        UserForm form = new UserForm();
        // name is null → violates @NotNull
        form.setAge(25);
        form.setEmail("test@test.com");

        BindingResult result = new BindingResult(form, "userForm");
        adapter.validate(form, result);

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.getFieldErrors()).anyMatch(
                fe -> fe.field().equals("name") && fe.code().equals("NotNull"));
    }

    @Test
    void shouldReportFieldError_WhenMinViolated() {
        UserForm form = new UserForm();
        form.setName("Bob");
        form.setAge(-5);  // violates @Min(0)
        form.setEmail("bob@test.com");

        BindingResult result = new BindingResult(form, "userForm");
        adapter.validate(form, result);

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.getFieldErrors()).anyMatch(
                fe -> fe.field().equals("age")
                        && fe.code().equals("Min")
                        && fe.rejectedValue().equals(-5));
    }

    @Test
    void shouldReportFieldError_WhenSizeViolated() {
        UserForm form = new UserForm();
        form.setName("Charlie");
        form.setAge(30);
        form.setEmail("");  // violates @Size(min=1)

        BindingResult result = new BindingResult(form, "userForm");
        adapter.validate(form, result);

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.getFieldErrors()).anyMatch(
                fe -> fe.field().equals("email") && fe.code().equals("Size"));
    }

    @Test
    void shouldReportMultipleErrors_WhenMultipleViolations() {
        UserForm form = new UserForm();
        // name is null (@NotNull), age is -1 (@Min(0)), email is "" (@Size(min=1))
        form.setAge(-1);
        form.setEmail("");

        BindingResult result = new BindingResult(form, "userForm");
        adapter.validate(form, result);

        assertThat(result.hasErrors()).isTrue();
        assertThat(result.getErrorCount()).isGreaterThanOrEqualTo(3);
    }

    @Test
    void shouldReportNoErrors_WhenNoConstraintsOnPojo() {
        NoConstraintsPojo pojo = new NoConstraintsPojo();
        // No constraints → no violations regardless of values

        BindingResult result = new BindingResult(pojo, "pojo");
        adapter.validate(pojo, result);

        assertThat(result.hasErrors()).isFalse();
    }

    @Test
    void shouldSetCorrectErrorCode_WhenConstraintDetected() {
        UserForm form = new UserForm();
        form.setAge(25);
        form.setEmail("ok@test.com");
        // name is null → @NotNull

        BindingResult result = new BindingResult(form, "userForm");
        adapter.validate(form, result);

        FieldError notNullError = result.getFieldErrors().stream()
                .filter(fe -> fe.field().equals("name"))
                .findFirst()
                .orElseThrow();

        // The error code should be the constraint annotation's simple name
        assertThat(notNullError.code()).isEqualTo("NotNull");
        // The message should be the interpolated constraint message
        assertThat(notNullError.message()).isNotEmpty();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/validation/MethodArgumentNotValidExceptionTest.java` [NEW]

```java
package com.simplespringmvc.validation;

import com.simplespringmvc.mapping.MethodParameter;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for {@link MethodArgumentNotValidException}.
 *
 * Verifies that the exception carries the correct parameter, binding result,
 * and produces a useful error message.
 */
class MethodArgumentNotValidExceptionTest {

    @SuppressWarnings("unused")
    void sampleMethod(String arg0, Object arg1) {}

    @Test
    void shouldCarryParameterAndBindingResult() throws Exception {
        MethodParameter param = getParameter("sampleMethod", 0);
        BindingResult bindingResult = new BindingResult(new Object(), "target");
        bindingResult.addFieldError(new FieldError("name", null, "must not be null", "NotNull"));

        MethodArgumentNotValidException ex =
                new MethodArgumentNotValidException(param, bindingResult);

        assertThat(ex.getParameter()).isSameAs(param);
        assertThat(ex.getBindingResult()).isSameAs(bindingResult);
    }

    @Test
    void shouldBuildDescriptiveMessage_WhenErrorsPresent() throws Exception {
        MethodParameter param = getParameter("sampleMethod", 1);
        BindingResult bindingResult = new BindingResult(new Object(), "form");
        bindingResult.addFieldError(new FieldError("age", -1, "must be >= 0", "Min"));
        bindingResult.addFieldError(new FieldError("name", null, "required", "NotNull"));

        MethodArgumentNotValidException ex =
                new MethodArgumentNotValidException(param, bindingResult);

        assertThat(ex.getMessage())
                .contains("Validation failed")
                .contains("argument [1]")
                .contains("2 error(s)")
                .contains("age")
                .contains("name");
    }

    @Test
    void shouldBeCheckedException() {
        // MethodArgumentNotValidException extends Exception (not RuntimeException)
        // This matches the real framework and ensures it's handled explicitly
        assertThat(Exception.class).isAssignableFrom(MethodArgumentNotValidException.class);
        assertThat(RuntimeException.class.isAssignableFrom(MethodArgumentNotValidException.class))
                .isFalse();
    }

    // ─── Helpers ─────────────────────────────────────────────────

    private MethodParameter getParameter(String methodName, int index) throws Exception {
        java.lang.reflect.Method method = getClass().getDeclaredMethod(
                methodName, String.class, Object.class);
        return new MethodParameter(method, index);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolverTest.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.convert.SimpleConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.validation.MethodArgumentNotValidException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;
import java.util.Collections;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link ModelAttributeArgumentResolver}.
 *
 * Tests the resolver's parameter support check, argument resolution (data binding),
 * and validation integration (@Valid triggering Jakarta Bean Validation).
 */
class ModelAttributeArgumentResolverTest {

    private ModelAttributeArgumentResolver resolver;
    private HttpServletRequest request;

    @BeforeEach
    void setUp() {
        resolver = new ModelAttributeArgumentResolver(new SimpleConversionService());
        request = mock(HttpServletRequest.class);
    }

    // ─── Test POJOs ──────────────────────────────────────────────

    public static class UserForm {
        private String name;
        private int age;

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    public static class ValidatedForm {
        @NotNull
        private String name;

        @Min(0)
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

    @SuppressWarnings("unused")
    void handlerWithValid(@Valid @ModelAttribute ValidatedForm form) {}

    @SuppressWarnings("unused")
    void handlerWithoutValid(@ModelAttribute ValidatedForm form) {}

    // ─── Support tests ───────────────────────────────────────────

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

    // ─── Binding tests (existing) ────────────────────────────────

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

    // ─── Validation tests (ch16) ─────────────────────────────────

    @Test
    void shouldPassValidation_WhenValidAnnotationPresentAndConstraintsSatisfied() throws Exception {
        setupParameters("name", "Alice", "age", "25");
        MethodParameter param = getParameter("handlerWithValid", 0);

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isInstanceOf(ValidatedForm.class);
        ValidatedForm form = (ValidatedForm) result;
        assertThat(form.getName()).isEqualTo("Alice");
        assertThat(form.getAge()).isEqualTo(25);
    }

    @Test
    void shouldThrowException_WhenValidAnnotationPresentAndConstraintViolated() throws Exception {
        // name is missing (null) → @NotNull violated
        setupParameters("age", "25");
        MethodParameter param = getParameter("handlerWithValid", 0);

        assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                .isInstanceOf(MethodArgumentNotValidException.class)
                .satisfies(ex -> {
                    var validEx = (MethodArgumentNotValidException) ex;
                    assertThat(validEx.getBindingResult().hasErrors()).isTrue();
                    assertThat(validEx.getBindingResult().getFieldErrors())
                            .anyMatch(fe -> fe.field().equals("name")
                                    && fe.code().equals("NotNull"));
                });
    }

    @Test
    void shouldThrowException_WhenMinConstraintViolated() throws Exception {
        setupParameters("name", "Bob", "age", "-5");
        MethodParameter param = getParameter("handlerWithValid", 0);

        assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                .isInstanceOf(MethodArgumentNotValidException.class)
                .satisfies(ex -> {
                    var validEx = (MethodArgumentNotValidException) ex;
                    assertThat(validEx.getBindingResult().getFieldErrors())
                            .anyMatch(fe -> fe.field().equals("age")
                                    && fe.code().equals("Min"));
                });
    }

    @Test
    void shouldSkipValidation_WhenValidAnnotationAbsent() throws Exception {
        // The form has constraints, but the parameter does NOT have @Valid
        // → no validation should run, even with violations
        setupParameters("age", "-5");  // @Min(0) would fail if validated
        MethodParameter param = getParameter("handlerWithoutValid", 0);

        Object result = resolver.resolveArgument(param, request);

        // No exception thrown — validation was skipped
        assertThat(result).isInstanceOf(ValidatedForm.class);
        ValidatedForm form = (ValidatedForm) result;
        assertThat(form.getAge()).isEqualTo(-5);
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

#### File: `src/test/java/com/simplespringmvc/integration/ValidationIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.annotation.ControllerAdvice;
import com.simplespringmvc.annotation.ExceptionHandler;
import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.annotation.PostMapping;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.annotation.RestController;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.exception.ExceptionHandlerExceptionResolver;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import com.simplespringmvc.validation.MethodArgumentNotValidException;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.Collections;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Integration test for Validation (Feature 16) with the full handler pipeline.
 *
 * Tests that @Valid + Jakarta constraints work end-to-end:
 * handler mapping → adapter resolves @ModelAttribute → validates → exception
 * handling via @ControllerAdvice produces structured error response.
 */
class ValidationIntegrationTest {

    private SimpleHandlerMapping handlerMapping;
    private SimpleHandlerAdapter handlerAdapter;
    private ExceptionHandlerExceptionResolver exceptionResolver;
    private HttpServletResponse response;
    private StringWriter responseBody;

    // ─── Test POJO with Jakarta constraints ─────────────────────

    public static class CreateUserForm {
        @NotNull
        private String username;

        @Min(0)
        private int age;

        @Size(min = 1, max = 255)
        private String email;

        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
    }

    // ─── Test Controller ─────────────────────────────────────────

    @RestController
    public static class UserController {

        @PostMapping("/users")
        public String createUser(@Valid @ModelAttribute CreateUserForm form) {
            return "Created: " + form.getUsername();
        }

        @PostMapping("/users/no-valid")
        public String createUserNoValidation(@ModelAttribute CreateUserForm form) {
            // No @Valid — no validation runs, even if constraints exist on the form
            return "Created without validation: " + form.getUsername();
        }
    }

    // ─── Global Exception Handler ─────────────────────────────────

    @ControllerAdvice
    public static class GlobalExceptionHandler {

        @ExceptionHandler(MethodArgumentNotValidException.class)
        @ResponseBody
        public Map<String, Object> handleValidationError(MethodArgumentNotValidException ex) {
            var errors = ex.getBindingResult().getFieldErrors().stream()
                    .map(fe -> Map.of(
                            "field", fe.field(),
                            "message", fe.message(),
                            "code", fe.code()))
                    .toList();

            return Map.of(
                    "status", 400,
                    "errors", errors);
        }
    }

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new UserController());
        container.registerBean(new GlobalExceptionHandler());

        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(container);
        handlerAdapter = new SimpleHandlerAdapter();

        exceptionResolver = new ExceptionHandlerExceptionResolver();
        exceptionResolver.init(container);

        response = mock(HttpServletResponse.class);
        responseBody = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(responseBody));
    }

    // ─── Happy path: valid data ──────────────────────────────────

    @Test
    void shouldProcessRequest_WhenAllConstraintsSatisfied() throws Exception {
        HttpServletRequest request = mockRequest("POST", "/users",
                "username", "Alice", "age", "25", "email", "alice@test.com");

        HandlerMethod handler = handlerMapping.lookupHandler(request);
        assertThat(handler).isNotNull();

        ModelAndView mav = handlerAdapter.handle(request, response, handler);

        assertThat(mav).isNull(); // @ResponseBody
        assertThat(responseBody.toString()).contains("Created: Alice");
    }

    // ─── Validation failure: @Valid triggers validation ───────────

    @Test
    void shouldThrowValidationException_WhenNotNullViolated() {
        HttpServletRequest request = mockRequest("POST", "/users",
                "age", "25", "email", "test@test.com");
        // username is missing → null → @NotNull violation

        HandlerMethod handler = handlerMapping.lookupHandler(request);

        assertThatThrownBy(() -> handlerAdapter.handle(request, response, handler))
                .isInstanceOf(MethodArgumentNotValidException.class)
                .satisfies(ex -> {
                    var validEx = (MethodArgumentNotValidException) ex;
                    assertThat(validEx.getBindingResult().hasErrors()).isTrue();
                    assertThat(validEx.getBindingResult().getFieldErrors())
                            .anyMatch(fe -> fe.field().equals("username")
                                    && fe.code().equals("NotNull"));
                });
    }

    @Test
    void shouldThrowValidationException_WhenMinViolated() {
        HttpServletRequest request = mockRequest("POST", "/users",
                "username", "Bob", "age", "-5", "email", "bob@test.com");
        // age = -5 → @Min(0) violation

        HandlerMethod handler = handlerMapping.lookupHandler(request);

        assertThatThrownBy(() -> handlerAdapter.handle(request, response, handler))
                .isInstanceOf(MethodArgumentNotValidException.class)
                .satisfies(ex -> {
                    var validEx = (MethodArgumentNotValidException) ex;
                    assertThat(validEx.getBindingResult().getFieldErrors())
                            .anyMatch(fe -> fe.field().equals("age")
                                    && fe.code().equals("Min"));
                });
    }

    @Test
    void shouldThrowValidationException_WhenSizeViolated() {
        HttpServletRequest request = mockRequest("POST", "/users",
                "username", "Charlie", "age", "30", "email", "");
        // email = "" → @Size(min=1) violation

        HandlerMethod handler = handlerMapping.lookupHandler(request);

        assertThatThrownBy(() -> handlerAdapter.handle(request, response, handler))
                .isInstanceOf(MethodArgumentNotValidException.class)
                .satisfies(ex -> {
                    var validEx = (MethodArgumentNotValidException) ex;
                    assertThat(validEx.getBindingResult().getFieldErrors())
                            .anyMatch(fe -> fe.field().equals("email")
                                    && fe.code().equals("Size"));
                });
    }

    @Test
    void shouldReportMultipleErrors_WhenMultipleViolations() {
        HttpServletRequest request = mockRequest("POST", "/users",
                "age", "-1", "email", "");
        // username=null (@NotNull), age=-1 (@Min), email="" (@Size)

        HandlerMethod handler = handlerMapping.lookupHandler(request);

        assertThatThrownBy(() -> handlerAdapter.handle(request, response, handler))
                .isInstanceOf(MethodArgumentNotValidException.class)
                .satisfies(ex -> {
                    var validEx = (MethodArgumentNotValidException) ex;
                    assertThat(validEx.getBindingResult().getErrorCount())
                            .isGreaterThanOrEqualTo(3);
                });
    }

    // ─── No @Valid: validation skipped ───────────────────────────

    @Test
    void shouldNotValidate_WhenValidAnnotationAbsent() throws Exception {
        HttpServletRequest request = mockRequest("POST", "/users/no-valid",
                "age", "-5", "email", "");
        // Constraints exist on the form, but no @Valid → no validation

        HandlerMethod handler = handlerMapping.lookupHandler(request);
        assertThat(handler).isNotNull();

        ModelAndView mav = handlerAdapter.handle(request, response, handler);

        assertThat(mav).isNull();
        assertThat(responseBody.toString()).contains("Created without validation: null");
    }

    // ─── Exception handling: structured error response ───────────

    @Test
    void shouldReturnStructuredError_WhenExceptionHandlerCatchesValidation() throws Exception {
        HttpServletRequest request = mockRequest("POST", "/users",
                "age", "25", "email", "test@test.com");
        // username=null → @NotNull violation

        HandlerMethod handler = handlerMapping.lookupHandler(request);

        // Simulate what DispatcherServlet does: catch exception → resolve
        try {
            handlerAdapter.handle(request, response, handler);
            fail("Expected MethodArgumentNotValidException");
        } catch (MethodArgumentNotValidException ex) {
            // The exception resolver should find our @ControllerAdvice handler
            boolean resolved = exceptionResolver.resolveException(
                    request, response, handler, ex);

            assertThat(resolved).isTrue();
            // The response should contain the structured JSON error
            String json = responseBody.toString();
            assertThat(json).contains("\"status\"");
            assertThat(json).contains("400");
            assertThat(json).contains("\"errors\"");
            assertThat(json).contains("username");
            assertThat(json).contains("NotNull");
        }
    }

    // ─── Helpers ─────────────────────────────────────────────────

    private HttpServletRequest mockRequest(String method, String path, String... params) {
        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getMethod()).thenReturn(method);
        when(request.getRequestURI()).thenReturn(path);
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
| **`@Valid`** | Opt-in annotation that triggers Jakarta Bean Validation on a method parameter after data binding |
| **`BindingResult`** | Collects all validation errors (`FieldError`s) for a single target object |
| **`FieldError`** | One constraint violation on one field — carries field name, rejected value, message, and error code |
| **`SimpleValidatorAdapter`** | Bridges Jakarta `Validator` → `BindingResult` by converting `ConstraintViolation`s to `FieldError`s |
| **`MethodArgumentNotValidException`** | Checked exception thrown when `@Valid` fails, carrying the full `BindingResult` for structured error responses |
| **Post-binding validation** | Validation runs *after* data binding — it checks the result, not the process |

**Next: Chapter 17 — Component Scanning** — Automatically discover and register `@Controller` (and `@Component`) classes by scanning the classpath, eliminating manual bean registration.
