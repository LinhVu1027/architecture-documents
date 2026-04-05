# Chapter 11: Method Security

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// Enable method security in your configuration:
@EnableSimpleMethodSecurity
@Configuration
public class MethodSecurityConfig { }

// Then annotate your service methods:
@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) { /* ... */ }

    @PreAuthorize("hasAuthority('ORDER_READ')")
    public Order getOrder(Long id) { /* ... */ }

    @PostAuthorize("returnObject.owner == authentication.name")
    public Order findOrder(Long id) { /* ... */ }
}

// When called with the right authentication → method executes normally.
// When called without the right authorities → AccessDeniedException thrown.
// For @PostAuthorize → method executes first, then the return value is checked.
```

Questions to guide your thinking:
- How is the SpEL expression `hasRole('ADMIN')` actually evaluated? What object provides the `hasRole()` method?
- Why does `@PostAuthorize` need a different interceptor than `@PreAuthorize`? How does `returnObject` become available in the expression?
- Why does method security use Spring AOP (proxying) while URL authorization uses servlet filters?
- How does `@EnableSimpleMethodSecurity` cause Spring to create AOP proxies for your service beans?
- What is an `InfrastructureAdvisorAutoProxyCreator` and why do the advisor beans need `@Role(ROLE_INFRASTRUCTURE)`?

---

## The API Contract

Feature 11 adds method-level security — the ability to annotate individual service methods with SpEL expressions that are evaluated against the current `Authentication`. Unlike URL authorization (Feature 7) which operates at the HTTP request level via servlet filters, method security operates at the Java method level via AOP.

### The Annotations

| Annotation | When Evaluated | SpEL Variables Available | Typical Use |
|-----------|:-:|---|---|
| `@PreAuthorize` | Before method executes | `authentication` | Role/authority checks |
| `@PostAuthorize` | After method executes | `authentication`, `returnObject` | Owner-based access control |

### SpEL Methods Available

| Expression | What It Checks |
|-----------|---------------|
| `hasRole('ADMIN')` | User has `ROLE_ADMIN` authority |
| `hasAnyRole('ADMIN', 'MANAGER')` | User has any of the listed roles |
| `hasAuthority('ORDER_READ')` | User has exact authority string |
| `hasAnyAuthority('SCOPE_read', 'SCOPE_write')` | User has any of the listed authorities |
| `isAuthenticated()` | User is authenticated (non-null) |
| `permitAll()` | Always `true` |
| `denyAll()` | Always `false` |
| `authentication.name` | Access to the `Authentication` object |
| `returnObject` | The method's return value (only in `@PostAuthorize`) |

### The Behavioral Contract

1. **`@EnableSimpleMethodSecurity`** on a `@Configuration` class → registers AOP infrastructure
2. **Spring creates proxies** for beans that have `@PreAuthorize` or `@PostAuthorize` on any method
3. **Before-method interception** (`@PreAuthorize`): evaluate SpEL → if false, throw `AccessDeniedException`; method never executes
4. **After-method interception** (`@PostAuthorize`): method executes first → evaluate SpEL with `returnObject` → if false, throw `AccessDeniedException`; return value is discarded

---

## Client Usage & Tests

### Test 1: `@PreAuthorize` with hasRole — access granted

```java
@Test
void shouldAllowAccess_WhenUserHasRequiredRole() {
    authenticateAs("admin", "ROLE_ADMIN");

    assertThatCode(() -> orderService.deleteOrder(1L))
            .doesNotThrowAnyException();
}
```

### Test 2: `@PreAuthorize` with hasRole — access denied

```java
@Test
void shouldDenyAccess_WhenUserLacksRequiredRole() {
    authenticateAs("user", "ROLE_USER");

    assertThatThrownBy(() -> orderService.deleteOrder(1L))
            .isInstanceOf(AccessDeniedException.class);
}
```

### Test 3: `@PreAuthorize` with hasAuthority

```java
@Test
void shouldAllowAccess_WhenUserHasRequiredAuthority() {
    authenticateAs("reader", "ORDER_READ");

    Order result = orderService.getOrder(1L);
    assertThat(result).isNotNull();
    assertThat(result.getId()).isEqualTo(1L);
}
```

### Test 4: `@PostAuthorize` with returnObject — match

```java
@Test
void shouldAllowPostAuthorize_WhenReturnObjectMatchesAuthentication() {
    authenticateAs("admin", "ROLE_USER");

    // Method returns Order with owner="admin", authentication.name="admin" → match
    Order result = orderService.findOrder(1L, "admin");
    assertThat(result.getOwner()).isEqualTo("admin");
}
```

### Test 5: `@PostAuthorize` with returnObject — mismatch

```java
@Test
void shouldDenyPostAuthorize_WhenReturnObjectDoesNotMatchAuthentication() {
    authenticateAs("other-user", "ROLE_USER");

    // Method returns Order with owner="admin", authentication.name="other-user" → no match
    assertThatThrownBy(() -> orderService.findOrder(1L, "admin"))
            .isInstanceOf(AccessDeniedException.class);
}
```

### Test 6: Methods without annotations are not intercepted

```java
@Test
void shouldNotIntercept_WhenMethodHasNoSecurityAnnotation() {
    // No authentication needed — method has no @PreAuthorize or @PostAuthorize
    assertThatCode(() -> orderService.publicMethod())
            .doesNotThrowAnyException();
}
```

### Full test setup

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = MethodSecurityApiTest.Config.class)
class MethodSecurityApiTest {

    @EnableSimpleMethodSecurity
    @Configuration
    static class Config {
        @Bean
        OrderService orderService() {
            return new OrderService();
        }
    }

    @Autowired
    OrderService orderService;

    @AfterEach
    void clearContext() {
        SecurityContextHolder.clearContext();
    }

    private void authenticateAs(String username, String... authorities) {
        List<SimpleGrantedAuthority> grantedAuthorities = new ArrayList<>();
        for (String authority : authorities) {
            grantedAuthorities.add(new SimpleGrantedAuthority(authority));
        }
        Authentication auth = UsernamePasswordAuthenticationToken.authenticated(
                username, null, grantedAuthorities);
        SecurityContextHolder.getContext().setAuthentication(auth);
    }
}
```

Notice the test uses a **full Spring application context** — method security requires the Spring IoC container to create AOP proxies. You can't test it without a context (unlike URL authorization, which can be tested with just a filter chain).

---

## Implementing the Call Chain

### The Complete Call Chain

```
@PreAuthorize("hasRole('ADMIN')")
public void deleteOrder(Long id) { ... }

client.deleteOrder(1L)
    │
    ▼
┌─ CGLIB Proxy (created by InfrastructureAdvisorAutoProxyCreator) ──────┐
│                                                                        │
│  SimpleAuthorizationManagerBeforeMethodInterceptor.invoke(mi)          │
│    │                                                                   │
│    ├── getAuthentication() ← SecurityContextHolder.getContext()         │
│    │                                                                   │
│    ├── SimplePreAuthorizeAuthorizationManager.authorize(auth, mi)       │
│    │     │                                                             │
│    │     ├── findAnnotation(mi) → @PreAuthorize("hasRole('ADMIN')")    │
│    │     │                                                             │
│    │     ├── expressionHandler.createEvaluationContext(auth, mi)        │
│    │     │     └── new SimpleMethodSecurityExpressionRoot(auth)         │
│    │     │         └── StandardEvaluationContext(root)                  │
│    │     │                                                             │
│    │     ├── expressionHandler.parseExpression("hasRole('ADMIN')")      │
│    │     │     └── SpelExpressionParser.parseExpression(...)            │
│    │     │                                                             │
│    │     └── expression.getValue(ctx, Boolean.class)                   │
│    │           └── SpEL calls root.hasRole("ADMIN")                    │
│    │                 └── checks ROLE_ADMIN in authentication.authorities│
│    │                                                                   │
│    ├── decision.isGranted()?                                           │
│    │     ├── YES → mi.proceed() → actual method executes               │
│    │     └── NO  → throw AccessDeniedException                         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Layer 1: The Annotations (API surface)

The annotations themselves are pure metadata — no logic, just markers:

```java
// PreAuthorize.java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PreAuthorize {
    String value();  // The SpEL expression
}

// PostAuthorize.java — identical structure
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PostAuthorize {
    String value();  // SpEL expression; can reference "returnObject"
}
```

**`@Inherited`** means if you annotate a class with `@PreAuthorize`, subclasses inherit the annotation. This is standard for class-level security annotations.

**`@Target({METHOD, TYPE})`** allows placement on both methods and classes (though our simplified version only supports method-level matching via the pointcut).

### Layer 2: The SpEL Root Object (expression engine)

This is the most interesting class — it's the "bridge" between SpEL and Spring Security:

```java
// SimpleMethodSecurityExpressionRoot.java
public class SimpleMethodSecurityExpressionRoot {

    private static final String ROLE_PREFIX = "ROLE_";
    private final Supplier<Authentication> authentication;
    private Object returnObject;

    public SimpleMethodSecurityExpressionRoot(Supplier<Authentication> authentication) {
        this.authentication = authentication;
    }

    // When SpEL evaluates "authentication.name", it calls:
    public Authentication getAuthentication() {
        return this.authentication.get();
    }

    // When SpEL evaluates "returnObject.owner", it calls:
    public Object getReturnObject() { return this.returnObject; }
    public void setReturnObject(Object returnObject) { this.returnObject = returnObject; }

    // When SpEL evaluates "hasRole('ADMIN')", it calls:
    public final boolean hasRole(String role) {
        String authority = role.startsWith(ROLE_PREFIX) ? role : ROLE_PREFIX + role;
        return hasAuthority(authority);
    }

    // When SpEL evaluates "hasAuthority('ORDER_READ')", it calls:
    public final boolean hasAuthority(String authority) {
        return getGrantedAuthorityStrings().contains(authority);
    }

    // When SpEL evaluates "isAuthenticated()", it calls:
    public final boolean isAuthenticated() {
        Authentication auth = getAuthentication();
        return auth != null && auth.isAuthenticated();
    }

    public final boolean permitAll() { return true; }
    public final boolean denyAll() { return false; }

    private Set<String> getGrantedAuthorityStrings() {
        Authentication auth = getAuthentication();
        if (auth == null) return Set.of();
        return auth.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toSet());
    }
}
```

**How SpEL resolves method calls on the root object**: When the `SpelExpressionParser` parses `hasRole('ADMIN')`, it creates an AST node representing a method call. During evaluation, SpEL looks for a method named `hasRole` taking a `String` parameter on the root object of the `EvaluationContext`. Since `SimpleMethodSecurityExpressionRoot` has `hasRole(String)`, SpEL calls it directly.

**Lazy authentication**: The `Supplier<Authentication>` pattern means the `Authentication` is only fetched from `SecurityContextHolder` when the expression actually references it (e.g., calls `hasRole()` which calls `getAuthentication()`). If the expression is `permitAll()`, the authentication is never resolved.

### Layer 3: The Expression Handler (context factory)

The handler creates the SpEL evaluation context that connects the expression to the security root:

```java
// SimpleMethodSecurityExpressionHandler.java
public class SimpleMethodSecurityExpressionHandler {

    private final SpelExpressionParser parser = new SpelExpressionParser();
    private final ParameterNameDiscoverer parameterNameDiscoverer =
            new DefaultParameterNameDiscoverer();

    public Expression parseExpression(String expressionString) {
        return parser.parseExpression(expressionString);
    }

    public StandardEvaluationContext createEvaluationContext(
            Supplier<Authentication> authentication, MethodInvocation mi) {
        SimpleMethodSecurityExpressionRoot root =
                new SimpleMethodSecurityExpressionRoot(authentication);
        StandardEvaluationContext ctx = new StandardEvaluationContext(root);

        // Make method parameters available as SpEL variables (#paramName)
        String[] paramNames = parameterNameDiscoverer.getParameterNames(mi.getMethod());
        if (paramNames != null) {
            Object[] args = mi.getArguments();
            for (int i = 0; i < paramNames.length && i < args.length; i++) {
                ctx.setVariable(paramNames[i], args[i]);
            }
        }
        return ctx;
    }

    public void setReturnObject(Object returnObject, StandardEvaluationContext ctx) {
        Object root = ctx.getRootObject().getValue();
        if (root instanceof SimpleMethodSecurityExpressionRoot expressionRoot) {
            expressionRoot.setReturnObject(returnObject);
        }
    }
}
```

**Three responsibilities**: parse → context → inject. The handler is the only class that knows about SpEL classes directly.

**`StandardEvaluationContext(root)`**: The root object becomes the default target for property access and method calls in SpEL. That single constructor call is what makes `hasRole()`, `authentication`, and `returnObject` all available.

### Layer 4: The Authorization Managers (decision logic)

These managers implement the same `AuthorizationManager<T>` interface used for URL authorization, but with different generic types:

```java
// SimplePreAuthorizeAuthorizationManager.java
public class SimplePreAuthorizeAuthorizationManager
        implements AuthorizationManager<MethodInvocation> {

    private final SimpleMethodSecurityExpressionHandler expressionHandler =
            new SimpleMethodSecurityExpressionHandler();

    @Override
    public AuthorizationDecision authorize(
            Supplier<Authentication> authentication, MethodInvocation mi) {
        PreAuthorize preAuthorize = AnnotationUtils.findAnnotation(
                mi.getMethod(), PreAuthorize.class);
        if (preAuthorize == null) {
            return null;  // abstain
        }
        StandardEvaluationContext ctx =
                expressionHandler.createEvaluationContext(authentication, mi);
        Expression expression = expressionHandler.parseExpression(preAuthorize.value());
        boolean granted = Boolean.TRUE.equals(expression.getValue(ctx, Boolean.class));
        return new AuthorizationDecision(granted);
    }
}
```

The `PostAuthorize` manager has one critical difference — it calls `setReturnObject()`:

```java
// SimplePostAuthorizeAuthorizationManager.java
public class SimplePostAuthorizeAuthorizationManager
        implements AuthorizationManager<MethodInvocationResult> {

    @Override
    public AuthorizationDecision authorize(
            Supplier<Authentication> authentication, MethodInvocationResult mi) {
        PostAuthorize postAuthorize = AnnotationUtils.findAnnotation(
                mi.getMethodInvocation().getMethod(), PostAuthorize.class);
        if (postAuthorize == null) return null;

        StandardEvaluationContext ctx = expressionHandler.createEvaluationContext(
                authentication, mi.getMethodInvocation());

        // THIS IS THE KEY LINE — inject returnObject into the SpEL context
        expressionHandler.setReturnObject(mi.getResult(), ctx);

        Expression expression = expressionHandler.parseExpression(postAuthorize.value());
        boolean granted = Boolean.TRUE.equals(expression.getValue(ctx, Boolean.class));
        return new AuthorizationDecision(granted);
    }
}
```

**`AnnotationUtils.findAnnotation()`**: Unlike `method.getAnnotation()`, this Spring utility traverses the method hierarchy. When Spring AOP creates a CGLIB proxy, `mi.getMethod()` may return the proxy's overridden method, which doesn't carry the annotation directly. `AnnotationUtils.findAnnotation()` walks up to the original target method to find it.

### Layer 5: The AOP Interceptors (entry point)

The interceptors implement both `MethodInterceptor` (AOP Alliance) and `PointcutAdvisor` (Spring AOP):

```java
// SimpleAuthorizationManagerBeforeMethodInterceptor.java
public class SimpleAuthorizationManagerBeforeMethodInterceptor
        implements MethodInterceptor, PointcutAdvisor, Ordered {

    private final Pointcut pointcut;
    private final AuthorizationManager<MethodInvocation> authorizationManager;

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        Supplier<Authentication> authentication = this::getAuthentication;
        AuthorizationDecision decision =
                authorizationManager.authorize(authentication, mi);
        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }
        return mi.proceed();  // Method executes only if authorized
    }

    private Authentication getAuthentication() {
        return SecurityContextHolder.getContext().getAuthentication();
    }

    public static SimpleAuthorizationManagerBeforeMethodInterceptor preAuthorize() {
        Pointcut pointcut = new AnnotationMatchingPointcut(null, PreAuthorize.class, true);
        return new SimpleAuthorizationManagerBeforeMethodInterceptor(
                pointcut, new SimplePreAuthorizeAuthorizationManager());
    }

    // PointcutAdvisor: getPointcut(), getAdvice() returns this
    // Ordered: getOrder() returns 200
}
```

The after-method interceptor has the opposite flow:

```java
// SimpleAuthorizationManagerAfterMethodInterceptor.java — the critical difference
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    Object result = mi.proceed();   // Step 1: Method executes FIRST

    Supplier<Authentication> authentication = this::getAuthentication;
    MethodInvocationResult invocationResult = new MethodInvocationResult(mi, result);
    AuthorizationDecision decision =
            authorizationManager.authorize(authentication, invocationResult);
    if (decision != null && !decision.isGranted()) {
        throw new AccessDeniedException("Access Denied"); // Result discarded
    }
    return result;                  // Step 3: Return only if authorized
}
```

**Why two interceptors?** They have fundamentally different lifecycles:
- Before: check → if denied, method never runs (saves resources, no side effects)
- After: run → check → if denied, discard result (method already ran, side effects happened)

**`AnnotationMatchingPointcut(null, PreAuthorize.class, true)`**: This creates a pointcut where:
- First arg (`null`): no class-level annotation requirement
- Second arg (`PreAuthorize.class`): matches methods with this annotation
- Third arg (`true`): check inherited annotations

### Layer 6: The Configuration (wiring)

The enable annotation and configuration class wire everything together:

```java
// EnableSimpleMethodSecurity.java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(SimplePrePostMethodSecurityConfiguration.class)
public @interface EnableSimpleMethodSecurity { }

// SimplePrePostMethodSecurityConfiguration.java
@Configuration(proxyBeanMethods = false)
public class SimplePrePostMethodSecurityConfiguration {

    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    static InfrastructureAdvisorAutoProxyCreator
            simpleMethodSecurityAdvisorAutoProxyCreator() {
        return new InfrastructureAdvisorAutoProxyCreator();
    }

    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    static Advisor simplePreAuthorizeAuthorizationMethodInterceptor() {
        return SimpleAuthorizationManagerBeforeMethodInterceptor.preAuthorize();
    }

    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    static Advisor simplePostAuthorizeAuthorizationMethodInterceptor() {
        return SimpleAuthorizationManagerAfterMethodInterceptor.postAuthorize();
    }
}
```

**`InfrastructureAdvisorAutoProxyCreator`**: This Spring AOP class scans the context for `Advisor` beans with `@Role(ROLE_INFRASTRUCTURE)`, then creates CGLIB proxies for any beans whose methods match the advisors' pointcuts. Without it, the interceptor beans exist but nothing calls them.

**`static` bean methods**: Prevents circular dependencies. The auto-proxy creator is a `BeanPostProcessor` that must be created very early in the context lifecycle. Non-static methods could trigger premature config class initialization.

**`@Configuration(proxyBeanMethods = false)`**: Since no `@Bean` method calls another, we skip CGLIB proxying of the config class itself — a performance optimization.

---

## Try It Yourself

1. **Expression caching**: Our implementation parses the SpEL expression string on every method call. The real framework caches parsed `Expression` objects in a `ConcurrentHashMap<Method, Expression>`. Add caching to `SimplePreAuthorizeAuthorizationManager`.

2. **Class-level `@PreAuthorize`**: Add support for placing `@PreAuthorize` on a class, which applies to all methods. You'll need to modify the pointcut to include a class-level `AnnotationMatchingPointcut` and update the annotation lookup to check both method and class.

3. **`@PreFilter`/`@PostFilter`**: These annotations filter collection arguments/return values using SpEL. For example, `@PostFilter("filterObject.owner == authentication.name")` removes items from a returned list that the user doesn't own. How would you implement the iteration and removal?

4. **Custom `AuthorizationManager`**: Replace the SpEL-based manager with one that reads authorization rules from a database. What interface would you implement?

---

## Why This Works

### Same Interface, Different Domain
Feature 7 (URL authorization) and Feature 11 (method security) both use `AuthorizationManager<T>`, but with different type parameters:
- `AuthorizationManager<HttpServletRequest>` — decides based on the URL pattern
- `AuthorizationManager<MethodInvocation>` — decides based on method annotations

This is a textbook example of the **Generic Interface Pattern**: one contract, multiple domains. The `Supplier<Authentication>` is the same in both cases — only the "secured object" type changes.

### SpEL as a DSL
The expression root object (`SimpleMethodSecurityExpressionRoot`) is essentially a **Domain-Specific Language** implemented via Java methods. SpEL provides the parser and evaluator; the root object provides the vocabulary. Adding a new expression (e.g., `isOwner(#id)`) is just adding a method to the root class.

### AOP vs. Filter: Right Tool for the Right Layer
URL authorization uses **servlet filters** because HTTP requests pass through a well-defined pipeline with a clear interception point. Method security uses **AOP proxies** because there's no pipeline for arbitrary Java method calls — the only way to intercept them is to wrap the target object in a proxy. The proxy transparently intercepts calls and adds security checks.

### The "Execute First, Check After" Pattern
`@PostAuthorize` is an example of **optimistic authorization**: the method runs first (optimistically), and the result is checked afterward. This is useful when the authorization decision depends on the method's output (e.g., "does the returned order belong to the current user?"). The tradeoff is that side effects from the method execution cannot be rolled back if authorization fails.

---

## What We Enhanced

| Component | Previous State | Enhancement in Feature 11 |
|-----------|---------------|--------------------------|
| `simple-security-core/build.gradle` | Depended only on `simple-security-crypto` | Added `spring-aop` and `spring-expression` dependencies for AOP interceptors and SpEL evaluation |

Feature 11 is unique among the features because it does not modify any existing source files. It introduces an entirely new subsystem (AOP-based method security) that is orthogonal to the filter-based web security. The only change to existing code is the build dependency addition.

---

## Connection to Real Framework

| Simplified Class | Real Framework Class | File |
|------------------|----------------------|------|
| `@PreAuthorize` | `@PreAuthorize` | `core/src/main/java/.../access/prepost/PreAuthorize.java` |
| `@PostAuthorize` | `@PostAuthorize` | `core/src/main/java/.../access/prepost/PostAuthorize.java` |
| `SimpleMethodSecurityExpressionRoot` | `SecurityExpressionRoot` + `MethodSecurityExpressionRoot` | `core/src/main/java/.../access/expression/SecurityExpressionRoot.java` |
| `SimpleMethodSecurityExpressionHandler` | `DefaultMethodSecurityExpressionHandler` | `core/src/main/java/.../access/expression/method/DefaultMethodSecurityExpressionHandler.java` |
| `SimplePreAuthorizeAuthorizationManager` | `PreAuthorizeAuthorizationManager` | `core/src/main/java/.../authorization/method/PreAuthorizeAuthorizationManager.java` |
| `SimplePostAuthorizeAuthorizationManager` | `PostAuthorizeAuthorizationManager` | `core/src/main/java/.../authorization/method/PostAuthorizeAuthorizationManager.java` |
| `SimpleAuthorizationManagerBeforeMethodInterceptor` | `AuthorizationManagerBeforeMethodInterceptor` | `core/src/main/java/.../authorization/method/AuthorizationManagerBeforeMethodInterceptor.java` |
| `SimpleAuthorizationManagerAfterMethodInterceptor` | `AuthorizationManagerAfterMethodInterceptor` | `core/src/main/java/.../authorization/method/AuthorizationManagerAfterMethodInterceptor.java` |
| `MethodInvocationResult` | `MethodInvocationResult` | `core/src/main/java/.../authorization/method/MethodInvocationResult.java` |
| `@EnableSimpleMethodSecurity` | `@EnableMethodSecurity` | `config/src/main/java/.../method/configuration/EnableMethodSecurity.java` |
| `SimplePrePostMethodSecurityConfiguration` | `PrePostMethodSecurityConfiguration` | `config/src/main/java/.../method/configuration/PrePostMethodSecurityConfiguration.java` |

### What the real framework adds

- **`MethodSecuritySelector`**: An `ImportSelector` that conditionally imports different configuration classes based on `@EnableMethodSecurity` attributes (`prePostEnabled`, `securedEnabled`, `jsr250Enabled`)
- **`@Secured` and JSR-250 (`@RolesAllowed`)**: Two additional annotation systems beyond `@PreAuthorize`/`@PostAuthorize`
- **`@PreFilter` / `@PostFilter`**: Annotations that filter collection arguments/return values element-by-element using SpEL
- **`AbstractExpressionAttributeRegistry`**: A `ConcurrentHashMap`-based cache that parses and caches SpEL expressions per method — our version re-parses on every call
- **`MethodSecurityEvaluationContext`**: Extends `MethodBasedEvaluationContext` to make method parameter names available as SpEL variables via bytecode analysis
- **`SecurityExpressionRoot` hierarchy**: Separate base class with `PermissionEvaluator`, `RoleHierarchy`, and `AuthorizationManagerFactory` integration
- **`MethodAuthorizationDeniedHandler`**: Customizable handling when access is denied (not just throwing an exception — can return a mask value)
- **`DeferringMethodInterceptor`**: Wraps the real interceptor for lazy initialization, avoiding circular bean dependencies during context startup
- **`AuthorizationEventPublisher`**: Publishes Spring application events for authorization decisions (for audit logging)
- **Template expressions**: Support for `@PreAuthorize("@bean.check({id})")` meta-annotation templates with placeholder resolution
- **`AnnotationTemplateExpressionDefaults`**: Configuration for expression template syntax
- **`AuthorizationAdvisor`**: A marker interface extending both `PointcutAdvisor` and `Ordered` for cleaner type hierarchy

---

## Complete Code

### Modified Files

#### `[MODIFIED] simple-security-core/build.gradle`

```groovy
// simple-security-core — Authentication model, SecurityContext, UserDetails, AuthorizationManager
// Mirrors: spring-security-core

dependencies {
    api project(':simple-security-crypto')
    implementation 'org.springframework:spring-aop:6.2.3'
    implementation 'org.springframework:spring-expression:6.2.3'
}
```

### New Files

#### `[NEW] simple-security-core/src/main/java/simple/security/access/prepost/PreAuthorize.java`

```java
package simple.security.access.prepost;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PreAuthorize {

	String value();

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/access/prepost/PostAuthorize.java`

```java
package simple.security.access.prepost;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PostAuthorize {

	String value();

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/SimpleMethodSecurityExpressionRoot.java`

```java
package simple.security.authorization.method;

import java.util.Collection;
import java.util.Set;
import java.util.function.Supplier;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import simple.security.core.Authentication;
import simple.security.core.GrantedAuthority;

public class SimpleMethodSecurityExpressionRoot {

	private static final String ROLE_PREFIX = "ROLE_";

	private final Supplier<Authentication> authentication;

	private Object returnObject;

	public SimpleMethodSecurityExpressionRoot(Supplier<Authentication> authentication) {
		this.authentication = authentication;
	}

	public Authentication getAuthentication() {
		return this.authentication.get();
	}

	public Object getReturnObject() {
		return this.returnObject;
	}

	public void setReturnObject(Object returnObject) {
		this.returnObject = returnObject;
	}

	public final boolean hasRole(String role) {
		String authority = role.startsWith(ROLE_PREFIX) ? role : ROLE_PREFIX + role;
		return hasAuthority(authority);
	}

	public final boolean hasAnyRole(String... roles) {
		String[] authorities = Stream.of(roles)
				.map(role -> role.startsWith(ROLE_PREFIX) ? role : ROLE_PREFIX + role)
				.toArray(String[]::new);
		return hasAnyAuthority(authorities);
	}

	public final boolean hasAuthority(String authority) {
		return getGrantedAuthorityStrings().contains(authority);
	}

	public final boolean hasAnyAuthority(String... authorities) {
		Set<String> required = Set.of(authorities);
		Set<String> userAuthorities = getGrantedAuthorityStrings();
		for (String r : required) {
			if (userAuthorities.contains(r)) {
				return true;
			}
		}
		return false;
	}

	public final boolean isAuthenticated() {
		Authentication auth = getAuthentication();
		return auth != null && auth.isAuthenticated();
	}

	public final boolean permitAll() {
		return true;
	}

	public final boolean denyAll() {
		return false;
	}

	private Set<String> getGrantedAuthorityStrings() {
		Authentication auth = getAuthentication();
		if (auth == null) {
			return Set.of();
		}
		Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
		if (authorities == null) {
			return Set.of();
		}
		return authorities.stream()
				.map(GrantedAuthority::getAuthority)
				.collect(Collectors.toSet());
	}

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/SimpleMethodSecurityExpressionHandler.java`

```java
package simple.security.authorization.method;

import java.util.function.Supplier;

import org.aopalliance.intercept.MethodInvocation;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.core.ParameterNameDiscoverer;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;

import simple.security.core.Authentication;

public class SimpleMethodSecurityExpressionHandler {

	private final SpelExpressionParser parser = new SpelExpressionParser();

	private final ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

	public Expression parseExpression(String expressionString) {
		return this.parser.parseExpression(expressionString);
	}

	public StandardEvaluationContext createEvaluationContext(
			Supplier<Authentication> authentication, MethodInvocation mi) {
		SimpleMethodSecurityExpressionRoot root = new SimpleMethodSecurityExpressionRoot(authentication);
		StandardEvaluationContext ctx = new StandardEvaluationContext(root);

		String[] paramNames = this.parameterNameDiscoverer.getParameterNames(mi.getMethod());
		if (paramNames != null) {
			Object[] args = mi.getArguments();
			for (int i = 0; i < paramNames.length && i < args.length; i++) {
				ctx.setVariable(paramNames[i], args[i]);
			}
		}

		return ctx;
	}

	public void setReturnObject(Object returnObject, StandardEvaluationContext ctx) {
		Object root = ctx.getRootObject().getValue();
		if (root instanceof SimpleMethodSecurityExpressionRoot expressionRoot) {
			expressionRoot.setReturnObject(returnObject);
		}
	}

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/MethodInvocationResult.java`

```java
package simple.security.authorization.method;

import org.aopalliance.intercept.MethodInvocation;

public class MethodInvocationResult {

	private final MethodInvocation methodInvocation;

	private final Object result;

	public MethodInvocationResult(MethodInvocation methodInvocation, Object result) {
		this.methodInvocation = methodInvocation;
		this.result = result;
	}

	public MethodInvocation getMethodInvocation() {
		return this.methodInvocation;
	}

	public Object getResult() {
		return this.result;
	}

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/SimplePreAuthorizeAuthorizationManager.java`

```java
package simple.security.authorization.method;

import java.util.function.Supplier;

import org.aopalliance.intercept.MethodInvocation;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.support.StandardEvaluationContext;

import simple.security.access.prepost.PreAuthorize;
import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.core.Authentication;

public class SimplePreAuthorizeAuthorizationManager implements AuthorizationManager<MethodInvocation> {

	private final SimpleMethodSecurityExpressionHandler expressionHandler =
			new SimpleMethodSecurityExpressionHandler();

	@Override
	public AuthorizationDecision authorize(Supplier<Authentication> authentication, MethodInvocation mi) {
		PreAuthorize preAuthorize = findAnnotation(mi);
		if (preAuthorize == null) {
			return null;
		}

		StandardEvaluationContext ctx = this.expressionHandler.createEvaluationContext(authentication, mi);
		Expression expression = this.expressionHandler.parseExpression(preAuthorize.value());
		boolean granted = Boolean.TRUE.equals(expression.getValue(ctx, Boolean.class));
		return new AuthorizationDecision(granted);
	}

	private PreAuthorize findAnnotation(MethodInvocation mi) {
		return AnnotationUtils.findAnnotation(mi.getMethod(), PreAuthorize.class);
	}

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/SimplePostAuthorizeAuthorizationManager.java`

```java
package simple.security.authorization.method;

import java.util.function.Supplier;

import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.support.StandardEvaluationContext;

import simple.security.access.prepost.PostAuthorize;
import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.core.Authentication;

public class SimplePostAuthorizeAuthorizationManager implements AuthorizationManager<MethodInvocationResult> {

	private final SimpleMethodSecurityExpressionHandler expressionHandler =
			new SimpleMethodSecurityExpressionHandler();

	@Override
	public AuthorizationDecision authorize(Supplier<Authentication> authentication,
			MethodInvocationResult mi) {
		PostAuthorize postAuthorize = findAnnotation(mi);
		if (postAuthorize == null) {
			return null;
		}

		StandardEvaluationContext ctx =
				this.expressionHandler.createEvaluationContext(authentication, mi.getMethodInvocation());
		this.expressionHandler.setReturnObject(mi.getResult(), ctx);

		Expression expression = this.expressionHandler.parseExpression(postAuthorize.value());
		boolean granted = Boolean.TRUE.equals(expression.getValue(ctx, Boolean.class));
		return new AuthorizationDecision(granted);
	}

	private PostAuthorize findAnnotation(MethodInvocationResult mi) {
		return AnnotationUtils.findAnnotation(
				mi.getMethodInvocation().getMethod(), PostAuthorize.class);
	}

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/SimpleAuthorizationManagerBeforeMethodInterceptor.java`

```java
package simple.security.authorization.method;

import java.util.function.Supplier;

import org.aopalliance.aop.Advice;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.Pointcut;
import org.springframework.aop.PointcutAdvisor;
import org.springframework.aop.support.annotation.AnnotationMatchingPointcut;
import org.springframework.core.Ordered;

import simple.security.access.AccessDeniedException;
import simple.security.access.prepost.PreAuthorize;
import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.core.Authentication;
import simple.security.core.context.SecurityContextHolder;

public class SimpleAuthorizationManagerBeforeMethodInterceptor
		implements MethodInterceptor, PointcutAdvisor, Ordered {

	private final Pointcut pointcut;

	private final AuthorizationManager<MethodInvocation> authorizationManager;

	private int order = 200;

	public SimpleAuthorizationManagerBeforeMethodInterceptor(
			Pointcut pointcut, AuthorizationManager<MethodInvocation> authorizationManager) {
		this.pointcut = pointcut;
		this.authorizationManager = authorizationManager;
	}

	public static SimpleAuthorizationManagerBeforeMethodInterceptor preAuthorize() {
		return preAuthorize(new SimplePreAuthorizeAuthorizationManager());
	}

	public static SimpleAuthorizationManagerBeforeMethodInterceptor preAuthorize(
			AuthorizationManager<MethodInvocation> authorizationManager) {
		Pointcut pointcut = new AnnotationMatchingPointcut(null, PreAuthorize.class, true);
		return new SimpleAuthorizationManagerBeforeMethodInterceptor(pointcut, authorizationManager);
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Supplier<Authentication> authentication = this::getAuthentication;
		AuthorizationDecision decision = this.authorizationManager.authorize(authentication, mi);
		if (decision != null && !decision.isGranted()) {
			throw new AccessDeniedException("Access Denied");
		}
		return mi.proceed();
	}

	private Authentication getAuthentication() {
		return SecurityContextHolder.getContext().getAuthentication();
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

	@Override
	public Advice getAdvice() {
		return this;
	}

	@Override
	public boolean isPerInstance() {
		return false;
	}

	@Override
	public int getOrder() {
		return this.order;
	}

	public void setOrder(int order) {
		this.order = order;
	}

}
```

#### `[NEW] simple-security-core/src/main/java/simple/security/authorization/method/SimpleAuthorizationManagerAfterMethodInterceptor.java`

```java
package simple.security.authorization.method;

import java.util.function.Supplier;

import org.aopalliance.aop.Advice;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.Pointcut;
import org.springframework.aop.PointcutAdvisor;
import org.springframework.aop.support.annotation.AnnotationMatchingPointcut;
import org.springframework.core.Ordered;

import simple.security.access.AccessDeniedException;
import simple.security.access.prepost.PostAuthorize;
import simple.security.authorization.AuthorizationDecision;
import simple.security.authorization.AuthorizationManager;
import simple.security.core.Authentication;
import simple.security.core.context.SecurityContextHolder;

public class SimpleAuthorizationManagerAfterMethodInterceptor
		implements MethodInterceptor, PointcutAdvisor, Ordered {

	private final Pointcut pointcut;

	private final AuthorizationManager<MethodInvocationResult> authorizationManager;

	private int order = 500;

	public SimpleAuthorizationManagerAfterMethodInterceptor(
			Pointcut pointcut, AuthorizationManager<MethodInvocationResult> authorizationManager) {
		this.pointcut = pointcut;
		this.authorizationManager = authorizationManager;
	}

	public static SimpleAuthorizationManagerAfterMethodInterceptor postAuthorize() {
		return postAuthorize(new SimplePostAuthorizeAuthorizationManager());
	}

	public static SimpleAuthorizationManagerAfterMethodInterceptor postAuthorize(
			AuthorizationManager<MethodInvocationResult> authorizationManager) {
		Pointcut pointcut = new AnnotationMatchingPointcut(null, PostAuthorize.class, true);
		return new SimpleAuthorizationManagerAfterMethodInterceptor(pointcut, authorizationManager);
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object result = mi.proceed();

		Supplier<Authentication> authentication = this::getAuthentication;
		MethodInvocationResult invocationResult = new MethodInvocationResult(mi, result);
		AuthorizationDecision decision = this.authorizationManager.authorize(authentication, invocationResult);
		if (decision != null && !decision.isGranted()) {
			throw new AccessDeniedException("Access Denied");
		}

		return result;
	}

	private Authentication getAuthentication() {
		return SecurityContextHolder.getContext().getAuthentication();
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

	@Override
	public Advice getAdvice() {
		return this;
	}

	@Override
	public boolean isPerInstance() {
		return false;
	}

	@Override
	public int getOrder() {
		return this.order;
	}

	public void setOrder(int order) {
		this.order = order;
	}

}
```

#### `[NEW] simple-security-config/src/main/java/simple/security/config/annotation/method/configuration/EnableSimpleMethodSecurity.java`

```java
package simple.security.config.annotation.method.configuration;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.context.annotation.Import;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(SimplePrePostMethodSecurityConfiguration.class)
public @interface EnableSimpleMethodSecurity {

}
```

#### `[NEW] simple-security-config/src/main/java/simple/security/config/annotation/method/configuration/SimplePrePostMethodSecurityConfiguration.java`

```java
package simple.security.config.annotation.method.configuration;

import org.springframework.aop.Advisor;
import org.springframework.aop.framework.autoproxy.InfrastructureAdvisorAutoProxyCreator;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Role;

import simple.security.authorization.method.SimpleAuthorizationManagerAfterMethodInterceptor;
import simple.security.authorization.method.SimpleAuthorizationManagerBeforeMethodInterceptor;

@Configuration(proxyBeanMethods = false)
public class SimplePrePostMethodSecurityConfiguration {

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	static InfrastructureAdvisorAutoProxyCreator simpleMethodSecurityAdvisorAutoProxyCreator() {
		return new InfrastructureAdvisorAutoProxyCreator();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	static Advisor simplePreAuthorizeAuthorizationMethodInterceptor() {
		return SimpleAuthorizationManagerBeforeMethodInterceptor.preAuthorize();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	static Advisor simplePostAuthorizeAuthorizationMethodInterceptor() {
		return SimpleAuthorizationManagerAfterMethodInterceptor.postAuthorize();
	}

}
```

### Test Files

#### `[NEW] simple-security-config/src/test/java/simple/security/config/MethodSecurityApiTest.java`

```java
package simple.security.config;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import simple.security.access.AccessDeniedException;
import simple.security.access.prepost.PostAuthorize;
import simple.security.access.prepost.PreAuthorize;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.config.annotation.method.configuration.EnableSimpleMethodSecurity;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;
import simple.security.core.context.SecurityContextHolder;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = MethodSecurityApiTest.Config.class)
class MethodSecurityApiTest {

	@EnableSimpleMethodSecurity
	@Configuration
	static class Config {

		@Bean
		OrderService orderService() {
			return new OrderService();
		}
	}

	static class Order {

		private final Long id;
		private final String owner;

		Order(Long id, String owner) {
			this.id = id;
			this.owner = owner;
		}

		public Long getId() { return this.id; }
		public String getOwner() { return this.owner; }
	}

	static class OrderService {

		@PreAuthorize("hasRole('ADMIN')")
		public void deleteOrder(Long id) { }

		@PreAuthorize("hasAuthority('ORDER_READ')")
		public Order getOrder(Long id) {
			return new Order(id, "admin");
		}

		@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
		public void approveOrder(Long id) { }

		@PreAuthorize("isAuthenticated()")
		public List<Order> listOrders() {
			return List.of(new Order(1L, "admin"), new Order(2L, "user"));
		}

		@PostAuthorize("returnObject.owner == authentication.name")
		public Order findOrder(Long id, String owner) {
			return new Order(id, owner);
		}

		public void publicMethod() { }
	}

	@Autowired
	OrderService orderService;

	@AfterEach
	void clearContext() {
		SecurityContextHolder.clearContext();
	}

	@Test
	void shouldAllowAccess_WhenUserHasRequiredRole() {
		authenticateAs("admin", "ROLE_ADMIN");
		assertThatCode(() -> orderService.deleteOrder(1L))
				.doesNotThrowAnyException();
	}

	@Test
	void shouldDenyAccess_WhenUserLacksRequiredRole() {
		authenticateAs("user", "ROLE_USER");
		assertThatThrownBy(() -> orderService.deleteOrder(1L))
				.isInstanceOf(AccessDeniedException.class);
	}

	@Test
	void shouldAllowAccess_WhenUserHasRequiredAuthority() {
		authenticateAs("reader", "ORDER_READ");
		Order result = orderService.getOrder(1L);
		assertThat(result).isNotNull();
		assertThat(result.getId()).isEqualTo(1L);
	}

	@Test
	void shouldDenyAccess_WhenUserLacksRequiredAuthority() {
		authenticateAs("user", "ROLE_USER");
		assertThatThrownBy(() -> orderService.getOrder(1L))
				.isInstanceOf(AccessDeniedException.class);
	}

	@Test
	void shouldAllowAccess_WhenUserHasAnyOfRequiredRoles() {
		authenticateAs("manager", "ROLE_MANAGER");
		assertThatCode(() -> orderService.approveOrder(1L))
				.doesNotThrowAnyException();
	}

	@Test
	void shouldDenyAccess_WhenUserHasNoneOfRequiredRoles() {
		authenticateAs("user", "ROLE_USER");
		assertThatThrownBy(() -> orderService.approveOrder(1L))
				.isInstanceOf(AccessDeniedException.class);
	}

	@Test
	void shouldAllowAccess_WhenUserIsAuthenticated() {
		authenticateAs("user", "ROLE_USER");
		List<Order> orders = orderService.listOrders();
		assertThat(orders).hasSize(2);
	}

	@Test
	void shouldDenyAccess_WhenUserIsNotAuthenticated() {
		assertThatThrownBy(() -> orderService.listOrders())
				.isInstanceOf(AccessDeniedException.class);
	}

	@Test
	void shouldAllowPostAuthorize_WhenReturnObjectMatchesAuthentication() {
		authenticateAs("admin", "ROLE_USER");
		Order result = orderService.findOrder(1L, "admin");
		assertThat(result.getOwner()).isEqualTo("admin");
	}

	@Test
	void shouldDenyPostAuthorize_WhenReturnObjectDoesNotMatchAuthentication() {
		authenticateAs("other-user", "ROLE_USER");
		assertThatThrownBy(() -> orderService.findOrder(1L, "admin"))
				.isInstanceOf(AccessDeniedException.class);
	}

	@Test
	void shouldNotIntercept_WhenMethodHasNoSecurityAnnotation() {
		assertThatCode(() -> orderService.publicMethod())
				.doesNotThrowAnyException();
	}

	@Test
	void shouldEnforceBothPreAndPostAuthorize_AcrossServiceMethods() {
		authenticateAs("admin", "ROLE_ADMIN");
		assertThatCode(() -> orderService.deleteOrder(1L))
				.doesNotThrowAnyException();
		Order order = orderService.findOrder(1L, "admin");
		assertThat(order.getOwner()).isEqualTo("admin");
	}

	private void authenticateAs(String username, String... authorities) {
		List<SimpleGrantedAuthority> grantedAuthorities = new java.util.ArrayList<>();
		for (String authority : authorities) {
			grantedAuthorities.add(new SimpleGrantedAuthority(authority));
		}
		Authentication auth = UsernamePasswordAuthenticationToken.authenticated(
				username, null, grantedAuthorities);
		SecurityContextHolder.getContext().setAuthentication(auth);
	}

}
```

#### `[NEW] simple-security-core/src/test/java/simple/security/authorization/method/SimpleMethodSecurityExpressionRootTest.java`

```java
package simple.security.authorization.method;

import java.util.List;

import org.junit.jupiter.api.Test;

import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleMethodSecurityExpressionRootTest {

	@Test
	void shouldReturnTrue_WhenUserHasRole() {
		SimpleMethodSecurityExpressionRoot root = rootWith("admin", "ROLE_ADMIN");
		assertThat(root.hasRole("ADMIN")).isTrue();
	}

	@Test
	void shouldReturnFalse_WhenUserLacksRole() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		assertThat(root.hasRole("ADMIN")).isFalse();
	}

	@Test
	void shouldHandleRolePrefixGracefully() {
		SimpleMethodSecurityExpressionRoot root = rootWith("admin", "ROLE_ADMIN");
		assertThat(root.hasRole("ROLE_ADMIN")).isTrue();
	}

	@Test
	void shouldReturnTrue_WhenUserHasAnyOfTheRoles() {
		SimpleMethodSecurityExpressionRoot root = rootWith("manager", "ROLE_MANAGER");
		assertThat(root.hasAnyRole("ADMIN", "MANAGER")).isTrue();
	}

	@Test
	void shouldReturnFalse_WhenUserHasNoneOfTheRoles() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		assertThat(root.hasAnyRole("ADMIN", "MANAGER")).isFalse();
	}

	@Test
	void shouldReturnTrue_WhenUserHasAuthority() {
		SimpleMethodSecurityExpressionRoot root = rootWith("reader", "ORDER_READ");
		assertThat(root.hasAuthority("ORDER_READ")).isTrue();
	}

	@Test
	void shouldReturnFalse_WhenUserLacksAuthority() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		assertThat(root.hasAuthority("ORDER_READ")).isFalse();
	}

	@Test
	void shouldReturnTrue_WhenUserHasAnyOfTheAuthorities() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "SCOPE_read", "ROLE_USER");
		assertThat(root.hasAnyAuthority("SCOPE_read", "SCOPE_write")).isTrue();
	}

	@Test
	void shouldReturnFalse_WhenUserHasNoneOfTheAuthorities() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		assertThat(root.hasAnyAuthority("SCOPE_read", "SCOPE_write")).isFalse();
	}

	@Test
	void shouldReturnTrue_WhenAuthenticated() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		assertThat(root.isAuthenticated()).isTrue();
	}

	@Test
	void shouldReturnFalse_WhenNotAuthenticated() {
		SimpleMethodSecurityExpressionRoot root = new SimpleMethodSecurityExpressionRoot(() -> null);
		assertThat(root.isAuthenticated()).isFalse();
	}

	@Test
	void shouldAlwaysReturnTrue_ForPermitAll() {
		SimpleMethodSecurityExpressionRoot root = new SimpleMethodSecurityExpressionRoot(() -> null);
		assertThat(root.permitAll()).isTrue();
	}

	@Test
	void shouldAlwaysReturnFalse_ForDenyAll() {
		SimpleMethodSecurityExpressionRoot root = new SimpleMethodSecurityExpressionRoot(() -> null);
		assertThat(root.denyAll()).isFalse();
	}

	@Test
	void shouldReturnReturnObject_WhenSet() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		root.setReturnObject("test-result");
		assertThat(root.getReturnObject()).isEqualTo("test-result");
	}

	@Test
	void shouldReturnNull_WhenReturnObjectNotSet() {
		SimpleMethodSecurityExpressionRoot root = rootWith("user", "ROLE_USER");
		assertThat(root.getReturnObject()).isNull();
	}

	@Test
	void shouldReturnAuthentication() {
		SimpleMethodSecurityExpressionRoot root = rootWith("admin", "ROLE_ADMIN");
		assertThat(root.getAuthentication()).isNotNull();
		assertThat(root.getAuthentication().getName()).isEqualTo("admin");
	}

	private SimpleMethodSecurityExpressionRoot rootWith(String username, String... authorities) {
		List<SimpleGrantedAuthority> grantedAuthorities = new java.util.ArrayList<>();
		for (String authority : authorities) {
			grantedAuthorities.add(new SimpleGrantedAuthority(authority));
		}
		Authentication auth = UsernamePasswordAuthenticationToken.authenticated(
				username, null, grantedAuthorities);
		return new SimpleMethodSecurityExpressionRoot(() -> auth);
	}

}
```

#### `[NEW] simple-security-core/src/test/java/simple/security/authorization/method/SimplePreAuthorizeAuthorizationManagerTest.java`

```java
package simple.security.authorization.method;

import java.lang.reflect.Method;
import java.util.List;

import org.aopalliance.intercept.MethodInvocation;
import org.junit.jupiter.api.Test;

import simple.security.access.prepost.PreAuthorize;
import simple.security.authentication.UsernamePasswordAuthenticationToken;
import simple.security.authorization.AuthorizationDecision;
import simple.security.core.Authentication;
import simple.security.core.authority.SimpleGrantedAuthority;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.mock;

class SimplePreAuthorizeAuthorizationManagerTest {

	private final SimplePreAuthorizeAuthorizationManager manager =
			new SimplePreAuthorizeAuthorizationManager();

	@SuppressWarnings("unused")
	static class TestService {

		@PreAuthorize("hasRole('ADMIN')")
		public void adminOnly() { }

		@PreAuthorize("hasAuthority('ORDER_READ')")
		public void readOrders() { }

		@PreAuthorize("isAuthenticated()")
		public void authenticatedOnly() { }

		@PreAuthorize("denyAll()")
		public void neverAllowed() { }

		@PreAuthorize("permitAll()")
		public void alwaysAllowed() { }

		public void noAnnotation() { }
	}

	@Test
	void shouldGrantAccess_WhenUserHasRequiredRole() throws Exception {
		MethodInvocation mi = mockInvocation("adminOnly");
		Authentication auth = authenticatedWith("ROLE_ADMIN");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isTrue();
	}

	@Test
	void shouldDenyAccess_WhenUserLacksRequiredRole() throws Exception {
		MethodInvocation mi = mockInvocation("adminOnly");
		Authentication auth = authenticatedWith("ROLE_USER");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isFalse();
	}

	@Test
	void shouldGrantAccess_WhenUserHasRequiredAuthority() throws Exception {
		MethodInvocation mi = mockInvocation("readOrders");
		Authentication auth = authenticatedWith("ORDER_READ");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isTrue();
	}

	@Test
	void shouldDenyAccess_WhenUserLacksRequiredAuthority() throws Exception {
		MethodInvocation mi = mockInvocation("readOrders");
		Authentication auth = authenticatedWith("ROLE_USER");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isFalse();
	}

	@Test
	void shouldGrantAccess_WhenUserIsAuthenticated() throws Exception {
		MethodInvocation mi = mockInvocation("authenticatedOnly");
		Authentication auth = authenticatedWith("ROLE_USER");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isTrue();
	}

	@Test
	void shouldDenyAccess_ForDenyAllExpression() throws Exception {
		MethodInvocation mi = mockInvocation("neverAllowed");
		Authentication auth = authenticatedWith("ROLE_ADMIN");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isFalse();
	}

	@Test
	void shouldGrantAccess_ForPermitAllExpression() throws Exception {
		MethodInvocation mi = mockInvocation("alwaysAllowed");
		Authentication auth = authenticatedWith("ROLE_USER");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNotNull();
		assertThat(decision.isGranted()).isTrue();
	}

	@Test
	void shouldAbstain_WhenMethodHasNoAnnotation() throws Exception {
		MethodInvocation mi = mockInvocation("noAnnotation");
		Authentication auth = authenticatedWith("ROLE_USER");
		AuthorizationDecision decision = manager.authorize(() -> auth, mi);
		assertThat(decision).isNull();
	}

	private MethodInvocation mockInvocation(String methodName) throws Exception {
		Method method = TestService.class.getMethod(methodName);
		MethodInvocation mi = mock(MethodInvocation.class);
		given(mi.getMethod()).willReturn(method);
		given(mi.getArguments()).willReturn(new Object[0]);
		return mi;
	}

	private Authentication authenticatedWith(String... authorities) {
		List<SimpleGrantedAuthority> grantedAuthorities = new java.util.ArrayList<>();
		for (String authority : authorities) {
			grantedAuthorities.add(new SimpleGrantedAuthority(authority));
		}
		return UsernamePasswordAuthenticationToken.authenticated(
				"testuser", null, grantedAuthorities);
	}

}
```
