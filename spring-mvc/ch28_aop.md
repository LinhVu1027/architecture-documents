# Chapter 28: AOP / Proxy-Based Features

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| The bean container (ch24) creates beans with constructor injection, @Autowired, @Value, and BeanPostProcessor hooks -- but every bean is the raw object, with no way to add cross-cutting behavior without modifying business code | Adding logging, timing, or transaction management to multiple services requires modifying each one -- violating the Open/Closed Principle | Implement AOP using JDK dynamic proxies and CGLIB-style subclass proxies, with @Aspect/@Before/@After/@Around annotation support that automatically wraps beans with proxy objects during container startup |

---

## 28.1 The Integration Point

The integration point is **`SimpleBeanContainer.createBean()`** -- specifically lines 332-336 where `postProcessAfterInitialization()` is called. This is where AOP proxies get created.

**Before (ch24):**

```java
// Phase 3: BeanPostProcessor.postProcessAfterInitialization()
// This is where AOP proxies would be created (future ch28)
for (BeanPostProcessor bpp : beanPostProcessors) {
    instance = bpp.postProcessAfterInitialization(instance, beanName);
}
```

The ch24 implementation already had this loop with a comment indicating the AOP hook. Every bean passes through, but no BeanPostProcessor was replacing beans with proxies. The loop was a no-op placeholder.

**After (ch28):**

The `AspectJAutoProxyCreator` implements `BeanPostProcessor` and in `postProcessAfterInitialization()` wraps matching beans with proxies. The container was already designed for this -- we just need to register the right BPP.

Three changes to `SimpleBeanContainer`:

1. **`autoRegisterAopProcessorIfNeeded()`** -- called during `refresh()`, detects `@EnableAspectJAutoProxy` on any bean definition or singleton, and registers the `AspectJAutoProxyCreator` BPP:

```java
// In refresh(), after registering AutowiredAnnotationBeanPostProcessor:
autoRegisterAopProcessorIfNeeded();
```

```java
private void autoRegisterAopProcessorIfNeeded() {
    EnableAspectJAutoProxy enableAop = null;

    // Check bean definition classes
    for (var def : beanDefinitions.values()) {
        enableAop = def.getBeanClass().getAnnotation(EnableAspectJAutoProxy.class);
        if (enableAop != null) break;
    }

    // Check existing singletons
    if (enableAop == null) {
        for (Object bean : singletonMap.values()) {
            enableAop = bean.getClass().getAnnotation(EnableAspectJAutoProxy.class);
            if (enableAop != null) break;
        }
    }

    if (enableAop != null) {
        boolean hasAopProcessor = beanPostProcessors.stream()
                .anyMatch(bpp -> bpp instanceof AspectJAutoProxyCreator);
        if (!hasAopProcessor) {
            AspectJAutoProxyCreator creator = new AspectJAutoProxyCreator(this);
            creator.setProxyTargetClass(enableAop.proxyTargetClass());
            beanPostProcessors.add(creator);
        }
    }
}
```

2. **`getType(String beanName)`** -- returns the bean's class without forcing instantiation. Critical for AOP: the auto-proxy creator needs to check whether beans are `@Aspect` or `Advisor` types without triggering creation of every bean in the container:

```java
public Class<?> getType(String beanName) {
    Object bean = singletonMap.get(beanName);
    if (bean != null) {
        return bean.getClass();
    }
    BeanDefinition definition = beanDefinitions.get(beanName);
    if (definition != null) {
        return definition.getBeanClass();
    }
    return null;
}
```

3. **Line 338-339** -- the existing `singletonMap.put(beanName, instance)` after the BPP loop ensures that when a BPP replaces the bean with a proxy, the proxy (not the original) is stored.

**Direction:** The container already has the hooks. To make AOP work, we need:
- Core AOP abstractions (Advice, Pointcut, Advisor, MethodInterceptor, MethodInvocation)
- An interceptor chain engine (ReflectiveMethodInvocation)
- Two proxy strategies (JdkDynamicAopProxy, CglibAopProxy)
- A factory to choose between them (ProxyFactory)
- AspectJ annotations (@Aspect, @Before, @After, @Around, @AfterReturning, @AfterThrowing)
- The auto-proxy creator BPP that ties it all together (AspectJAutoProxyCreator)

## 28.2 Core AOP Abstractions

Five interfaces and four implementations form the foundation of the AOP infrastructure.

### The Advice Hierarchy

```
Advice (tag interface)
  |
  MethodInterceptor (around advice -- the core abstraction)
    |
    [all @Before/@After/@Around are CONVERTED into MethodInterceptors]
```

**New file:** `src/main/java/com/simplespringmvc/aop/Advice.java`

```java
public interface Advice {
}
```

Every piece of AOP advice ultimately implements this marker. In the real framework this is the AOP Alliance standard `org.aopalliance.aop.Advice`, completely empty -- just a type marker that the advisor infrastructure uses to identify advice objects.

**New file:** `src/main/java/com/simplespringmvc/aop/MethodInterceptor.java`

```java
@FunctionalInterface
public interface MethodInterceptor extends Advice {
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

This is the CORE advice abstraction. All other advice types (@Before, @After, @AfterReturning, @AfterThrowing) are converted into MethodInterceptors before execution. The Chain of Responsibility contract:

```
interceptor.invoke(invocation)
  -> do pre-processing
  -> Object result = invocation.proceed()   // advance to next interceptor or target
  -> do post-processing
  -> return result
```

If an interceptor does NOT call `invocation.proceed()`, the target method is never invoked -- this is how @Around advice can short-circuit execution. Maps to `org.aopalliance.intercept.MethodInterceptor`.

**New file:** `src/main/java/com/simplespringmvc/aop/MethodInvocation.java`

```java
public interface MethodInvocation {
    Object proceed() throws Throwable;
    Method getMethod();
    Object[] getArguments();
    Object getThis();
}
```

Represents a method invocation being intercepted -- the joinpoint. The real framework has a deeper hierarchy (`Joinpoint -> Invocation -> MethodInvocation`). We flatten it into a single interface. Maps to `org.aopalliance.intercept.MethodInvocation`.

### Pointcut -- The "Where"

**New file:** `src/main/java/com/simplespringmvc/aop/Pointcut.java`

```java
public interface Pointcut {
    boolean matches(Method method, Class<?> targetClass);
    Pointcut TRUE = (method, targetClass) -> true;
}
```

Determines whether advice should apply to a given method on a given class. The real framework separates `Pointcut` into two parts:

```
Pointcut
  ClassFilter   getClassFilter()   -- coarse-grained: does this class match?
  MethodMatcher getMethodMatcher()  -- fine-grained: does this method match?
```

We combine both checks into a single `matches()` method. The two-phase design in real Spring allows caching: the ClassFilter is checked once per class, then MethodMatcher per method.

Two implementations:

**New file:** `src/main/java/com/simplespringmvc/aop/NamePatternPointcut.java` -- matches methods by name using wildcard patterns (`"*"`, `"save*"`, `"*Data"`, `"*ave*"`, `"doSomething"`). Maps to `NameMatchMethodPointcut`. The real framework also supports AspectJ expression pointcuts like `"execution(* com.example.service.*.*(..))"` -- we use simple name patterns instead.

**New file:** `src/main/java/com/simplespringmvc/aop/AnnotationPointcut.java` -- matches methods carrying a specific annotation (e.g., `@Transactional`). Also checks the implementation class method when the method comes from an interface. Maps to `AnnotationMethodMatcher`.

### Advisor -- Advice + Pointcut

**New file:** `src/main/java/com/simplespringmvc/aop/Advisor.java`

```java
public interface Advisor {
    Advice getAdvice();
    Pointcut getPointcut();
}
```

The fundamental unit of AOP configuration -- each advisor represents one cross-cutting concern applied to specific methods. Maps to `PointcutAdvisor`.

**New file:** `src/main/java/com/simplespringmvc/aop/DefaultPointcutAdvisor.java` -- simple implementation pairing an `Advice` with a `Pointcut`. When no pointcut is specified, uses `Pointcut.TRUE` (matches everything).

### AdvisedSupport -- Configuration Holder

**New file:** `src/main/java/com/simplespringmvc/aop/AdvisedSupport.java`

Stores the target object, interfaces to proxy, and the advisor chain. Both `JdkDynamicAopProxy` and `CglibAopProxy` read their configuration from this object.

The key method is `getInterceptors(Method)` -- it builds the interceptor chain for a specific method by iterating all advisors, checking each pointcut, and collecting matching MethodInterceptors:

```java
public List<MethodInterceptor> getInterceptors(Method method) {
    List<MethodInterceptor> result = new ArrayList<>();
    Class<?> targetClass = target.getClass();
    Method targetMethod = resolveTargetMethod(method, targetClass);

    for (Advisor advisor : advisors) {
        if (advisor.getPointcut().matches(targetMethod, targetClass)) {
            Advice advice = advisor.getAdvice();
            if (advice instanceof MethodInterceptor mi) {
                result.add(mi);
            }
        }
    }
    return result;
}
```

Maps to `AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice()` (line 516) which delegates to `DefaultAdvisorChainFactory`. The real framework caches chains per method and uses `AdvisorAdapterRegistry` to convert non-interceptor advice into `MethodInterceptor`s.

## 28.3 Proxy Implementations

Three classes handle proxy creation: the interceptor chain engine, and two proxy strategies.

### ReflectiveMethodInvocation -- The Heart of AOP

**New file:** `src/main/java/com/simplespringmvc/aop/ReflectiveMethodInvocation.java`

Walks the interceptor chain using the **Chain of Responsibility** pattern, then invokes the target method via reflection when all interceptors have run:

```
proceed()  [index=-1]
  -> interceptor[0].invoke(this)
    -> proceed()  [index=0]
      -> interceptor[1].invoke(this)
        -> proceed()  [index=1]
          -> (all interceptors done) method.invoke(target, args)
        <- return result
      <- return result
    <- return result
  <- return result
```

Each interceptor receives THIS invocation object. When it calls `proceed()`, the index advances to the next interceptor. When all interceptors have run, `proceed()` invokes the real target method. The call stack naturally unwinds through all interceptors.

```java
public Object proceed() throws Throwable {
    if (currentInterceptorIndex == interceptors.size() - 1) {
        return invokeJoinpoint();  // all interceptors done -- call target
    }
    MethodInterceptor interceptor = interceptors.get(++currentInterceptorIndex);
    return interceptor.invoke(this);
}
```

Maps to `ReflectiveMethodInvocation.proceed()` (line 155).

### The Strategy Pattern: AopProxy

```
                        AopProxy
                       /         \
          JdkDynamicAopProxy    CglibAopProxy
          (interface-based)     (subclass-based)
```

**New file:** `src/main/java/com/simplespringmvc/aop/AopProxy.java`

```java
public interface AopProxy {
    Object getProxy();
}
```

### JdkDynamicAopProxy -- Interface-Based

**New file:** `src/main/java/com/simplespringmvc/aop/JdkDynamicAopProxy.java`

Uses `java.lang.reflect.Proxy.newProxyInstance()` to create a proxy implementing the target's interfaces. Implements `InvocationHandler` -- every method call on the proxy triggers `invoke()`:

```
proxy.greet("World")
  -> JdkDynamicAopProxy.invoke(proxy, method, args)
    -> advised.getInterceptors(method)    // build chain
    -> chain empty? method.invoke(target, args)   // direct call
    -> chain not empty? new ReflectiveMethodInvocation(...).proceed()
```

Maps to `JdkDynamicAopProxy` (line 70, ~359 lines). The real version also handles `Advised` interface exposure, `DecoratingProxy`, `AopContext.setCurrentProxy()`, and `TargetSource` lifecycle.

### CglibAopProxy -- Subclass-Based (via ByteBuddy)

**New file:** `src/main/java/com/simplespringmvc/aop/CglibAopProxy.java`

Creates a subclass of the target class at runtime using ByteBuddy. The proxy IS an instance of the target class (`proxy instanceof TargetClass` is true).

```
JDK proxy:   interface-based -- Proxy.newProxyInstance()
             + No extra library needed
             + Target must implement interfaces
             - Cannot proxy class-only methods

CGLIB proxy: subclass-based -- generates a subclass at runtime
             + Can proxy any class (not just interfaces)
             + proxy instanceof TargetClass is true
             - Requires bytecode generation library
             - Cannot proxy final classes/methods
```

The real Spring uses CGLIB (repackaged in spring-core as `org.springframework.cglib`). We use ByteBuddy -- a modern bytecode generation library. The concept is identical: create a subclass that overrides methods and delegates to the advice chain.

**Simplification:** Requires target class to have a no-arg constructor. Real Spring uses Objenesis to bypass constructor invocation.

Maps to `CglibAopProxy` (line 87, ~910 lines). The real version has 7 different callback types selected per-method by `ProxyCallbackFilter`.

### ProxyFactory -- The User-Facing API

**New file:** `src/main/java/com/simplespringmvc/aop/ProxyFactory.java`

The main API for creating AOP proxies. Auto-selects JDK vs CGLIB based on config:

```
proxyTargetClass=true?           -> CGLIB
target has interfaces?           -> JDK dynamic proxy
target has NO interfaces?        -> CGLIB
```

This mirrors `DefaultAopProxyFactory.createAopProxy()` (line 60-76). Usage:

```java
ProxyFactory factory = new ProxyFactory(myService);
factory.addAdvice((MethodInterceptor) invocation -> {
    System.out.println("Before: " + invocation.getMethod().getName());
    return invocation.proceed();
});
MyService proxy = (MyService) factory.getProxy();
```

The full proxy creation flow:

```
  ProxyFactory.getProxy()
    |
    v
  createAopProxy()  ---- selects strategy
    |                    (DefaultAopProxyFactory logic)
    |-- has interfaces & !proxyTargetClass --> JdkDynamicAopProxy
    |-- no interfaces  | proxyTargetClass  --> CglibAopProxy
    |
    v
  AopProxy.getProxy()
    |
    |-- JDK:   Proxy.newProxyInstance(classLoader, interfaces, handler)
    |-- CGLIB: ByteBuddy.subclass(targetClass)...intercept(handler)...make()
    |
    v
  proxy object (wraps target + advisor chain)
    |
    v
  every method call on proxy:
    |
    |-- build interceptor chain: AdvisedSupport.getInterceptors(method)
    |-- if chain empty: invoke target directly
    |-- if chain not empty: ReflectiveMethodInvocation.proceed()
    |     |
    |     |-- interceptor[0].invoke(this)
    |     |     |-- interceptor[1].invoke(this)
    |     |     |     |-- ... (chain walks)
    |     |     |     |-- method.invoke(target, args)  <-- actual call
    |     |     |     |-- return result
    |     |     |-- return result
    |     |-- return result
```

## 28.4 AspectJ Annotations and Auto-Proxy Creator

### The Annotations

Six annotations in the `aop.aspectj` subpackage:

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/Aspect.java` -- marks a class as containing advice methods. Maps to `org.aspectj.lang.annotation.Aspect`.

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/Before.java` -- before advice, value is method name pattern. Runs before the matched method; if it throws, the target is NOT called.

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/After.java` -- after (finally) advice. Runs after the matched method regardless of outcome.

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/Around.java` -- the most powerful advice type. The advice method receives a `MethodInvocation` and fully controls whether/when to call `proceed()`.

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/AfterReturning.java` -- runs only when the matched method returns normally.

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/AfterThrowing.java` -- runs only when the matched method throws an exception.

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/EnableAspectJAutoProxy.java` -- activates automatic AOP proxy creation. Has a `proxyTargetClass` attribute to force CGLIB proxies. Maps to `org.springframework.context.annotation.EnableAspectJAutoProxy`.

### AspectJAutoProxyCreator -- The Orchestrator

**New file:** `src/main/java/com/simplespringmvc/aop/aspectj/AspectJAutoProxyCreator.java`

This is the BeanPostProcessor that ties everything together. For every bean created by the container:

```
postProcessAfterInitialization(bean, beanName)
  |
  |-- isInfrastructureClass(bean.getClass())?  --> skip (aspects, BPPs, advisors)
  |
  |-- findApplicableAdvisors(bean.getClass())
  |     |-- getAllAdvisors()
  |     |     |-- find programmatic Advisor beans
  |     |     |-- find @Aspect beans -> buildAdvisorsFromAspect()
  |     |     |     |-- for each @Before/@After/@Around method:
  |     |     |     |     convert to MethodInterceptor + NamePatternPointcut
  |     |     |     |     wrap in DefaultPointcutAdvisor
  |     |
  |     |-- for each advisor: hasMatchingMethod(pointcut, targetClass)?
  |
  |-- advisors empty?  --> return bean (no proxy needed)
  |-- advisors found?  --> createProxy(bean, advisors)
  |     |-- ProxyFactory.setTarget(bean)
  |     |-- ProxyFactory.addInterface(...) for each interface
  |     |-- ProxyFactory.addAdvisor(...) for each advisor
  |     |-- ProxyFactory.getProxy()  --> JDK or CGLIB proxy
  |
  v
  return proxy (container stores this instead of original bean)
```

Maps to `AnnotationAwareAspectJAutoProxyCreator` (line 88) which extends a deep class hierarchy: `AbstractAutoProxyCreator -> AbstractAdvisorAutoProxyCreator -> AspectJAwareAdvisorAutoProxyCreator -> AnnotationAwareAspectJAutoProxyCreator`.

**Advice conversion** -- each annotation type is converted into a `MethodInterceptor` with the appropriate execution semantics:

| Annotation | MethodInterceptor pattern |
|---|---|
| `@Before` | `invokeAdvice(); return invocation.proceed();` |
| `@After` | `try { return invocation.proceed(); } finally { invokeAdvice(); }` |
| `@Around` | `return adviceMethod.invoke(aspect, invocation);` (advice controls proceed) |
| `@AfterReturning` | `Object result = invocation.proceed(); invokeAdvice(result); return result;` |
| `@AfterThrowing` | `try { return invocation.proceed(); } catch (Throwable ex) { invokeAdvice(ex); throw ex; }` |

This conversion is the **Adapter pattern** -- all five advice types become uniform `MethodInterceptor`s, so the interceptor chain only needs one abstraction.

---

## 28.5 Try It Yourself

Before looking at the tests, try implementing these features:

<details>
<summary><strong>Challenge 1:</strong> NamePatternPointcut</summary>

Implement a `Pointcut` that matches methods by name using wildcard patterns:
- `"*"` matches all methods
- `"save*"` matches prefix
- `"*User"` matches suffix
- `"*ave*"` matches contains
- `"doSomething"` matches exact

Hint: Check `pattern.startsWith("*")` and `pattern.endsWith("*")` to determine the match mode.
</details>

<details>
<summary><strong>Challenge 2:</strong> ReflectiveMethodInvocation</summary>

Implement the interceptor chain walker. Key insight: use an `int currentInterceptorIndex` starting at -1. Each call to `proceed()` increments it. When `currentInterceptorIndex == interceptors.size() - 1`, all interceptors have run -- invoke the target method.

Hint: `interceptors.get(++currentInterceptorIndex).invoke(this)` -- the interceptor receives THIS invocation, and when it calls `proceed()`, the index naturally advances.
</details>

<details>
<summary><strong>Challenge 3:</strong> JDK vs CGLIB proxy selection</summary>

In `ProxyFactory.createAopProxy()`, implement the decision logic:
1. `proxyTargetClass=true` -> CGLIB
2. No interfaces -> CGLIB
3. Has interfaces -> JDK

Hint: Check `config.isProxyTargetClass() || config.getInterfaces().isEmpty()`.
</details>

<details>
<summary><strong>Challenge 4:</strong> @Before to MethodInterceptor conversion</summary>

In `AspectJAutoProxyCreator.buildAdvisor()`, convert a `@Before`-annotated method on an aspect into a `MethodInterceptor`. The interceptor should invoke the advice method, then call `invocation.proceed()`.

Hint: `invokeAdviceMethod(aspect, adviceMethod, invocation, null, null); return invocation.proceed();`
</details>

---

## 28.6 Tests

### Pointcut Tests

`src/test/java/com/simplespringmvc/aop/PointcutTest.java` -- 9 tests covering both pointcut implementations:

| Test | What it verifies |
|---|---|
| `shouldMatchAllMethods_WhenPatternIsStar` | `"*"` matches any method |
| `shouldMatchPrefix_WhenPatternEndsWithStar` | `"save*"` matches `saveUser`, `saveOrder`, not `findUser` |
| `shouldMatchSuffix_WhenPatternStartsWithStar` | `"*User"` matches `saveUser`, `findUser`, not `saveOrder` |
| `shouldMatchContains_WhenPatternHasStarOnBothEnds` | `"*User*"` matches methods containing "User" |
| `shouldMatchExact_WhenPatternHasNoStar` | `"saveUser"` matches exactly, not `saveOrder` |
| `shouldMatchAnnotatedMethod_WhenAnnotationPresent` | `@Logged` on method -> match |
| `shouldNotMatchUnannotatedMethod_WhenAnnotationAbsent` | No `@Logged` -> no match |
| `shouldMatchAnnotationOnImplMethod_WhenCalledViaInterface` | Annotation on impl class found when method is from interface |
| `shouldAlwaysMatch_WhenUsingPointcutTrue` | `Pointcut.TRUE` matches everything |

### ReflectiveMethodInvocation Tests

`src/test/java/com/simplespringmvc/aop/ReflectiveMethodInvocationTest.java` -- 5 tests:

| Test | What it verifies |
|---|---|
| `shouldInvokeTargetDirectly_WhenNoInterceptors` | Empty chain -> direct call, correct result |
| `shouldWalkChainInOrder_WhenMultipleInterceptors` | Two interceptors run in order: first-before, second-before, second-after, first-after |
| `shouldExposeMethodAndArguments_WhenInterceptorInspects` | `getMethod()`, `getArguments()`, `getThis()` expose joinpoint info |
| `shouldShortCircuit_WhenInterceptorDoesNotCallProceed` | Interceptor returns early -> target never called |
| `shouldPropagateTargetException_WhenTargetThrows` | `ArithmeticException` propagates through chain |

### JdkDynamicAopProxy Tests

`src/test/java/com/simplespringmvc/aop/JdkDynamicAopProxyTest.java` -- 5 tests:

| Test | What it verifies |
|---|---|
| `shouldDelegateToTarget_WhenNoAdvisors` | No advisors -> pass-through to target |
| `shouldApplyInterceptor_WhenAdvisorMatches` | Interceptor modifies return value |
| `shouldImplementSpecifiedInterfaces_WhenProxyCreated` | `proxy instanceof Service` but NOT `instanceof SimpleService` |
| `shouldHandleFluentApi_WhenTargetReturnsThis` | Target returns `this` -> proxy returns proxy (not raw target) |
| `shouldPropagateException_WhenTargetThrows` | Target exception propagates through proxy |

### CglibAopProxy Tests

`src/test/java/com/simplespringmvc/aop/CglibAopProxyTest.java` -- 6 tests:

| Test | What it verifies |
|---|---|
| `shouldDelegateToTarget_WhenNoAdvisors` | Pass-through for unadvised methods |
| `shouldApplyInterceptor_WhenAdvisorMatches` | Interceptor doubles result: `(3+4)*2 = 14` |
| `shouldBeInstanceOfTargetClass_WhenProxyCreated` | `proxy instanceof Calculator` (subclass-based) |
| `shouldApplyPointcutFiltering_WhenMethodNameMatches` | Only `add` intercepted, not `multiply` |
| `shouldPropagateException_WhenTargetThrows` | `ArithmeticException` propagates through CGLIB proxy |
| `shouldThrowClearError_WhenNoDefaultConstructor` | Clear error message when no-arg constructor missing |

### ProxyFactory Tests

`src/test/java/com/simplespringmvc/aop/ProxyFactoryTest.java` -- 6 tests:

| Test | What it verifies |
|---|---|
| `shouldCreateJdkProxy_WhenTargetImplementsInterfaces` | Auto-selects JDK proxy, interceptor wraps result |
| `shouldApplyMultipleAdvice_WhenMultipleInterceptorsAdded` | Two interceptors run in order |
| `shouldApplyPointcutFiltering_WhenAdvisorHasPointcut` | Only matching methods intercepted |
| `shouldNotIntercept_WhenPointcutDoesNotMatch` | Non-matching pointcut -> no interception |
| `shouldCreateCglibProxy_WhenProxyTargetClassIsTrue` | Forced CGLIB: `proxy instanceof SimpleGreetingService` |
| `shouldCreateCglibProxy_WhenTargetHasNoInterfaces` | No interfaces -> auto-selects CGLIB |

### AspectJAutoProxyCreator Tests

`src/test/java/com/simplespringmvc/aop/aspectj/AspectJAutoProxyCreatorTest.java` -- 7 tests:

| Test | What it verifies |
|---|---|
| `shouldApplyBeforeAdvice_WhenAspectBeanRegistered` | `@Before("greet")` runs before target |
| `shouldApplyAfterAdvice_WhenAspectBeanRegistered` | `@After("greet")` runs after target |
| `shouldApplyAroundAdvice_WhenAspectBeanRegistered` | `@Around("greet")` wraps result with `"[TIMED] "` |
| `shouldApplyAfterReturningAdvice_WhenMethodReturnsNormally` | Captures return value |
| `shouldApplyAfterThrowingAdvice_WhenMethodThrows` | Captures thrown exception |
| `shouldApplyProgrammaticAdvisor_WhenAdvisorBeanRegistered` | Programmatic `Advisor` bean works alongside @Aspect |
| `shouldNotProxyAspectBean_WhenAspectClassDetected` | Infrastructure beans (aspects) are NOT proxied |

### Integration Tests

`src/test/java/com/simplespringmvc/integration/AopIntegrationTest.java` -- 5 tests:

| Test | What it verifies |
|---|---|
| `shouldProxyBean_WhenContainerRefreshWithAspectAndEnableAop` | End-to-end: container + @Aspect + @EnableAspectJAutoProxy |
| `shouldProxyMultipleBeans_WhenMultipleMatchingBeans` | Two different services both proxied by one @Around("*") aspect |
| `shouldNotProxy_WhenNoMatchingAdvice` | Aspect matching "save*" does NOT intercept "find*" methods |
| `shouldWorkWithProgrammaticAdvisors_WhenAdvisorBeanRegistered` | Programmatic `Advisor` bean with container integration |
| `shouldNotEnableAop_WhenNoEnableAspectJAutoProxy` | Without @EnableAspectJAutoProxy, no proxying occurs at all |

---

## 28.7 Why This Works

> **Insight: The BeanPostProcessor hook is the entire AOP integration point.** The container's `createBean()` loop calls `postProcessAfterInitialization()` for every bean. By returning a proxy instead of the original bean, the `AspectJAutoProxyCreator` transparently replaces the raw object with an advised proxy. No other container code needs to change. This is the same hook the real Spring Framework uses -- the AOP infrastructure is a plugin to the bean lifecycle, not hardcoded into the container.

> **Insight: All advice types reduce to MethodInterceptor.** @Before, @After, @AfterReturning, @AfterThrowing, and @Around all become uniform `MethodInterceptor` instances. This is the Adapter pattern -- the interceptor chain only works with one abstraction, and the annotation-specific semantics (before/after/around) are encoded in the lambda wrapping. The real framework does the same conversion via `AdvisorAdapterRegistry.getInterceptors()`.

> **Insight: The interceptor chain is recursive, not iterative.** `ReflectiveMethodInvocation.proceed()` does not use a for-loop. Each interceptor receives the invocation and calls `proceed()` on its own schedule. The call stack IS the chain. This means an interceptor can wrap the entire invocation in a try-catch (for @AfterThrowing), a try-finally (for @After), or transform the result (for @Around) -- all using natural Java control flow. An iterative design would require separate "before" and "after" phases, losing the ability to wrap.

> **Insight: JDK vs CGLIB is a Strategy swap, invisible to callers.** The `ProxyFactory` selects between `JdkDynamicAopProxy` and `CglibAopProxy` based on configuration, but the `AopProxy.getProxy()` return type is just `Object`. The caller casts to the target's interface or class. The proxy implementation is completely hidden behind the Strategy interface.

---

## 28.8 What We Enhanced

| File | Status | Change |
|------|--------|--------|
| `aop/Advice.java` | NEW | Tag interface for all advice types |
| `aop/MethodInterceptor.java` | NEW | @FunctionalInterface for around advice |
| `aop/MethodInvocation.java` | NEW | Joinpoint interface: proceed(), getMethod(), getArguments(), getThis() |
| `aop/ReflectiveMethodInvocation.java` | NEW | Chain of Responsibility implementation |
| `aop/Pointcut.java` | NEW | Method matching interface with TRUE constant |
| `aop/NamePatternPointcut.java` | NEW | Wildcard method name matching |
| `aop/AnnotationPointcut.java` | NEW | Annotation-based method matching |
| `aop/Advisor.java` | NEW | Advice + Pointcut composite |
| `aop/DefaultPointcutAdvisor.java` | NEW | Simple Advisor implementation |
| `aop/AdvisedSupport.java` | NEW | Configuration holder with interceptor chain building |
| `aop/AopProxy.java` | NEW | Strategy interface for proxy creation |
| `aop/JdkDynamicAopProxy.java` | NEW | Interface-based proxy (Proxy.newProxyInstance) |
| `aop/CglibAopProxy.java` | NEW | Subclass-based proxy (ByteBuddy) |
| `aop/ProxyFactory.java` | NEW | Factory with auto JDK/CGLIB selection |
| `aop/aspectj/Aspect.java` | NEW | Marks class as aspect |
| `aop/aspectj/Before.java` | NEW | Before advice annotation |
| `aop/aspectj/After.java` | NEW | After (finally) advice annotation |
| `aop/aspectj/Around.java` | NEW | Around advice annotation |
| `aop/aspectj/AfterReturning.java` | NEW | After-returning advice annotation |
| `aop/aspectj/AfterThrowing.java` | NEW | After-throwing advice annotation |
| `aop/aspectj/EnableAspectJAutoProxy.java` | NEW | Activates auto-proxying |
| `aop/aspectj/AspectJAutoProxyCreator.java` | NEW | BPP that discovers advisors and wraps beans |
| `container/SimpleBeanContainer.java` | MODIFIED | Added getType(), autoRegisterAopProcessorIfNeeded(), AOP imports |
| `build.gradle` | MODIFIED | Added net.bytebuddy:byte-buddy:1.14.18 |

---

## 28.9 Connection to Real Framework

| Our class | Real Spring class | Key line / file |
|-----------|-------------------|-----------------|
| `Advice` | `org.aopalliance.aop.Advice` | AOP Alliance standard |
| `MethodInterceptor` | `org.aopalliance.intercept.MethodInterceptor` | AOP Alliance standard |
| `MethodInvocation` | `org.aopalliance.intercept.MethodInvocation` | AOP Alliance standard |
| `ReflectiveMethodInvocation` | `ReflectiveMethodInvocation` | `ReflectiveMethodInvocation.java:155` (proceed) |
| `Pointcut` + `NamePatternPointcut` | `Pointcut` + `MethodMatcher` + `NameMatchMethodPointcut` | `NameMatchMethodPointcut.java` |
| `AnnotationPointcut` | `AnnotationMethodMatcher` | `AnnotationMethodMatcher.java` |
| `Advisor` + `DefaultPointcutAdvisor` | `PointcutAdvisor` + `DefaultPointcutAdvisor` | `DefaultPointcutAdvisor.java` |
| `AdvisedSupport` | `AdvisedSupport` | `AdvisedSupport.java:516` (getInterceptorsAndDynamicInterceptionAdvice) |
| `AopProxy` | `AopProxy` | `AopProxy.java` |
| `JdkDynamicAopProxy` | `JdkDynamicAopProxy` | `JdkDynamicAopProxy.java:166` (invoke) |
| `CglibAopProxy` | `CglibAopProxy` | `CglibAopProxy.java:682` (DynamicAdvisedInterceptor) |
| `ProxyFactory` | `ProxyFactory` + `DefaultAopProxyFactory` | `ProxyFactory.java:96` (getProxy), `DefaultAopProxyFactory.java:60` (JDK vs CGLIB) |
| `AspectJAutoProxyCreator` | `AnnotationAwareAspectJAutoProxyCreator` | `AnnotationAwareAspectJAutoProxyCreator.java:88` (findCandidateAdvisors) |
| `@Aspect` | `org.aspectj.lang.annotation.Aspect` | AspectJ library |
| `@Before/@After/@Around` | `org.aspectj.lang.annotation.Before/After/Around` | AspectJ library |
| `@EnableAspectJAutoProxy` | `EnableAspectJAutoProxy` | `EnableAspectJAutoProxy.java` |

**Simplifications vs real Spring:**

| Simplification | Real Framework |
|---|---|
| No AspectJ expression pointcuts | Full `execution()`, `within()`, `@annotation()` etc. via `AspectJExpressionPointcut` |
| No advisor ordering | `@Order` / `PriorityOrdered` for deterministic advice order |
| No early proxy reference | Three-level singleton cache for circular dependency with AOP |
| No proxy class caching | `CglibAopProxy` caches generated subclasses |
| No TargetSource abstraction | Supports hot-swappable targets, pooling, lazy init |
| No AdvisorAdapterRegistry | Direct conversion in auto-proxy creator |
| CGLIB proxy requires no-arg constructor | Real Spring uses Objenesis to bypass constructors |
| No ExposeInvocationInterceptor | Thread-local `MethodInvocation` for self-invocation |
| Uses ByteBuddy instead of CGLIB | Same concept (runtime subclass creation), different library |

---

## 28.10 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/aop/Advice.java` [NEW]

```java
package com.simplespringmvc.aop;

/**
 * Tag interface for all advice types. Every piece of AOP advice
 * (before, after, around) ultimately implements this marker.
 *
 * Maps to: {@code org.aopalliance.aop.Advice}
 * (AOP Alliance standard — bundled inside Spring's source tree)
 *
 * In the real framework this is completely empty — just a type marker
 * that the advisor infrastructure uses to identify advice objects.
 */
public interface Advice {
}
```

#### File: `src/main/java/com/simplespringmvc/aop/MethodInterceptor.java` [NEW]

```java
package com.simplespringmvc.aop;

/**
 * Around advice — intercepts a method invocation and controls whether/when
 * to proceed to the next interceptor or the target method.
 *
 * Maps to: {@code org.aopalliance.intercept.MethodInterceptor}
 *
 * This is the CORE advice abstraction. All other advice types (@Before, @After,
 * @AfterReturning, @AfterThrowing) are converted into MethodInterceptors
 * before execution. The interceptor chain is a uniform list of MethodInterceptors.
 *
 * <h3>The Chain of Responsibility contract:</h3>
 * <pre>
 *   interceptor.invoke(invocation)
 *     → do pre-processing
 *     → Object result = invocation.proceed()   // advance to next interceptor or target
 *     → do post-processing
 *     → return result
 * </pre>
 *
 * If an interceptor does NOT call {@code invocation.proceed()}, the target
 * method is never invoked — this is how @Around advice can short-circuit execution.
 */
@FunctionalInterface
public interface MethodInterceptor extends Advice {

    /**
     * Intercept the given method invocation.
     *
     * @param invocation the method invocation joinpoint
     * @return the result of calling {@code invocation.proceed()},
     *         possibly modified by the interceptor
     * @throws Throwable if the interceptor or the target method throws
     */
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

#### File: `src/main/java/com/simplespringmvc/aop/MethodInvocation.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.reflect.Method;

/**
 * Represents a method invocation that is being intercepted — the joinpoint.
 * Provides access to the target object, method, arguments, and the ability
 * to advance the interceptor chain via {@link #proceed()}.
 *
 * Maps to: {@code org.aopalliance.intercept.MethodInvocation}
 * (extends Invocation extends Joinpoint)
 *
 * The real framework has a deeper hierarchy:
 * <pre>
 *   Joinpoint          → proceed(), getThis(), getStaticPart()
 *     Invocation       → getArguments()
 *       MethodInvocation → getMethod()
 * </pre>
 *
 * We flatten this into a single interface for simplicity.
 */
public interface MethodInvocation {

    /**
     * Proceed to the next interceptor in the chain, or invoke the target
     * method if all interceptors have been called.
     *
     * @return the return value of the method invocation
     * @throws Throwable if the invocation throws
     */
    Object proceed() throws Throwable;

    /**
     * Return the method being invoked.
     */
    Method getMethod();

    /**
     * Return the arguments to the method invocation.
     */
    Object[] getArguments();

    /**
     * Return the target object — the object the method will be invoked on.
     */
    Object getThis();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/ReflectiveMethodInvocation.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.List;

/**
 * Walks the interceptor chain using the Chain of Responsibility pattern,
 * then invokes the target method via reflection when all interceptors have run.
 *
 * Maps to: {@code org.springframework.aop.framework.ReflectiveMethodInvocation}
 *
 * <h3>Chain execution (the heart of AOP):</h3>
 * <pre>
 *   proceed()  [index=-1]
 *     → interceptor[0].invoke(this)
 *       → proceed()  [index=0]
 *         → interceptor[1].invoke(this)
 *           → proceed()  [index=1]
 *             → (all interceptors done) method.invoke(target, args)
 *           ← return result
 *         ← return result
 *       ← return result
 *     ← return result
 * </pre>
 *
 * Each interceptor receives THIS invocation object. When it calls
 * {@code proceed()}, the index advances to the next interceptor.
 * When all interceptors have run, {@code proceed()} invokes the real
 * target method. The call stack naturally unwinds through all interceptors.
 */
public class ReflectiveMethodInvocation implements MethodInvocation {

    private final Object proxy;
    private final Object target;
    private final Method method;
    private final Object[] arguments;
    private final List<MethodInterceptor> interceptors;

    /**
     * Index of the current interceptor. Starts at -1 (before any interceptor).
     * Each call to proceed() increments this before invoking the next interceptor.
     *
     * Real Spring: ReflectiveMethodInvocation.currentInterceptorIndex (line 72)
     */
    private int currentInterceptorIndex = -1;

    public ReflectiveMethodInvocation(Object proxy, Object target, Method method,
                                      Object[] arguments, List<MethodInterceptor> interceptors) {
        this.proxy = proxy;
        this.target = target;
        this.method = method;
        this.arguments = arguments != null ? arguments : new Object[0];
        this.interceptors = interceptors;
    }

    /**
     * Advance to the next interceptor, or invoke the target method if
     * all interceptors have been called.
     *
     * Maps to: {@code ReflectiveMethodInvocation.proceed()} (line 155-181)
     */
    @Override
    public Object proceed() throws Throwable {
        // All interceptors have been invoked — call the actual target method
        if (currentInterceptorIndex == interceptors.size() - 1) {
            return invokeJoinpoint();
        }

        // Advance to the next interceptor and invoke it
        MethodInterceptor interceptor = interceptors.get(++currentInterceptorIndex);
        return interceptor.invoke(this);
    }

    /**
     * Invoke the target method via reflection.
     * Unwraps InvocationTargetException to propagate the original exception.
     *
     * Maps to: {@code AopUtils.invokeJoinpointUsingReflection()} (line 296)
     */
    protected Object invokeJoinpoint() throws Throwable {
        try {
            method.setAccessible(true);
            return method.invoke(target, arguments);
        } catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }

    @Override
    public Method getMethod() {
        return method;
    }

    @Override
    public Object[] getArguments() {
        return arguments;
    }

    @Override
    public Object getThis() {
        return target;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/Pointcut.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.reflect.Method;

/**
 * Determines whether advice should apply to a given method on a given class.
 * The "where" of AOP — while {@link Advice} is the "what".
 *
 * Maps to: {@code org.springframework.aop.Pointcut} combined with
 * {@code org.springframework.aop.MethodMatcher}
 *
 * The real framework separates Pointcut into two parts:
 * <pre>
 *   Pointcut
 *     ClassFilter   getClassFilter()   — coarse-grained: does this class match?
 *     MethodMatcher getMethodMatcher()  — fine-grained: does this method match?
 * </pre>
 *
 * We combine both checks into a single {@code matches()} method.
 * The two-phase design in real Spring allows caching: the ClassFilter
 * is checked once per class, then MethodMatcher per method.
 */
public interface Pointcut {

    /**
     * Does this pointcut match the given method on the given target class?
     *
     * @param method      the candidate method
     * @param targetClass the target class (may differ from method.getDeclaringClass()
     *                    if the method is inherited)
     * @return true if advice should apply to this method
     */
    boolean matches(Method method, Class<?> targetClass);

    /**
     * Canonical Pointcut that matches everything — used when no filtering is needed.
     */
    Pointcut TRUE = (method, targetClass) -> true;
}
```

#### File: `src/main/java/com/simplespringmvc/aop/NamePatternPointcut.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.reflect.Method;

/**
 * Matches methods by name using a simple wildcard pattern.
 *
 * Supported patterns:
 * <ul>
 *   <li>{@code "*"} — matches all methods</li>
 *   <li>{@code "save*"} — prefix match (methods starting with "save")</li>
 *   <li>{@code "*Data"} — suffix match (methods ending with "Data")</li>
 *   <li>{@code "*ave*"} — contains match (methods containing "ave")</li>
 *   <li>{@code "doSomething"} — exact match</li>
 * </ul>
 *
 * Maps to: {@code org.springframework.aop.support.NameMatchMethodPointcut}
 *
 * The real framework also supports AspectJ expression pointcuts like
 * {@code "execution(* com.example.service.*.*(..))"} via
 * {@code AspectJExpressionPointcut}. We use simple name patterns instead —
 * the matching concept is identical, just the expression syntax differs.
 */
public class NamePatternPointcut implements Pointcut {

    private final String pattern;

    public NamePatternPointcut(String pattern) {
        this.pattern = pattern;
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        String methodName = method.getName();

        if ("*".equals(pattern)) {
            return true;
        }

        boolean startsWithStar = pattern.startsWith("*");
        boolean endsWithStar = pattern.endsWith("*");

        if (startsWithStar && endsWithStar && pattern.length() > 2) {
            // *contains* — match if name contains the middle part
            String middle = pattern.substring(1, pattern.length() - 1);
            return methodName.contains(middle);
        }
        if (startsWithStar) {
            // *suffix — match if name ends with the suffix
            return methodName.endsWith(pattern.substring(1));
        }
        if (endsWithStar) {
            // prefix* — match if name starts with the prefix
            return methodName.startsWith(pattern.substring(0, pattern.length() - 1));
        }

        // Exact match
        return methodName.equals(pattern);
    }

    public String getPattern() {
        return pattern;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/AnnotationPointcut.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

/**
 * Matches methods that carry a specific annotation.
 * For example, {@code new AnnotationPointcut(Transactional.class)} matches
 * all methods annotated with {@code @Transactional}.
 *
 * Maps to: {@code org.springframework.aop.support.annotation.AnnotationMethodMatcher}
 *
 * This enables annotation-driven AOP — the most common usage pattern in modern
 * Spring applications (e.g., @Transactional, @Cacheable, @Async).
 */
public class AnnotationPointcut implements Pointcut {

    private final Class<? extends Annotation> annotationType;

    public AnnotationPointcut(Class<? extends Annotation> annotationType) {
        this.annotationType = annotationType;
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        // Check the method itself
        if (method.isAnnotationPresent(annotationType)) {
            return true;
        }

        // Also check the method on the target class (in case method is from an interface)
        try {
            Method targetMethod = targetClass.getMethod(method.getName(), method.getParameterTypes());
            return targetMethod.isAnnotationPresent(annotationType);
        } catch (NoSuchMethodException e) {
            return false;
        }
    }

    public Class<? extends Annotation> getAnnotationType() {
        return annotationType;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/Advisor.java` [NEW]

```java
package com.simplespringmvc.aop;

/**
 * Combines advice (what to do) with a pointcut (where to apply it).
 * This is the fundamental unit of AOP configuration — each advisor
 * represents one cross-cutting concern applied to specific methods.
 *
 * Maps to: {@code org.springframework.aop.PointcutAdvisor}
 * (extends {@code org.springframework.aop.Advisor})
 *
 * The real framework has a base Advisor interface with just getAdvice(),
 * and PointcutAdvisor adds getPointcut(). We combine them since all our
 * advisors are pointcut-based.
 */
public interface Advisor {

    /**
     * Return the advice (action) for this advisor.
     */
    Advice getAdvice();

    /**
     * Return the pointcut (filter) that determines where the advice applies.
     */
    Pointcut getPointcut();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/DefaultPointcutAdvisor.java` [NEW]

```java
package com.simplespringmvc.aop;

/**
 * Default implementation pairing an {@link Advice} with a {@link Pointcut}.
 *
 * Maps to: {@code org.springframework.aop.support.DefaultPointcutAdvisor}
 *
 * When no pointcut is specified, uses {@link Pointcut#TRUE} (matches everything).
 * This is what {@link AdvisedSupport#addAdvice(Advice)} uses to wrap bare advice.
 */
public class DefaultPointcutAdvisor implements Advisor {

    private final Pointcut pointcut;
    private final Advice advice;

    /**
     * Create an advisor that applies the given advice to ALL methods.
     */
    public DefaultPointcutAdvisor(Advice advice) {
        this(Pointcut.TRUE, advice);
    }

    /**
     * Create an advisor that applies the given advice to methods matching the pointcut.
     */
    public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice) {
        this.pointcut = pointcut;
        this.advice = advice;
    }

    @Override
    public Pointcut getPointcut() {
        return pointcut;
    }

    @Override
    public Advice getAdvice() {
        return advice;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/AdvisedSupport.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

/**
 * Configuration holder for AOP proxy creation. Stores the target object,
 * interfaces to proxy, and the advisor chain. Both {@link JdkDynamicAopProxy}
 * and {@link CglibAopProxy} read their configuration from this object.
 *
 * Maps to: {@code org.springframework.aop.framework.AdvisedSupport}
 * (extends ProxyConfig implements Advised)
 *
 * The real AdvisedSupport is ~750 lines with:
 * <ul>
 *   <li>TargetSource abstraction (supports hot-swappable targets, pooling)</li>
 *   <li>AdvisorChainFactory with method-level caching</li>
 *   <li>Advisor change listeners</li>
 *   <li>ProxyConfig flags (optimize, exposeProxy, frozen, etc.)</li>
 * </ul>
 *
 * We keep the essential fields: target, interfaces, advisors, proxyTargetClass.
 */
public class AdvisedSupport {

    private Object target;
    private final List<Class<?>> interfaces = new ArrayList<>();
    private final List<Advisor> advisors = new ArrayList<>();
    private boolean proxyTargetClass = false;

    // ─── Target ──────────────────────────────────────────────

    public void setTarget(Object target) {
        this.target = target;
    }

    public Object getTarget() {
        return target;
    }

    // ─── Interfaces ──────────────────────────────────────────

    public void addInterface(Class<?> iface) {
        if (!iface.isInterface()) {
            throw new IllegalArgumentException(iface.getName() + " is not an interface");
        }
        if (!interfaces.contains(iface)) {
            interfaces.add(iface);
        }
    }

    public List<Class<?>> getInterfaces() {
        return List.copyOf(interfaces);
    }

    // ─── Advisors ────────────────────────────────────────────

    public void addAdvisor(Advisor advisor) {
        advisors.add(advisor);
    }

    /**
     * Convenience method: wraps bare advice in a {@link DefaultPointcutAdvisor}
     * that matches all methods, then adds it.
     */
    public void addAdvice(Advice advice) {
        advisors.add(new DefaultPointcutAdvisor(advice));
    }

    public List<Advisor> getAdvisors() {
        return List.copyOf(advisors);
    }

    // ─── Proxy configuration ─────────────────────────────────

    public void setProxyTargetClass(boolean proxyTargetClass) {
        this.proxyTargetClass = proxyTargetClass;
    }

    public boolean isProxyTargetClass() {
        return proxyTargetClass;
    }

    // ─── Interceptor chain building ──────────────────────────

    /**
     * Build the interceptor chain for a specific method. Iterates all advisors,
     * checks each pointcut, and collects matching MethodInterceptors.
     *
     * Maps to: {@code AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice()}
     * (line 516) which delegates to {@code DefaultAdvisorChainFactory} (line 58-112).
     *
     * The real framework:
     * <ol>
     *   <li>Caches chains per method (we don't — fine for educational use)</li>
     *   <li>Uses AdvisorAdapterRegistry to convert non-interceptor advice
     *       (MethodBeforeAdvice, AfterReturningAdvice) into MethodInterceptors</li>
     *   <li>Supports runtime (dynamic) matching via InterceptorAndDynamicMethodMatcher</li>
     * </ol>
     *
     * @param method the method being invoked
     * @return ordered list of MethodInterceptors that apply to this method
     */
    public List<MethodInterceptor> getInterceptors(Method method) {
        List<MethodInterceptor> result = new ArrayList<>();
        Class<?> targetClass = target.getClass();

        // Resolve the method against the target class.
        // When using JDK proxies, the method comes from the interface,
        // but annotations may only be on the implementation class's method.
        Method targetMethod = resolveTargetMethod(method, targetClass);

        for (Advisor advisor : advisors) {
            if (advisor.getPointcut().matches(targetMethod, targetClass)) {
                Advice advice = advisor.getAdvice();
                if (advice instanceof MethodInterceptor mi) {
                    result.add(mi);
                }
            }
        }
        return result;
    }

    /**
     * Resolve the method against the target class to ensure we check annotations
     * on the actual implementation, not just the interface.
     */
    private Method resolveTargetMethod(Method method, Class<?> targetClass) {
        try {
            return targetClass.getMethod(method.getName(), method.getParameterTypes());
        } catch (NoSuchMethodException e) {
            return method;
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/AopProxy.java` [NEW]

```java
package com.simplespringmvc.aop;

/**
 * Strategy interface for creating AOP proxies. Implementations decide
 * HOW to create the proxy — JDK dynamic proxy or CGLIB-style subclass.
 *
 * Maps to: {@code org.springframework.aop.framework.AopProxy}
 *
 * The Strategy pattern here is key: the {@link ProxyFactory} delegates
 * to an AopProxy implementation, and the decision between JDK and CGLIB
 * is made by the factory, not by the caller.
 */
public interface AopProxy {

    /**
     * Create a new proxy object.
     *
     * @return the proxy object (implements the target's interfaces for JDK proxy,
     *         or is a subclass of the target for CGLIB proxy)
     */
    Object getProxy();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/JdkDynamicAopProxy.java` [NEW]

```java
package com.simplespringmvc.aop;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.List;

/**
 * AopProxy implementation using JDK dynamic proxies (interface-based).
 *
 * Maps to: {@code org.springframework.aop.framework.JdkDynamicAopProxy}
 * (line 70, ~359 lines)
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@link #getProxy()} creates a JDK proxy implementing the target's interfaces,
 *       with THIS object as the {@link InvocationHandler}</li>
 *   <li>Every method call on the proxy triggers {@link #invoke(Object, Method, Object[])}</li>
 *   <li>{@code invoke()} builds the interceptor chain for the method, then either:
 *       <ul>
 *         <li>Invokes the target directly (empty chain)</li>
 *         <li>Creates a {@link ReflectiveMethodInvocation} to walk the chain</li>
 *       </ul>
 *   </li>
 * </ol>
 *
 * <h3>JDK proxy constraint: interfaces only</h3>
 * {@code java.lang.reflect.Proxy} can only proxy interfaces. If the target
 * doesn't implement any interfaces, use {@link CglibAopProxy} instead.
 * The {@link ProxyFactory} makes this decision automatically.
 */
public class JdkDynamicAopProxy implements AopProxy, InvocationHandler {

    private final AdvisedSupport advised;

    public JdkDynamicAopProxy(AdvisedSupport advised) {
        this.advised = advised;
    }

    /**
     * Create the JDK dynamic proxy.
     *
     * Maps to: {@code JdkDynamicAopProxy.getProxy(ClassLoader)} (line 120-125)
     * which calls {@code Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)}
     */
    @Override
    public Object getProxy() {
        Class<?>[] interfaces = advised.getInterfaces().toArray(new Class<?>[0]);
        ClassLoader classLoader = advised.getTarget().getClass().getClassLoader();
        return Proxy.newProxyInstance(classLoader, interfaces, this);
    }

    /**
     * The interception point — called for EVERY method invoked on the proxy.
     *
     * Maps to: {@code JdkDynamicAopProxy.invoke()} (line 166-255)
     *
     * The real version also handles:
     * <ul>
     *   <li>Advised interface exposure (cast proxy to Advised to inspect config)</li>
     *   <li>DecoratingProxy interface</li>
     *   <li>AopContext.setCurrentProxy() for self-invocation</li>
     *   <li>TargetSource.getTarget() / releaseTarget() lifecycle</li>
     * </ul>
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object target = advised.getTarget();

        // Handle Object methods directly on the target
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(target, args);
        }

        // Build the interceptor chain for this specific method
        List<MethodInterceptor> chain = advised.getInterceptors(method);

        Object result;
        if (chain.isEmpty()) {
            // No advice — invoke the target directly (optimization)
            result = invokeTarget(target, method, args);
        } else {
            // Create a ReflectiveMethodInvocation and kick off the chain
            result = new ReflectiveMethodInvocation(
                    proxy, target, method, args, chain
            ).proceed();
        }

        // If the target returned itself ("this"), return the proxy instead.
        // This ensures fluent APIs (return this) work correctly through proxies.
        // Real Spring: JdkDynamicAopProxy.invoke() lines 227-234
        if (result != null && result == target
                && method.getReturnType().isInstance(proxy)) {
            result = proxy;
        }

        return result;
    }

    private Object invokeTarget(Object target, Method method, Object[] args) throws Throwable {
        try {
            method.setAccessible(true);
            return method.invoke(target, args);
        } catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/CglibAopProxy.java` [NEW]

```java
package com.simplespringmvc.aop;

import net.bytebuddy.ByteBuddy;
import net.bytebuddy.dynamic.loading.ClassLoadingStrategy;
import net.bytebuddy.implementation.InvocationHandlerAdapter;
import net.bytebuddy.matcher.ElementMatchers;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.List;

/**
 * AopProxy implementation using CGLIB-style subclass proxying (via ByteBuddy).
 * Creates a subclass of the target class at runtime that intercepts all public
 * method calls and routes them through the advisor chain.
 *
 * Maps to: {@code org.springframework.aop.framework.CglibAopProxy}
 * (line 87, ~910 lines)
 *
 * <h3>JDK proxy vs CGLIB proxy:</h3>
 * <pre>
 *   JDK proxy:  interface-based — Proxy.newProxyInstance()
 *               + No extra library needed
 *               + Target must implement interfaces
 *               - Cannot proxy class-only methods
 *
 *   CGLIB proxy: subclass-based — generates a subclass at runtime
 *               + Can proxy any class (not just interfaces)
 *               + proxy instanceof TargetClass is true
 *               - Requires bytecode generation library
 *               - Cannot proxy final classes/methods
 * </pre>
 *
 * The real Spring uses CGLIB (repackaged in spring-core as
 * {@code org.springframework.cglib}). We use ByteBuddy — a modern bytecode
 * generation library. The concept is identical: create a subclass that
 * overrides methods and delegates to the advice chain.
 *
 * <h3>Simplification:</h3>
 * Requires target class to have a no-arg constructor.
 * Real Spring uses Objenesis to bypass constructor invocation.
 */
public class CglibAopProxy implements AopProxy {

    private final AdvisedSupport advised;

    public CglibAopProxy(AdvisedSupport advised) {
        this.advised = advised;
    }

    /**
     * Create a CGLIB-style proxy by generating a subclass of the target.
     *
     * Maps to: {@code CglibAopProxy.getProxy(ClassLoader)} (line 174-244)
     * which uses {@code Enhancer} to create a subclass with callback filters.
     *
     * The real version has 7 different callback types (AOP_PROXY, INVOKE_TARGET,
     * NO_OVERRIDE, etc.) selected per-method by ProxyCallbackFilter. We use a
     * single InvocationHandler for all methods — simpler but less optimized.
     */
    @Override
    public Object getProxy() {
        Object target = advised.getTarget();
        Class<?> targetClass = target.getClass();

        // The handler that ByteBuddy will invoke for every intercepted method.
        // Structurally identical to JdkDynamicAopProxy.invoke() — same chain logic.
        // Real Spring: CglibAopProxy.DynamicAdvisedInterceptor (line 682-748)
        InvocationHandler handler = (proxy, method, args) -> {
            // Skip Object methods — let the proxy handle them natively
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(target, args);
            }

            List<MethodInterceptor> chain = advised.getInterceptors(method);

            Object result;
            if (chain.isEmpty()) {
                result = invokeTarget(target, method, args);
            } else {
                result = new ReflectiveMethodInvocation(
                        proxy, target, method, args, chain
                ).proceed();
            }

            // Fluent API support: replace target self-reference with proxy
            if (result != null && result == target
                    && method.getReturnType().isInstance(proxy)) {
                result = proxy;
            }

            return result;
        };

        try {
            // Generate a subclass with ByteBuddy that routes all public methods
            // through our InvocationHandler
            Class<?> proxyClass = new ByteBuddy()
                    .subclass(targetClass)
                    .method(ElementMatchers.isPublic()
                            .and(ElementMatchers.not(
                                    ElementMatchers.isDeclaredBy(Object.class))))
                    .intercept(InvocationHandlerAdapter.of(handler))
                    .make()
                    .load(targetClass.getClassLoader(),
                            ClassLoadingStrategy.Default.WRAPPER)
                    .getLoaded();

            return proxyClass.getDeclaredConstructor().newInstance();
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(
                    "Cannot create CGLIB proxy for " + targetClass.getName()
                            + ": no default constructor found. "
                            + "Real Spring uses Objenesis to bypass this limitation.", e);
        } catch (Exception e) {
            throw new RuntimeException(
                    "Failed to create CGLIB proxy for " + targetClass.getName(), e);
        }
    }

    private Object invokeTarget(Object target, Method method, Object[] args) throws Throwable {
        try {
            method.setAccessible(true);
            return method.invoke(target, args);
        } catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/ProxyFactory.java` [NEW]

```java
package com.simplespringmvc.aop;

/**
 * The main user-facing API for creating AOP proxies programmatically.
 * Configures the proxy (target, interfaces, advisors) and delegates
 * actual proxy creation to the appropriate {@link AopProxy} implementation.
 *
 * Maps to: {@code org.springframework.aop.framework.ProxyFactory}
 * (extends ProxyCreatorSupport extends AdvisedSupport extends ProxyConfig)
 *
 * <h3>Usage:</h3>
 * <pre>
 *   ProxyFactory factory = new ProxyFactory(myService);
 *   factory.addAdvice((MethodInterceptor) invocation -> {
 *       System.out.println("Before: " + invocation.getMethod().getName());
 *       return invocation.proceed();
 *   });
 *   MyService proxy = (MyService) factory.getProxy();
 * </pre>
 *
 * <h3>Proxy selection (mirrors DefaultAopProxyFactory):</h3>
 * <pre>
 *   proxyTargetClass=true?           → CGLIB
 *   target has interfaces?           → JDK dynamic proxy
 *   target has NO interfaces?        → CGLIB
 * </pre>
 */
public class ProxyFactory {

    private final AdvisedSupport config = new AdvisedSupport();

    public ProxyFactory() {
    }

    /**
     * Create a ProxyFactory for the given target, auto-detecting interfaces.
     *
     * Maps to: {@code ProxyFactory(Object target)} (line 49)
     * which calls {@code ClassUtils.getAllInterfacesForClass()} to detect interfaces.
     */
    public ProxyFactory(Object target) {
        config.setTarget(target);
        // Auto-detect interfaces implemented by the target
        for (Class<?> iface : target.getClass().getInterfaces()) {
            config.addInterface(iface);
        }
    }

    public void setTarget(Object target) {
        config.setTarget(target);
    }

    public void addInterface(Class<?> iface) {
        config.addInterface(iface);
    }

    public void addAdvisor(Advisor advisor) {
        config.addAdvisor(advisor);
    }

    /**
     * Convenience: wraps the advice in a DefaultPointcutAdvisor (matches all).
     */
    public void addAdvice(Advice advice) {
        config.addAdvice(advice);
    }

    public void setProxyTargetClass(boolean proxyTargetClass) {
        config.setProxyTargetClass(proxyTargetClass);
    }

    /**
     * Create the AOP proxy.
     *
     * Maps to: {@code ProxyFactory.getProxy()} (line 96-98)
     * which calls {@code createAopProxy().getProxy()}
     */
    public Object getProxy() {
        return createAopProxy().getProxy();
    }

    /**
     * Select the proxy strategy — the same decision the real framework makes in
     * {@code DefaultAopProxyFactory.createAopProxy()} (line 60-76).
     *
     * Decision logic:
     * <ol>
     *   <li>proxyTargetClass=true → CGLIB (forced class-based proxy)</li>
     *   <li>No interfaces → CGLIB (only option for class-based targets)</li>
     *   <li>Has interfaces → JDK dynamic proxy (preferred — lighter weight)</li>
     * </ol>
     *
     * The real framework also considers: optimize flag, whether the target is
     * itself a Proxy class, whether it's a lambda, etc.
     */
    private AopProxy createAopProxy() {
        if (config.isProxyTargetClass() || config.getInterfaces().isEmpty()) {
            return new CglibAopProxy(config);
        }
        return new JdkDynamicAopProxy(config);
    }

    // Expose config for testing
    AdvisedSupport getConfig() {
        return config;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/Aspect.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as an aspect containing advice methods (@Before, @After, @Around, etc.).
 *
 * Maps to: {@code org.aspectj.lang.annotation.Aspect}
 *
 * IMPORTANT: @Aspect alone does NOT make a class a Spring bean.
 * The class must also be annotated with @Component (or registered manually)
 * for the container to discover it. This is the same behavior as real Spring.
 *
 * <h3>Example:</h3>
 * <pre>
 *   &#64;Aspect
 *   &#64;Component
 *   public class LoggingAspect {
 *       &#64;Before("save*")
 *       public void logBeforeSave() {
 *           System.out.println("About to save...");
 *       }
 *   }
 * </pre>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Aspect {
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/Before.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Before advice — runs before the matched method executes.
 * If the advice method throws an exception, the target method is NOT called.
 *
 * Maps to: {@code org.aspectj.lang.annotation.Before}
 *
 * The value is a method name pattern (simplified from AspectJ pointcut expressions):
 * <ul>
 *   <li>{@code "*"} — matches all methods</li>
 *   <li>{@code "save*"} — methods starting with "save"</li>
 *   <li>{@code "*Data"} — methods ending with "Data"</li>
 *   <li>{@code "doSomething"} — exact match</li>
 * </ul>
 *
 * The advice method can optionally accept a {@code MethodInvocation} parameter
 * for inspecting the joinpoint.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Before {
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/After.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * After (finally) advice — runs after the matched method, regardless of
 * whether it completed normally or threw an exception (like a finally block).
 *
 * Maps to: {@code org.aspectj.lang.annotation.After}
 *
 * The value is a method name pattern (same syntax as {@link Before}).
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface After {
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/Around.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Around advice — the most powerful advice type. The advice method receives
 * a {@link com.simplespringmvc.aop.MethodInvocation} and fully controls
 * whether and when to call {@code proceed()}.
 *
 * Maps to: {@code org.aspectj.lang.annotation.Around}
 *
 * <h3>Example:</h3>
 * <pre>
 *   &#64;Around("*")
 *   public Object measureTime(MethodInvocation invocation) throws Throwable {
 *       long start = System.nanoTime();
 *       Object result = invocation.proceed();
 *       long elapsed = System.nanoTime() - start;
 *       System.out.println(invocation.getMethod().getName() + " took " + elapsed + "ns");
 *       return result;
 *   }
 * </pre>
 *
 * The value is a method name pattern (same syntax as {@link Before}).
 * The advice method MUST accept a {@code MethodInvocation} parameter
 * and SHOULD call {@code invocation.proceed()} to continue the chain.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Around {
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/AfterReturning.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * After-returning advice — runs only when the matched method returns normally
 * (no exception). The advice method can optionally accept the return value
 * as a parameter.
 *
 * Maps to: {@code org.aspectj.lang.annotation.AfterReturning}
 *
 * <h3>Example:</h3>
 * <pre>
 *   &#64;AfterReturning("find*")
 *   public void logResult(Object result) {
 *       System.out.println("Method returned: " + result);
 *   }
 * </pre>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AfterReturning {
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/AfterThrowing.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * After-throwing advice — runs only when the matched method throws an exception.
 * The advice method can optionally accept the thrown exception as a parameter.
 * The exception is always re-thrown after the advice runs.
 *
 * Maps to: {@code org.aspectj.lang.annotation.AfterThrowing}
 *
 * <h3>Example:</h3>
 * <pre>
 *   &#64;AfterThrowing("*")
 *   public void logError(Throwable ex) {
 *       System.err.println("Method threw: " + ex.getMessage());
 *   }
 * </pre>
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AfterThrowing {
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/EnableAspectJAutoProxy.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Enables automatic AOP proxy creation for beans matched by @Aspect advisors.
 * Place on any @Component class (typically a configuration class).
 *
 * Maps to: {@code org.springframework.context.annotation.EnableAspectJAutoProxy}
 *
 * When detected during {@code SimpleBeanContainer.refresh()}, this triggers
 * registration of {@link AspectJAutoProxyCreator} as a BeanPostProcessor.
 *
 * In the real framework, this annotation is meta-annotated with
 * {@code @Import(AspectJAutoProxyRegistrar.class)} which handles
 * the BeanPostProcessor registration via the @Import mechanism.
 *
 * <h3>Example:</h3>
 * <pre>
 *   &#64;Component
 *   &#64;EnableAspectJAutoProxy
 *   public class AppConfig {
 *   }
 * </pre>
 *
 * @see AspectJAutoProxyCreator
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableAspectJAutoProxy {

    /**
     * Whether to create subclass-based (CGLIB) proxies instead of
     * interface-based (JDK) proxies. Default is false (prefer JDK proxies).
     */
    boolean proxyTargetClass() default false;
}
```

#### File: `src/main/java/com/simplespringmvc/aop/aspectj/AspectJAutoProxyCreator.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import com.simplespringmvc.aop.Advice;
import com.simplespringmvc.aop.Advisor;
import com.simplespringmvc.aop.DefaultPointcutAdvisor;
import com.simplespringmvc.aop.MethodInterceptor;
import com.simplespringmvc.aop.MethodInvocation;
import com.simplespringmvc.aop.NamePatternPointcut;
import com.simplespringmvc.aop.Pointcut;
import com.simplespringmvc.aop.ProxyFactory;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.injection.BeanPostProcessor;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * BeanPostProcessor that automatically creates AOP proxies for beans
 * matched by advisor configurations — either programmatic {@link Advisor}
 * beans or declarative @Aspect beans with @Before/@After/@Around methods.
 *
 * Maps to: {@code AnnotationAwareAspectJAutoProxyCreator}
 *   extends AspectJAwareAdvisorAutoProxyCreator
 *   extends AbstractAdvisorAutoProxyCreator
 *   extends AbstractAutoProxyCreator (implements SmartInstantiationAwareBeanPostProcessor)
 *
 * <h3>How it works:</h3>
 * <pre>
 *   For each bean created by the container:
 *     1. postProcessAfterInitialization(bean, beanName)
 *     2. Skip infrastructure beans (aspects, BPPs, advisors)
 *     3. Find all advisors applicable to this bean's class
 *     4. If any apply → create a proxy wrapping the bean
 *     5. Return the proxy (container stores it instead of the original)
 * </pre>
 *
 * <h3>Advisor discovery (two sources):</h3>
 * <ol>
 *   <li><b>Programmatic Advisors:</b> beans implementing {@link Advisor}</li>
 *   <li><b>@Aspect Advisors:</b> @Before/@After/@Around methods on @Aspect beans,
 *       each converted into a {@link DefaultPointcutAdvisor} with a
 *       {@link NamePatternPointcut}</li>
 * </ol>
 *
 * <h3>Key simplification vs real Spring:</h3>
 * <ul>
 *   <li>No AspectJ expression pointcuts — uses method name patterns</li>
 *   <li>No advisor ordering (@Order / PriorityOrdered)</li>
 *   <li>No early proxy reference (three-level singleton cache)</li>
 *   <li>No proxy class caching</li>
 * </ul>
 */
public class AspectJAutoProxyCreator implements BeanPostProcessor {

    private final SimpleBeanContainer container;
    private boolean proxyTargetClass = false;

    /**
     * Lazily-built list of all advisors discovered in the container.
     * Built on first call to postProcessAfterInitialization() and cached.
     */
    private List<Advisor> cachedAdvisors;

    /**
     * Tracks beans currently being post-processed to avoid infinite recursion
     * when advisor discovery triggers creation of other beans.
     */
    private final Set<String> currentlyProxying = new HashSet<>();

    public AspectJAutoProxyCreator(SimpleBeanContainer container) {
        this.container = container;
    }

    public void setProxyTargetClass(boolean proxyTargetClass) {
        this.proxyTargetClass = proxyTargetClass;
    }

    /**
     * The integration point — called by SimpleBeanContainer.createBean()
     * for every bean after construction and @Autowired injection.
     *
     * Maps to: {@code AbstractAutoProxyCreator.postProcessAfterInitialization()}
     * → {@code wrapIfNecessary()} (line 338-380)
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Don't proxy infrastructure beans
        if (isInfrastructureClass(bean.getClass())) {
            return bean;
        }

        // Prevent re-entrance during advisor discovery
        if (currentlyProxying.contains(beanName)) {
            return bean;
        }

        currentlyProxying.add(beanName);
        try {
            List<Advisor> advisors = findApplicableAdvisors(bean.getClass());
            if (advisors.isEmpty()) {
                return bean;
            }
            return createProxy(bean, advisors);
        } finally {
            currentlyProxying.remove(beanName);
        }
    }

    /**
     * Infrastructure classes should never be proxied — they ARE the AOP infrastructure.
     *
     * Maps to: {@code AbstractAutoProxyCreator.isInfrastructureClass()} (line 410-420)
     * + {@code AnnotationAwareAspectJAutoProxyCreator.isInfrastructureClass()} (line 99-110)
     */
    private boolean isInfrastructureClass(Class<?> beanClass) {
        return Advisor.class.isAssignableFrom(beanClass)
                || Advice.class.isAssignableFrom(beanClass)
                || BeanPostProcessor.class.isAssignableFrom(beanClass)
                || beanClass.isAnnotationPresent(Aspect.class);
    }

    // ─── Advisor discovery ──────────────────────────────────

    /**
     * Find advisors whose pointcut matches at least one method of the target class.
     *
     * Maps to: {@code AbstractAdvisorAutoProxyCreator.findEligibleAdvisors()} (line 96-110)
     * which calls findCandidateAdvisors(), findAdvisorsThatCanApply(), extendAdvisors()
     */
    private List<Advisor> findApplicableAdvisors(Class<?> targetClass) {
        List<Advisor> allAdvisors = getAllAdvisors();
        List<Advisor> applicable = new ArrayList<>();

        for (Advisor advisor : allAdvisors) {
            if (hasMatchingMethod(advisor.getPointcut(), targetClass)) {
                applicable.add(advisor);
            }
        }
        return applicable;
    }

    /**
     * Check if any public method of the target class matches the pointcut.
     *
     * Maps to: {@code AopUtils.canApply(Pointcut, Class, boolean)} (line 225-269)
     */
    private boolean hasMatchingMethod(Pointcut pointcut, Class<?> targetClass) {
        for (Method method : targetClass.getMethods()) {
            if (method.getDeclaringClass() == Object.class) {
                continue;
            }
            if (pointcut.matches(method, targetClass)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Discover all advisors in the container — both programmatic Advisor beans
     * and advisors derived from @Aspect beans.
     *
     * Maps to: {@code AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors()} (line 88-96)
     * which calls super.findCandidateAdvisors() + aspectJAdvisorsBuilder.buildAspectJAdvisors()
     */
    private List<Advisor> getAllAdvisors() {
        if (cachedAdvisors != null) {
            return cachedAdvisors;
        }

        // Initialize to empty list to prevent re-entrance during discovery
        cachedAdvisors = new ArrayList<>();

        // 1. Find programmatic Advisor beans
        for (String name : container.getBeanNames()) {
            Class<?> beanClass = container.getType(name);
            if (beanClass != null && Advisor.class.isAssignableFrom(beanClass)
                    && !currentlyProxying.contains(name)) {
                try {
                    Object bean = container.getBean(name);
                    if (bean instanceof Advisor advisor) {
                        cachedAdvisors.add(advisor);
                    }
                } catch (Exception e) {
                    // Skip beans that can't be retrieved
                }
            }
        }

        // 2. Find @Aspect beans and build advisors from annotated methods
        for (String name : container.getBeanNames()) {
            Class<?> beanClass = container.getType(name);
            if (beanClass != null && beanClass.isAnnotationPresent(Aspect.class)
                    && !currentlyProxying.contains(name)) {
                try {
                    Object aspect = container.getBean(name);
                    cachedAdvisors.addAll(buildAdvisorsFromAspect(aspect));
                } catch (Exception e) {
                    // Skip aspects that can't be retrieved
                }
            }
        }

        return cachedAdvisors;
    }

    // ─── @Aspect to Advisor conversion ──────────────────────

    /**
     * Convert an @Aspect bean's annotated methods into Advisor objects.
     *
     * Maps to: {@code ReflectiveAspectJAdvisorFactory.getAdvisors()} (line 132)
     * which iterates the aspect's methods and calls getAdvisor() for each.
     */
    private List<Advisor> buildAdvisorsFromAspect(Object aspect) {
        List<Advisor> advisors = new ArrayList<>();

        for (Method method : aspect.getClass().getDeclaredMethods()) {
            Advisor advisor = buildAdvisor(aspect, method);
            if (advisor != null) {
                advisors.add(advisor);
            }
        }

        return advisors;
    }

    /**
     * Convert a single @Before/@After/@Around/@AfterReturning/@AfterThrowing method
     * into an Advisor (MethodInterceptor + Pointcut).
     *
     * Maps to: {@code ReflectiveAspectJAdvisorFactory.getAdvisor()} (line 192)
     * + InstantiationModelAwarePointcutAdvisorImpl
     *
     * Each annotation type is converted into a MethodInterceptor with the
     * appropriate execution semantics:
     * - @Before: invoke advice, then proceed
     * - @After: proceed in try-finally, invoke advice in finally
     * - @Around: advice controls proceed() directly
     * - @AfterReturning: proceed, then invoke advice with result
     * - @AfterThrowing: proceed in try-catch, invoke advice on exception
     */
    private Advisor buildAdvisor(Object aspect, Method adviceMethod) {
        Before before = adviceMethod.getAnnotation(Before.class);
        if (before != null) {
            MethodInterceptor interceptor = invocation -> {
                invokeAdviceMethod(aspect, adviceMethod, invocation, null, null);
                return invocation.proceed();
            };
            return new DefaultPointcutAdvisor(
                    new NamePatternPointcut(before.value()), interceptor);
        }

        After after = adviceMethod.getAnnotation(After.class);
        if (after != null) {
            MethodInterceptor interceptor = invocation -> {
                try {
                    return invocation.proceed();
                } finally {
                    invokeAdviceMethod(aspect, adviceMethod, invocation, null, null);
                }
            };
            return new DefaultPointcutAdvisor(
                    new NamePatternPointcut(after.value()), interceptor);
        }

        Around around = adviceMethod.getAnnotation(Around.class);
        if (around != null) {
            MethodInterceptor interceptor = invocation -> {
                adviceMethod.setAccessible(true);
                try {
                    return adviceMethod.invoke(aspect, invocation);
                } catch (InvocationTargetException ex) {
                    throw ex.getTargetException();
                }
            };
            return new DefaultPointcutAdvisor(
                    new NamePatternPointcut(around.value()), interceptor);
        }

        AfterReturning afterReturning = adviceMethod.getAnnotation(AfterReturning.class);
        if (afterReturning != null) {
            MethodInterceptor interceptor = invocation -> {
                Object result = invocation.proceed();
                invokeAdviceMethod(aspect, adviceMethod, invocation, result, null);
                return result;
            };
            return new DefaultPointcutAdvisor(
                    new NamePatternPointcut(afterReturning.value()), interceptor);
        }

        AfterThrowing afterThrowing = adviceMethod.getAnnotation(AfterThrowing.class);
        if (afterThrowing != null) {
            MethodInterceptor interceptor = invocation -> {
                try {
                    return invocation.proceed();
                } catch (Throwable ex) {
                    invokeAdviceMethod(aspect, adviceMethod, invocation, null, ex);
                    throw ex;
                }
            };
            return new DefaultPointcutAdvisor(
                    new NamePatternPointcut(afterThrowing.value()), interceptor);
        }

        return null;
    }

    /**
     * Invoke the advice method, matching parameters by type.
     *
     * Maps to: {@code AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs()} (line 633)
     * + {@code argBinding()} (line 556) for parameter resolution.
     *
     * Supported parameter types:
     * <ul>
     *   <li>{@link MethodInvocation} — the current joinpoint</li>
     *   <li>{@link Throwable} (subclass) — the thrown exception (@AfterThrowing)</li>
     *   <li>Any other type — the return value (@AfterReturning)</li>
     * </ul>
     */
    private static void invokeAdviceMethod(Object aspect, Method adviceMethod,
                                           MethodInvocation invocation,
                                           Object returnValue, Throwable exception) throws Throwable {
        adviceMethod.setAccessible(true);
        Class<?>[] paramTypes = adviceMethod.getParameterTypes();

        try {
            if (paramTypes.length == 0) {
                adviceMethod.invoke(aspect);
            } else {
                Object[] args = new Object[paramTypes.length];
                for (int i = 0; i < paramTypes.length; i++) {
                    if (MethodInvocation.class.isAssignableFrom(paramTypes[i])) {
                        args[i] = invocation;
                    } else if (Throwable.class.isAssignableFrom(paramTypes[i])) {
                        args[i] = exception;
                    } else {
                        args[i] = returnValue;
                    }
                }
                adviceMethod.invoke(aspect, args);
            }
        } catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }

    // ─── Proxy creation ─────────────────────────────────────

    /**
     * Create a proxy wrapping the target bean with the given advisors.
     *
     * Maps to: {@code AbstractAutoProxyCreator.createProxy()} (line 425-493)
     * which creates a ProxyFactory, configures it, and calls getProxy().
     */
    private Object createProxy(Object target, List<Advisor> advisors) {
        ProxyFactory factory = new ProxyFactory();
        factory.setTarget(target);

        // Add interfaces from the target for JDK proxy
        for (Class<?> iface : target.getClass().getInterfaces()) {
            factory.addInterface(iface);
        }

        // Add all applicable advisors
        for (Advisor advisor : advisors) {
            factory.addAdvisor(advisor);
        }

        factory.setProxyTargetClass(proxyTargetClass);

        return factory.getProxy();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java` [MODIFIED]

Key additions (see section 28.1 above for explanation):
- New imports: `AspectJAutoProxyCreator`, `EnableAspectJAutoProxy`
- New method: `getType(String beanName)` -- returns bean class without forcing instantiation
- New method: `autoRegisterAopProcessorIfNeeded()` -- detects `@EnableAspectJAutoProxy` and registers BPP
- Modified `refresh()`: calls `autoRegisterAopProcessorIfNeeded()` after autowired processor registration

#### File: `build.gradle` [MODIFIED]

Added ByteBuddy dependency for CGLIB-style proxy generation:

```groovy
// ByteBuddy — ch28: CGLIB-style subclass proxy generation for AOP
// The real Spring uses CGLIB (repackaged in spring-core). ByteBuddy is the modern
// bytecode generation library — same concept (runtime subclass creation), cleaner API.
implementation 'net.bytebuddy:byte-buddy:1.14.18'
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/aop/PointcutTest.java` [NEW]

```java
package com.simplespringmvc.aop;

import org.junit.jupiter.api.Test;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link NamePatternPointcut} and {@link AnnotationPointcut}.
 */
class PointcutTest {

    // ─── NamePatternPointcut ─────────────────────────────────

    @Test
    void shouldMatchAllMethods_WhenPatternIsStar() throws Exception {
        var pointcut = new NamePatternPointcut("*");
        Method method = SampleService.class.getMethod("saveUser");
        assertThat(pointcut.matches(method, SampleService.class)).isTrue();
    }

    @Test
    void shouldMatchPrefix_WhenPatternEndsWithStar() throws Exception {
        var pointcut = new NamePatternPointcut("save*");
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveUser"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveOrder"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("findUser"), SampleService.class)).isFalse();
    }

    @Test
    void shouldMatchSuffix_WhenPatternStartsWithStar() throws Exception {
        var pointcut = new NamePatternPointcut("*User");
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveUser"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("findUser"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveOrder"), SampleService.class)).isFalse();
    }

    @Test
    void shouldMatchContains_WhenPatternHasStarOnBothEnds() throws Exception {
        var pointcut = new NamePatternPointcut("*User*");
        // "saveUser" doesn't contain "User" at a position that leaves chars on both sides?
        // Actually *User* means methodName.contains("User")
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveUser"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("findUser"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveOrder"), SampleService.class)).isFalse();
    }

    @Test
    void shouldMatchExact_WhenPatternHasNoStar() throws Exception {
        var pointcut = new NamePatternPointcut("saveUser");
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveUser"), SampleService.class)).isTrue();
        assertThat(pointcut.matches(
                SampleService.class.getMethod("saveOrder"), SampleService.class)).isFalse();
    }

    // ─── AnnotationPointcut ──────────────────────────────────

    @Test
    void shouldMatchAnnotatedMethod_WhenAnnotationPresent() throws Exception {
        var pointcut = new AnnotationPointcut(Logged.class);
        assertThat(pointcut.matches(
                AnnotatedService.class.getMethod("trackedMethod"), AnnotatedService.class)).isTrue();
    }

    @Test
    void shouldNotMatchUnannotatedMethod_WhenAnnotationAbsent() throws Exception {
        var pointcut = new AnnotationPointcut(Logged.class);
        assertThat(pointcut.matches(
                AnnotatedService.class.getMethod("normalMethod"), AnnotatedService.class)).isFalse();
    }

    @Test
    void shouldMatchAnnotationOnImplMethod_WhenCalledViaInterface() throws Exception {
        var pointcut = new AnnotationPointcut(Logged.class);
        // The method from the interface doesn't have the annotation,
        // but the implementation class method does
        Method interfaceMethod = Greeter.class.getMethod("greet");
        assertThat(pointcut.matches(interfaceMethod, AnnotatedGreeter.class)).isTrue();
    }

    // ─── Pointcut.TRUE ───────────────────────────────────────

    @Test
    void shouldAlwaysMatch_WhenUsingPointcutTrue() throws Exception {
        Method method = SampleService.class.getMethod("saveUser");
        assertThat(Pointcut.TRUE.matches(method, SampleService.class)).isTrue();
    }

    // ─── Test fixtures ───────────────────────────────────────

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Logged {
    }

    public static class SampleService {
        public void saveUser() {}
        public void saveOrder() {}
        public void findUser() {}
    }

    public static class AnnotatedService {
        @Logged
        public void trackedMethod() {}
        public void normalMethod() {}
    }

    public interface Greeter {
        String greet();
    }

    public static class AnnotatedGreeter implements Greeter {
        @Logged
        @Override
        public String greet() { return "hello"; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/aop/ReflectiveMethodInvocationTest.java` [NEW]

```java
package com.simplespringmvc.aop;

import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link ReflectiveMethodInvocation} — the Chain of Responsibility engine.
 */
class ReflectiveMethodInvocationTest {

    @Test
    void shouldInvokeTargetDirectly_WhenNoInterceptors() throws Throwable {
        var target = new Calculator();
        Method method = Calculator.class.getMethod("add", int.class, int.class);

        var invocation = new ReflectiveMethodInvocation(
                null, target, method, new Object[]{3, 4}, List.of());

        Object result = invocation.proceed();
        assertThat(result).isEqualTo(7);
    }

    @Test
    void shouldWalkChainInOrder_WhenMultipleInterceptors() throws Throwable {
        var target = new Calculator();
        Method method = Calculator.class.getMethod("add", int.class, int.class);
        List<String> callOrder = new ArrayList<>();

        MethodInterceptor first = inv -> {
            callOrder.add("first-before");
            Object result = inv.proceed();
            callOrder.add("first-after");
            return result;
        };
        MethodInterceptor second = inv -> {
            callOrder.add("second-before");
            Object result = inv.proceed();
            callOrder.add("second-after");
            return result;
        };

        var invocation = new ReflectiveMethodInvocation(
                null, target, method, new Object[]{3, 4}, List.of(first, second));

        Object result = invocation.proceed();

        assertThat(result).isEqualTo(7);
        assertThat(callOrder).containsExactly(
                "first-before", "second-before", "second-after", "first-after");
    }

    @Test
    void shouldExposeMethodAndArguments_WhenInterceptorInspects() throws Throwable {
        var target = new Calculator();
        Method method = Calculator.class.getMethod("add", int.class, int.class);

        MethodInterceptor inspector = inv -> {
            assertThat(inv.getMethod().getName()).isEqualTo("add");
            assertThat(inv.getArguments()).containsExactly(3, 4);
            assertThat(inv.getThis()).isSameAs(target);
            return inv.proceed();
        };

        var invocation = new ReflectiveMethodInvocation(
                null, target, method, new Object[]{3, 4}, List.of(inspector));
        invocation.proceed();
    }

    @Test
    void shouldShortCircuit_WhenInterceptorDoesNotCallProceed() throws Throwable {
        var target = new Calculator();
        Method method = Calculator.class.getMethod("add", int.class, int.class);

        MethodInterceptor shortCircuit = inv -> -1; // Never calls proceed()

        var invocation = new ReflectiveMethodInvocation(
                null, target, method, new Object[]{3, 4}, List.of(shortCircuit));

        Object result = invocation.proceed();
        assertThat(result).isEqualTo(-1);
    }

    @Test
    void shouldPropagateTargetException_WhenTargetThrows() {
        var target = new Calculator();
        Method method;
        try {
            method = Calculator.class.getMethod("divide", int.class, int.class);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        }

        var invocation = new ReflectiveMethodInvocation(
                null, target, method, new Object[]{1, 0}, List.of());

        assertThatThrownBy(invocation::proceed)
                .isInstanceOf(ArithmeticException.class)
                .hasMessage("/ by zero");
    }

    // ─── Test fixture ────────────────────────────────────────

    public static class Calculator {
        public int add(int a, int b) { return a + b; }
        public int divide(int a, int b) { return a / b; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/aop/JdkDynamicAopProxyTest.java` [NEW]

```java
package com.simplespringmvc.aop;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link JdkDynamicAopProxy} — interface-based proxying.
 */
class JdkDynamicAopProxyTest {

    @Test
    void shouldDelegateToTarget_WhenNoAdvisors() {
        var config = new AdvisedSupport();
        config.setTarget(new SimpleService());
        config.addInterface(Service.class);

        var proxy = (Service) new JdkDynamicAopProxy(config).getProxy();

        assertThat(proxy.echo("hello")).isEqualTo("hello");
    }

    @Test
    void shouldApplyInterceptor_WhenAdvisorMatches() {
        var config = new AdvisedSupport();
        config.setTarget(new SimpleService());
        config.addInterface(Service.class);
        config.addAdvisor(new DefaultPointcutAdvisor(
                (MethodInterceptor) inv -> "intercepted:" + inv.proceed()
        ));

        var proxy = (Service) new JdkDynamicAopProxy(config).getProxy();

        assertThat(proxy.echo("test")).isEqualTo("intercepted:test");
    }

    @Test
    void shouldImplementSpecifiedInterfaces_WhenProxyCreated() {
        var config = new AdvisedSupport();
        config.setTarget(new SimpleService());
        config.addInterface(Service.class);

        Object proxy = new JdkDynamicAopProxy(config).getProxy();

        assertThat(proxy).isInstanceOf(Service.class);
        // JDK proxy is NOT an instance of the target class
        assertThat(proxy).isNotInstanceOf(SimpleService.class);
    }

    @Test
    void shouldHandleFluentApi_WhenTargetReturnsThis() {
        var target = new FluentService();
        var config = new AdvisedSupport();
        config.setTarget(target);
        config.addInterface(Fluent.class);
        config.addAdvice((MethodInterceptor) MethodInvocation::proceed);

        var proxy = (Fluent) new JdkDynamicAopProxy(config).getProxy();

        // The target returns "this" (the target), but the proxy should
        // return itself (the proxy) so the fluent chain continues through the proxy
        Fluent result = proxy.doSomething();
        assertThat(result).isSameAs(proxy);
    }

    @Test
    void shouldPropagateException_WhenTargetThrows() {
        var config = new AdvisedSupport();
        config.setTarget(new SimpleService());
        config.addInterface(Service.class);

        var proxy = (Service) new JdkDynamicAopProxy(config).getProxy();

        assertThatThrownBy(() -> proxy.fail())
                .isInstanceOf(RuntimeException.class)
                .hasMessage("boom");
    }

    // ─── Test fixtures ───────────────────────────────────────

    public interface Service {
        String echo(String msg);
        void fail();
    }

    public interface Fluent {
        Fluent doSomething();
    }

    public static class SimpleService implements Service {
        @Override
        public String echo(String msg) { return msg; }

        @Override
        public void fail() { throw new RuntimeException("boom"); }
    }

    public static class FluentService implements Fluent {
        @Override
        public Fluent doSomething() { return this; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/aop/CglibAopProxyTest.java` [NEW]

```java
package com.simplespringmvc.aop;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link CglibAopProxy} — subclass-based proxying via ByteBuddy.
 */
class CglibAopProxyTest {

    @Test
    void shouldDelegateToTarget_WhenNoAdvisors() {
        var config = new AdvisedSupport();
        config.setTarget(new Calculator());

        var proxy = (Calculator) new CglibAopProxy(config).getProxy();

        assertThat(proxy.add(3, 4)).isEqualTo(7);
    }

    @Test
    void shouldApplyInterceptor_WhenAdvisorMatches() {
        var config = new AdvisedSupport();
        config.setTarget(new Calculator());
        config.addAdvisor(new DefaultPointcutAdvisor(
                (MethodInterceptor) inv -> ((int) inv.proceed()) * 2
        ));

        var proxy = (Calculator) new CglibAopProxy(config).getProxy();

        assertThat(proxy.add(3, 4)).isEqualTo(14); // (3+4)*2
    }

    @Test
    void shouldBeInstanceOfTargetClass_WhenProxyCreated() {
        var config = new AdvisedSupport();
        config.setTarget(new Calculator());

        Object proxy = new CglibAopProxy(config).getProxy();

        // CGLIB proxy IS a subclass of the target
        assertThat(proxy).isInstanceOf(Calculator.class);
    }

    @Test
    void shouldApplyPointcutFiltering_WhenMethodNameMatches() {
        var config = new AdvisedSupport();
        config.setTarget(new Calculator());

        List<String> intercepted = new ArrayList<>();
        config.addAdvisor(new DefaultPointcutAdvisor(
                new NamePatternPointcut("add"),
                (MethodInterceptor) inv -> {
                    intercepted.add(inv.getMethod().getName());
                    return inv.proceed();
                }
        ));

        var proxy = (Calculator) new CglibAopProxy(config).getProxy();
        proxy.add(1, 2);
        proxy.multiply(3, 4);

        // Only "add" should be intercepted, not "multiply"
        assertThat(intercepted).containsExactly("add");
    }

    @Test
    void shouldPropagateException_WhenTargetThrows() {
        var config = new AdvisedSupport();
        config.setTarget(new Calculator());

        var proxy = (Calculator) new CglibAopProxy(config).getProxy();

        assertThatThrownBy(() -> proxy.divide(1, 0))
                .isInstanceOf(ArithmeticException.class);
    }

    @Test
    void shouldThrowClearError_WhenNoDefaultConstructor() {
        var config = new AdvisedSupport();
        config.setTarget(new NoDefaultCtor("test"));

        assertThatThrownBy(() -> new CglibAopProxy(config).getProxy())
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("no default constructor");
    }

    // ─── Test fixtures ───────────────────────────────────────

    public static class Calculator {
        public int add(int a, int b) { return a + b; }
        public int multiply(int a, int b) { return a * b; }
        public int divide(int a, int b) { return a / b; }
    }

    public static class NoDefaultCtor {
        public NoDefaultCtor(String required) {}
    }
}
```

#### File: `src/test/java/com/simplespringmvc/aop/ProxyFactoryTest.java` [NEW]

```java
package com.simplespringmvc.aop;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link ProxyFactory} — end-to-end proxy creation and advice application.
 */
class ProxyFactoryTest {

    // ─── JDK proxy (interface-based) ─────────────────────────

    @Test
    void shouldCreateJdkProxy_WhenTargetImplementsInterfaces() {
        var target = new SimpleGreetingService();
        var factory = new ProxyFactory(target);
        factory.addAdvice((MethodInterceptor) invocation -> {
            return "Intercepted: " + invocation.proceed();
        });

        Object proxy = factory.getProxy();

        assertThat(proxy).isInstanceOf(GreetingService.class);
        assertThat(proxy).isNotInstanceOf(SimpleGreetingService.class);
        assertThat(((GreetingService) proxy).greet("World"))
                .isEqualTo("Intercepted: Hello, World");
    }

    @Test
    void shouldApplyMultipleAdvice_WhenMultipleInterceptorsAdded() {
        var target = new SimpleGreetingService();
        var factory = new ProxyFactory(target);

        List<String> callOrder = new ArrayList<>();
        factory.addAdvice((MethodInterceptor) invocation -> {
            callOrder.add("first");
            return invocation.proceed();
        });
        factory.addAdvice((MethodInterceptor) invocation -> {
            callOrder.add("second");
            return invocation.proceed();
        });

        var proxy = (GreetingService) factory.getProxy();
        proxy.greet("Test");

        assertThat(callOrder).containsExactly("first", "second");
    }

    @Test
    void shouldApplyPointcutFiltering_WhenAdvisorHasPointcut() {
        var target = new SimpleGreetingService();
        var factory = new ProxyFactory(target);

        List<String> intercepted = new ArrayList<>();
        factory.addAdvisor(new DefaultPointcutAdvisor(
                new NamePatternPointcut("greet"),
                (MethodInterceptor) invocation -> {
                    intercepted.add(invocation.getMethod().getName());
                    return invocation.proceed();
                }
        ));

        var proxy = (GreetingService) factory.getProxy();
        proxy.greet("Test");

        assertThat(intercepted).containsExactly("greet");
    }

    @Test
    void shouldNotIntercept_WhenPointcutDoesNotMatch() {
        var target = new SimpleGreetingService();
        var factory = new ProxyFactory(target);

        List<String> intercepted = new ArrayList<>();
        factory.addAdvisor(new DefaultPointcutAdvisor(
                new NamePatternPointcut("save*"),
                (MethodInterceptor) invocation -> {
                    intercepted.add(invocation.getMethod().getName());
                    return invocation.proceed();
                }
        ));

        var proxy = (GreetingService) factory.getProxy();
        proxy.greet("Test");

        assertThat(intercepted).isEmpty();
    }

    // ─── CGLIB proxy (class-based) ───────────────────────────

    @Test
    void shouldCreateCglibProxy_WhenProxyTargetClassIsTrue() {
        var target = new SimpleGreetingService();
        var factory = new ProxyFactory(target);
        factory.setProxyTargetClass(true);
        factory.addAdvice((MethodInterceptor) invocation -> {
            return "CGLIB: " + invocation.proceed();
        });

        Object proxy = factory.getProxy();

        // CGLIB proxy IS a subclass of the target
        assertThat(proxy).isInstanceOf(SimpleGreetingService.class);
        assertThat(((GreetingService) proxy).greet("World"))
                .isEqualTo("CGLIB: Hello, World");
    }

    @Test
    void shouldCreateCglibProxy_WhenTargetHasNoInterfaces() {
        var target = new Calculator();
        var factory = new ProxyFactory();
        factory.setTarget(target);
        factory.addAdvice((MethodInterceptor) invocation -> {
            return ((int) invocation.proceed()) * 10;
        });

        Object proxy = factory.getProxy();

        assertThat(proxy).isInstanceOf(Calculator.class);
        assertThat(((Calculator) proxy).add(3, 4)).isEqualTo(70);
    }

    // ─── Test fixtures ───────────────────────────────────────

    public interface GreetingService {
        String greet(String name);
    }

    public static class SimpleGreetingService implements GreetingService {
        @Override
        public String greet(String name) {
            return "Hello, " + name;
        }
    }

    public static class Calculator {
        public int add(int a, int b) { return a + b; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/aop/aspectj/AspectJAutoProxyCreatorTest.java` [NEW]

```java
package com.simplespringmvc.aop.aspectj;

import com.simplespringmvc.aop.Advisor;
import com.simplespringmvc.aop.DefaultPointcutAdvisor;
import com.simplespringmvc.aop.MethodInterceptor;
import com.simplespringmvc.aop.MethodInvocation;
import com.simplespringmvc.aop.NamePatternPointcut;
import com.simplespringmvc.container.SimpleBeanContainer;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link AspectJAutoProxyCreator} — the BeanPostProcessor that
 * auto-proxies beans matched by @Aspect advisors.
 */
class AspectJAutoProxyCreatorTest {

    // ─── @Before advice ──────────────────────────────────────

    @Test
    void shouldApplyBeforeAdvice_WhenAspectBeanRegistered() {
        var container = new SimpleBeanContainer();
        var aspect = new LoggingAspect();

        container.registerBean("loggingAspect", aspect);
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);

        // Register config with @EnableAspectJAutoProxy
        container.registerBean("config", new AopConfig());
        container.refresh();

        var service = (GreetingService) container.getBean("greetingService");
        String result = service.greet("World");

        assertThat(result).isEqualTo("Hello, World");
        assertThat(aspect.logs).contains("before:greet");
    }

    // ─── @After advice ───────────────────────────────────────

    @Test
    void shouldApplyAfterAdvice_WhenAspectBeanRegistered() {
        var container = new SimpleBeanContainer();
        var aspect = new AfterAspect();

        container.registerBean("afterAspect", aspect);
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);
        container.registerBean("config", new AopConfig());
        container.refresh();

        var service = (GreetingService) container.getBean("greetingService");
        service.greet("World");

        assertThat(aspect.logs).contains("after:greet");
    }

    // ─── @Around advice ──────────────────────────────────────

    @Test
    void shouldApplyAroundAdvice_WhenAspectBeanRegistered() {
        var container = new SimpleBeanContainer();
        container.registerBean("timingAspect", new TimingAspect());
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);
        container.registerBean("config", new AopConfig());
        container.refresh();

        var service = (GreetingService) container.getBean("greetingService");
        String result = service.greet("World");

        assertThat(result).isEqualTo("[TIMED] Hello, World");
    }

    // ─── @AfterReturning advice ──────────────────────────────

    @Test
    void shouldApplyAfterReturningAdvice_WhenMethodReturnsNormally() {
        var container = new SimpleBeanContainer();
        var aspect = new AfterReturningAspect();

        container.registerBean("arAspect", aspect);
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);
        container.registerBean("config", new AopConfig());
        container.refresh();

        var service = (GreetingService) container.getBean("greetingService");
        service.greet("Test");

        assertThat(aspect.returnValues).contains("Hello, Test");
    }

    // ─── @AfterThrowing advice ───────────────────────────────

    @Test
    void shouldApplyAfterThrowingAdvice_WhenMethodThrows() {
        var container = new SimpleBeanContainer();
        var aspect = new AfterThrowingAspect();

        container.registerBean("atAspect", aspect);
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);
        container.registerBean("config", new AopConfig());
        container.refresh();

        var service = (GreetingService) container.getBean("greetingService");
        try {
            service.fail();
        } catch (RuntimeException e) {
            // expected
        }

        assertThat(aspect.exceptions).hasSize(1);
        assertThat(aspect.exceptions.get(0)).hasMessage("boom");
    }

    // ─── Programmatic Advisor beans ──────────────────────────

    @Test
    void shouldApplyProgrammaticAdvisor_WhenAdvisorBeanRegistered() {
        var container = new SimpleBeanContainer();
        List<String> log = new ArrayList<>();

        Advisor advisor = new DefaultPointcutAdvisor(
                new NamePatternPointcut("greet"),
                (MethodInterceptor) inv -> {
                    log.add("advised:" + inv.getMethod().getName());
                    return inv.proceed();
                }
        );

        container.registerBean("loggingAdvisor", advisor);
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);
        container.registerBean("config", new AopConfig());
        container.refresh();

        var service = (GreetingService) container.getBean("greetingService");
        service.greet("Test");

        assertThat(log).contains("advised:greet");
    }

    // ─── Infrastructure class protection ─────────────────────

    @Test
    void shouldNotProxyAspectBean_WhenAspectClassDetected() {
        var container = new SimpleBeanContainer();
        var aspect = new LoggingAspect();

        container.registerBean("loggingAspect", aspect);
        container.registerBeanDefinition("greetingService", SimpleGreetingService.class);
        container.registerBean("config", new AopConfig());
        container.refresh();

        // The aspect bean itself should NOT be proxied
        Object retrievedAspect = container.getBean("loggingAspect");
        assertThat(retrievedAspect).isSameAs(aspect);
    }

    // ─── Test fixtures ───────────────────────────────────────

    @EnableAspectJAutoProxy
    public static class AopConfig {
    }

    public interface GreetingService {
        String greet(String name);
        void fail();
    }

    public static class SimpleGreetingService implements GreetingService {
        @Override
        public String greet(String name) { return "Hello, " + name; }

        @Override
        public void fail() { throw new RuntimeException("boom"); }
    }

    @Aspect
    public static class LoggingAspect {
        final List<String> logs = new ArrayList<>();

        @Before("greet")
        public void logBefore() {
            logs.add("before:greet");
        }
    }

    @Aspect
    public static class AfterAspect {
        final List<String> logs = new ArrayList<>();

        @After("greet")
        public void logAfter() {
            logs.add("after:greet");
        }
    }

    @Aspect
    public static class TimingAspect {
        @Around("greet")
        public Object measureTime(MethodInvocation invocation) throws Throwable {
            Object result = invocation.proceed();
            return "[TIMED] " + result;
        }
    }

    @Aspect
    public static class AfterReturningAspect {
        final List<Object> returnValues = new ArrayList<>();

        @AfterReturning("greet")
        public void captureReturn(Object result) {
            returnValues.add(result);
        }
    }

    @Aspect
    public static class AfterThrowingAspect {
        final List<Throwable> exceptions = new ArrayList<>();

        @AfterThrowing("fail")
        public void captureError(Throwable ex) {
            exceptions.add(ex);
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/AopIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Component;
import com.simplespringmvc.aop.Advisor;
import com.simplespringmvc.aop.DefaultPointcutAdvisor;
import com.simplespringmvc.aop.MethodInterceptor;
import com.simplespringmvc.aop.MethodInvocation;
import com.simplespringmvc.aop.NamePatternPointcut;
import com.simplespringmvc.aop.aspectj.Around;
import com.simplespringmvc.aop.aspectj.Aspect;
import com.simplespringmvc.aop.aspectj.Before;
import com.simplespringmvc.aop.aspectj.EnableAspectJAutoProxy;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.scan.ClasspathScanner;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for AOP with the bean container and component scanning.
 * Tests that all the pieces work together: scanning discovers @Aspect beans,
 * @EnableAspectJAutoProxy triggers the auto-proxy creator, and beans
 * are correctly proxied during container refresh.
 */
class AopIntegrationTest {

    // ─── Manual registration + AOP ───────────────────────────

    @Test
    void shouldProxyBean_WhenContainerRefreshWithAspectAndEnableAop() {
        var container = new SimpleBeanContainer();
        List<String> log = new ArrayList<>();

        // Register @EnableAspectJAutoProxy config
        container.registerBean("config", new AopEnabledConfig());

        // Register @Aspect bean
        container.registerBean("loggingAspect", new TestLoggingAspect(log));

        // Register target bean as definition (goes through BPP lifecycle)
        container.registerBeanDefinition("userService", SimpleUserService.class);

        container.refresh();

        var service = container.getBean(UserService.class);
        String result = service.findUser("alice");

        assertThat(result).isEqualTo("User: alice");
        assertThat(log).contains("before:findUser");
    }

    @Test
    void shouldProxyMultipleBeans_WhenMultipleMatchingBeans() {
        var container = new SimpleBeanContainer();
        List<String> log = new ArrayList<>();

        container.registerBean("config", new AopEnabledConfig());
        container.registerBean("loggingAspect", new AllMethodAspect(log));
        container.registerBeanDefinition("userService", SimpleUserService.class);
        container.registerBeanDefinition("orderService", SimpleOrderService.class);

        container.refresh();

        var userService = container.getBean(UserService.class);
        var orderService = container.getBean(OrderService.class);

        userService.findUser("alice");
        orderService.createOrder("item1");

        // Both services should be intercepted
        assertThat(log).contains("around:findUser", "around:createOrder");
    }

    @Test
    void shouldNotProxy_WhenNoMatchingAdvice() {
        var container = new SimpleBeanContainer();
        List<String> log = new ArrayList<>();

        container.registerBean("config", new AopEnabledConfig());
        // Aspect only matches "save*" methods
        container.registerBean("saveAspect", new SaveOnlyAspect(log));
        container.registerBeanDefinition("userService", SimpleUserService.class);

        container.refresh();

        var service = container.getBean(UserService.class);
        service.findUser("alice");

        // findUser should NOT be intercepted (doesn't match "save*")
        assertThat(log).isEmpty();
    }

    @Test
    void shouldWorkWithProgrammaticAdvisors_WhenAdvisorBeanRegistered() {
        var container = new SimpleBeanContainer();
        List<String> log = new ArrayList<>();

        container.registerBean("config", new AopEnabledConfig());
        container.registerBean("myAdvisor", new DefaultPointcutAdvisor(
                new NamePatternPointcut("find*"),
                (MethodInterceptor) inv -> {
                    log.add("advisor:" + inv.getMethod().getName());
                    return inv.proceed();
                }
        ));
        container.registerBeanDefinition("userService", SimpleUserService.class);

        container.refresh();

        container.getBean(UserService.class).findUser("test");

        assertThat(log).contains("advisor:findUser");
    }

    @Test
    void shouldNotEnableAop_WhenNoEnableAspectJAutoProxy() {
        var container = new SimpleBeanContainer();
        List<String> log = new ArrayList<>();

        // No @EnableAspectJAutoProxy config registered!
        container.registerBean("loggingAspect", new TestLoggingAspect(log));
        container.registerBeanDefinition("userService", SimpleUserService.class);

        container.refresh();

        var service = container.getBean(UserService.class);
        service.findUser("alice");

        // Without @EnableAspectJAutoProxy, no proxying should occur
        assertThat(log).isEmpty();
    }

    // ─── Test fixtures ───────────────────────────────────────

    @EnableAspectJAutoProxy
    public static class AopEnabledConfig {
    }

    public interface UserService {
        String findUser(String name);
    }

    public interface OrderService {
        String createOrder(String item);
    }

    public static class SimpleUserService implements UserService {
        @Override
        public String findUser(String name) { return "User: " + name; }
    }

    public static class SimpleOrderService implements OrderService {
        @Override
        public String createOrder(String item) { return "Order: " + item; }
    }

    @Aspect
    public static class TestLoggingAspect {
        private final List<String> log;
        public TestLoggingAspect(List<String> log) { this.log = log; }

        @Before("find*")
        public void logBefore() { log.add("before:findUser"); }
    }

    @Aspect
    public static class AllMethodAspect {
        private final List<String> log;
        public AllMethodAspect(List<String> log) { this.log = log; }

        @Around("*")
        public Object logAround(MethodInvocation inv) throws Throwable {
            log.add("around:" + inv.getMethod().getName());
            return inv.proceed();
        }
    }

    @Aspect
    public static class SaveOnlyAspect {
        private final List<String> log;
        public SaveOnlyAspect(List<String> log) { this.log = log; }

        @Before("save*")
        public void logBeforeSave() { log.add("before:save"); }
    }
}
```

---

## Summary

| What | Detail |
|------|--------|
| **New classes** | `Advice`, `MethodInterceptor`, `MethodInvocation`, `ReflectiveMethodInvocation`, `Pointcut`, `NamePatternPointcut`, `AnnotationPointcut`, `Advisor`, `DefaultPointcutAdvisor`, `AdvisedSupport`, `AopProxy`, `JdkDynamicAopProxy`, `CglibAopProxy`, `ProxyFactory`, `@Aspect`, `@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`, `@EnableAspectJAutoProxy`, `AspectJAutoProxyCreator` |
| **Modified classes** | `SimpleBeanContainer` (getType, autoRegisterAopProcessorIfNeeded), `build.gradle` (ByteBuddy) |
| **New tests** | 43 tests (9 Pointcut + 5 ReflectiveMethodInvocation + 5 JdkDynamicAopProxy + 6 CglibAopProxy + 6 ProxyFactory + 7 AspectJAutoProxyCreator + 5 AopIntegration) |
| **Total tests** | 1003 (all passing) |
| **Key patterns** | Strategy (AopProxy), Chain of Responsibility (ReflectiveMethodInvocation), Adapter (@Before/@After to MethodInterceptor), Composite (Advisor = Advice + Pointcut) |
| **Key insight** | The BeanPostProcessor hook is the entire AOP integration point -- returning a proxy instead of the original bean transparently wraps any bean with cross-cutting behavior |

**Next chapter:** Chapter 29 -- Event System (ApplicationEvent / ApplicationListener).
