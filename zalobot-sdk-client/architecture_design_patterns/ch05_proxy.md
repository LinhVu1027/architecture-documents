# Chapter 5: Proxy — Transparent Cross-Cutting Concerns

## What You'll Learn
- How the **Proxy** pattern intercepts method calls without modifying target classes
- How JDK Dynamic Proxies work under the hood
- How Spring AOP builds interceptor chains for `@Transactional`, `@Cacheable`, etc.
- Why proxies are the foundation of Spring's "magic"

---

## 5.1 The Problem: Cross-Cutting Concerns Everywhere

From Chapter 1, adding logging/timing/transactions to every method means modifying every method:

```java
// WITHOUT proxy: timing is mixed into business logic
public User findUser(long id) {
    long start = System.currentTimeMillis();     // cross-cutting
    log.info("Entering findUser");                // cross-cutting
    beginTransaction();                           // cross-cutting

    User user = repository.findById(id);          // ◄── actual logic (1 line!)

    commitTransaction();                          // cross-cutting
    log.info("Exiting findUser");                 // cross-cutting
    log.info("Took " + (System.currentTimeMillis() - start) + "ms"); // cross-cutting
    return user;
}
```

**7 lines** of cross-cutting concerns wrapping **1 line** of business logic. And this must be repeated in EVERY method.

## 5.2 Proxy Pattern: Intercept Without Modifying

A proxy wraps the target object and intercepts calls:

```
  Client                    Proxy                    Target
    │                        │                        │
    │  findUser(42)          │                        │
    │ ──────────────────────►│                        │
    │                        │  [before advice]       │
    │                        │  log("Entering...")    │
    │                        │  beginTransaction()   │
    │                        │                        │
    │                        │  findUser(42)          │
    │                        │───────────────────────►│
    │                        │                        │  actual logic
    │                        │◄───────────────────────│
    │                        │  result = User(...)    │
    │                        │                        │
    │                        │  [after advice]        │
    │                        │  commitTransaction()  │
    │                        │  log("Took Xms")      │
    │                        │                        │
    │◄───────────────────────│  return result         │
    │                        │                        │
```

The target class has **no idea** it's being proxied. The business logic stays clean.

> ★ **Insight** ─────────────────────────────────────
> - The Proxy pattern is the **most impactful** pattern in Spring. It's how `@Transactional`, `@Cacheable`, `@Async`, `@Secured`, and `@Retryable` all work — they're just advice applied through proxies.
> - Java provides two proxy mechanisms: **JDK Dynamic Proxy** (for interfaces — uses `java.lang.reflect.Proxy`) and **CGLIB** (for concrete classes — generates a subclass at runtime). Spring auto-selects: interface → JDK, class → CGLIB.
> - **When NOT to use Proxy:** For internal method calls within the same class (the proxy is bypassed). This is the famous "self-invocation" limitation of Spring AOP. If `methodA()` calls `this.methodB()`, the proxy around `methodB()` is NOT applied.
> ─────────────────────────────────────────────────────

## 5.3 Building SimpleAopProxy

### Step 1: The Interceptor Interface (Advice)

```java
// === MethodInterceptor.java ===
/**
 * Intercepts method invocations on a proxy.
 * This is the AOP Alliance standard interface.
 */
@FunctionalInterface
public interface MethodInterceptor {
    /**
     * @param invocation contains the method, arguments, and a way to proceed
     * @return the method's return value (potentially modified)
     */
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

```java
// === MethodInvocation.java ===
import java.lang.reflect.Method;

/**
 * Represents a method call that can be proceeded through the interceptor chain.
 */
public interface MethodInvocation {
    Method getMethod();
    Object[] getArguments();
    Object getTarget();
    Object proceed() throws Throwable;  // ◄── call next interceptor or target
}
```

### Step 2: The Interceptor Chain (Recursive Proceed)

```java
// === SimpleMethodInvocation.java ===
import java.lang.reflect.Method;
import java.util.List;

/**
 * Manages the interceptor chain. Each call to proceed() invokes the next
 * interceptor, or the actual target method when the chain is exhausted.
 *
 * This is a simplified version of Spring's ReflectiveMethodInvocation.
 */
public class SimpleMethodInvocation implements MethodInvocation {

    private final Object proxy;
    private final Object target;
    private final Method method;
    private final Object[] arguments;
    private final List<MethodInterceptor> interceptors;
    private int currentIndex = -1;  // tracks position in chain

    public SimpleMethodInvocation(Object proxy, Object target, Method method,
                                   Object[] arguments, List<MethodInterceptor> interceptors) {
        this.proxy = proxy;
        this.target = target;
        this.method = method;
        this.arguments = arguments;
        this.interceptors = interceptors;
    }

    @Override
    public Object proceed() throws Throwable {
        // All interceptors invoked → call target method
        if (++this.currentIndex >= this.interceptors.size()) {
            return this.method.invoke(this.target, this.arguments);
        }

        // Get next interceptor and invoke it
        MethodInterceptor interceptor = this.interceptors.get(this.currentIndex);
        return interceptor.invoke(this);
        // ↑ The interceptor calls invocation.proceed() when ready,
        //   which comes back here with currentIndex incremented
    }

    @Override
    public Method getMethod() { return method; }

    @Override
    public Object[] getArguments() { return arguments; }

    @Override
    public Object getTarget() { return target; }
}
```

### Step 3: The JDK Dynamic Proxy

```java
// === SimpleAopProxy.java ===
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.ArrayList;
import java.util.List;

/**
 * Creates JDK dynamic proxies with an interceptor chain.
 * Simplified version of Spring's JdkDynamicAopProxy.
 */
public class SimpleAopProxy implements InvocationHandler {

    private final Object target;
    private final List<MethodInterceptor> interceptors = new ArrayList<>();

    public SimpleAopProxy(Object target) {
        this.target = target;
    }

    public SimpleAopProxy addInterceptor(MethodInterceptor interceptor) {
        this.interceptors.add(interceptor);
        return this;
    }

    /**
     * Create the proxy instance.
     * Uses java.lang.reflect.Proxy which requires interfaces.
     */
    @SuppressWarnings("unchecked")
    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this  // this object handles all method calls
        );
    }

    /**
     * Called for EVERY method invocation on the proxy.
     * This is where the magic happens.
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // Handle Object methods directly
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(target, args);
        }

        if (interceptors.isEmpty()) {
            // No interceptors → direct call (no overhead)
            return method.invoke(target, args);
        }

        // Build the invocation chain and proceed
        SimpleMethodInvocation invocation = new SimpleMethodInvocation(
            proxy, target, method, args, interceptors
        );
        return invocation.proceed();
    }
}
```

### Step 4: ProxyFactory — Convenient Creation

```java
// === SimpleProxyFactory.java ===
import java.util.ArrayList;
import java.util.List;

/**
 * Facade for creating AOP proxies.
 * Simplified version of Spring's ProxyFactory.
 */
public class SimpleProxyFactory {

    private Object target;
    private final List<MethodInterceptor> interceptors = new ArrayList<>();

    public SimpleProxyFactory() {}

    public SimpleProxyFactory(Object target) {
        this.target = target;
    }

    public SimpleProxyFactory setTarget(Object target) {
        this.target = target;
        return this;
    }

    public SimpleProxyFactory addAdvice(MethodInterceptor interceptor) {
        this.interceptors.add(interceptor);
        return this;
    }

    @SuppressWarnings("unchecked")
    public <T> T getProxy() {
        SimpleAopProxy aopProxy = new SimpleAopProxy(target);
        for (MethodInterceptor interceptor : interceptors) {
            aopProxy.addInterceptor(interceptor);
        }
        return aopProxy.getProxy();
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - The `proceed()` mechanism is a **recursive chain**: each interceptor calls `invocation.proceed()` which invokes the NEXT interceptor, until the chain is exhausted and the actual target method executes. This is functionally similar to **Chain of Responsibility** but implemented via recursion rather than a linked list.
> - In real Spring (`JdkDynamicAopProxy.invoke()` at line 166), the proxy also handles: `equals()` / `hashCode()` delegation, the `Advised` interface for runtime proxy configuration, `AopContext.setCurrentProxy()` for self-invocation workarounds, and Kotlin coroutine support.
> - The `ProxyFactory` is a **Facade** over the proxy creation process — it hides `AopProxy`, `AdvisedSupport`, and `AopProxyFactory` behind simple `setTarget()` + `addAdvice()` + `getProxy()`.
> ─────────────────────────────────────────────────────

## 5.4 Common Interceptor Implementations

```java
// === LoggingInterceptor.java ===
/**
 * Logs method entry/exit — demonstrates @Around advice.
 */
public class LoggingInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        String methodName = invocation.getMethod().getName();
        System.out.println("[LOG] Entering: " + methodName);

        Object result = invocation.proceed();  // ◄── call next interceptor / target

        System.out.println("[LOG] Exiting: " + methodName + " → " + result);
        return result;
    }
}
```

```java
// === TimingInterceptor.java ===
/**
 * Measures method execution time.
 */
public class TimingInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        long start = System.nanoTime();

        Object result = invocation.proceed();

        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.println("[TIMING] " + invocation.getMethod().getName() + " took " + elapsed + "ms");
        return result;
    }
}
```

```java
// === TransactionInterceptor.java ===
/**
 * Wraps method in transaction — this is how @Transactional works!
 */
public class TransactionInterceptor implements MethodInterceptor {
    private final SimpleTransactionManager txManager;

    public TransactionInterceptor(SimpleTransactionManager txManager) {
        this.txManager = txManager;
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        SimpleTransactionStatus status = txManager.begin();
        try {
            Object result = invocation.proceed();
            txManager.commit(status);
            return result;
        } catch (Throwable ex) {
            txManager.rollback(status);
            throw ex;
        }
    }
}
```

## 5.5 Usage

```java
// === UserService.java (interface required for JDK proxy) ===
public interface UserService {
    String findUser(long id);
    void saveUser(String name);
}

// === UserServiceImpl.java ===
public class UserServiceImpl implements UserService {
    @Override
    public String findUser(long id) {
        // Pure business logic — no boilerplate!
        return "User-" + id;
    }

    @Override
    public void saveUser(String name) {
        System.out.println("Saving user: " + name);
    }
}
```

```java
// === ProxyDemo.java ===
public class ProxyDemo {
    public static void main(String[] args) {
        // Create target
        UserService target = new UserServiceImpl();

        // Create proxy with interceptor chain
        InMemoryTransactionManager txManager = new InMemoryTransactionManager();

        UserService proxy = new SimpleProxyFactory(target)
            .addAdvice(new LoggingInterceptor())
            .addAdvice(new TimingInterceptor())
            .addAdvice(new TransactionInterceptor(txManager))
            .getProxy();

        // Call through proxy — interceptors fire automatically!
        String user = proxy.findUser(42);
        System.out.println("Got: " + user);
        System.out.println("TX Log: " + txManager.getLog());
    }
}
```

**Output:**
```
[LOG] Entering: findUser
[TIMING] findUser took 0ms
[LOG] Exiting: findUser → User-42
Got: User-42
TX Log: [BEGIN, COMMIT]
```

The interceptor chain executes in order: Logging → Timing → Transaction → Target. Each interceptor wraps the next via `proceed()`.

## 5.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleAopProxy` | `JdkDynamicAopProxy` | Implements `InvocationHandler` |
| `SimpleAopProxy.invoke()` | `JdkDynamicAopProxy.invoke()` | `:166-255` |
| `SimpleAopProxy.getProxy()` | `JdkDynamicAopProxy.getProxy()` | `:120-125` |
| `SimpleMethodInvocation` | `ReflectiveMethodInvocation` | `spring-aop` |
| `SimpleMethodInvocation.proceed()` | `ReflectiveMethodInvocation.proceed()` | Recursive chain |
| `SimpleProxyFactory` | `ProxyFactory` | `:36-151` |
| `MethodInterceptor` | `org.aopalliance.intercept.MethodInterceptor` | AOP Alliance |
| `TransactionInterceptor` | `TransactionInterceptor` | `spring-tx` |
| `LoggingInterceptor` | Custom `@Around` advice | Via `@Aspect` |

**What real Spring adds:**
- `ObjenesisCglibAopProxy` for proxying concrete classes (no interface needed)
- `Pointcut` expressions to selectively apply advice to specific methods
- `Advisor` = `Pointcut` + `Advice` — combines WHERE and WHAT
- `DefaultAdvisorChainFactory` for building the interceptor chain per method
- `TargetSource` abstraction for lazy init, pooling, and hot-swapping targets
- `AopContext.setCurrentProxy()` for self-invocation scenarios

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Proxy Pattern** | Wrap target object to intercept method calls transparently |
| **JDK Dynamic Proxy** | `java.lang.reflect.Proxy` + `InvocationHandler`; requires interfaces |
| **Interceptor Chain** | Ordered list of interceptors; each calls `proceed()` to invoke next |
| **ProxyFactory** | Facade for configuring and creating proxies |
| **Around Advice** | Interceptor that wraps the target call (before + after in one) |

**Next: Chapter 6** — We build `SimpleEventBus` using the **Observer** pattern for decoupled component communication.
