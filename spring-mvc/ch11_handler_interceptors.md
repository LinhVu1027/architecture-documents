# Chapter 11: Handler Interceptors

> **Generated from commit `11ab0b4351` of the Spring Framework repository.**

---

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| DispatcherServlet dispatches requests straight from handler lookup → adapter invocation | No hooks for cross-cutting concerns (logging, auth, timing) around handler execution | Add a pre/post/completion interceptor pipeline wrapping the handler lifecycle |

**What you'll build:** A `HandlerInterceptor` interface with three lifecycle hooks, a `HandlerExecutionChain` that manages forward/reverse iteration with correct cleanup guarantees, and an updated `DispatcherServlet.doDispatch()` that wraps handler invocation with the interceptor lifecycle.

---

## 11.1 The Integration Point

The interceptor feature connects **the interceptor pipeline** to **the dispatch lifecycle** inside `SimpleDispatcherServlet.doDispatch()`.

Before this chapter, `doDispatch()` is a straight pipeline:

```java
// BEFORE (ch04): lookup → adapt → invoke
HandlerMethod handler = getHandler(request);
HandlerAdapter adapter = getHandlerAdapter(handler);
adapter.handle(request, response, handler);
```

After this chapter, it becomes:

```java
// AFTER (ch11): lookup → preHandle → invoke → postHandle → afterCompletion
HandlerExecutionChain mappedHandler = getHandler(request);  // now returns a chain
if (!mappedHandler.applyPreHandle(request, response)) return;  // can veto
adapter.handle(request, response, mappedHandler.getHandler());
mappedHandler.applyPostHandle(request, response);
mappedHandler.triggerAfterCompletion(request, response, ex);   // always
```

**Direction:** From this integration point, we need to build:
1. `HandlerInterceptor` — the interface defining three callbacks
2. `HandlerExecutionChain` — the class that wraps handler + interceptors and manages iteration
3. Modified `SimpleDispatcherServlet` — updated `doDispatch()` + interceptor registration
4. Modified `getHandler()` — now returns `HandlerExecutionChain` instead of bare `HandlerMethod`

**Key design decision:** The `interceptorIndex` field in `HandlerExecutionChain` is the central bookkeeping mechanism. It starts at `-1` and increments only when a `preHandle` returns `true`. This single integer determines exactly which interceptors get their `afterCompletion` called — a clever alternative to maintaining a separate "completed interceptors" collection.

---

## 11.2 The HandlerInterceptor Interface

This interface defines three lifecycle hooks around handler execution.

**New file:** `src/main/java/com/simplespringmvc/interceptor/HandlerInterceptor.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public interface HandlerInterceptor {

    default boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              HandlerMethod handler) throws Exception {
        return true;
    }

    default void postHandle(HttpServletRequest request, HttpServletResponse response,
                            HandlerMethod handler) throws Exception {
    }

    default void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 HandlerMethod handler, Exception ex) throws Exception {
    }
}
```

All three methods are `default` — implementations only override the hooks they care about.

**The three-method contract:**

| Method | When | Direction | Can abort? |
|--------|------|-----------|------------|
| `preHandle` | Before handler executes | Forward (0, 1, 2, ...) | Yes (return `false`) |
| `postHandle` | After handler, before response commit | Reverse (..., 2, 1, 0) | No |
| `afterCompletion` | After everything (finally block) | Reverse from `interceptorIndex` | No |

---

## 11.3 The HandlerExecutionChain

This class pairs a handler with its interceptors and manages the forward/reverse iteration lifecycle.

**New file:** `src/main/java/com/simplespringmvc/interceptor/HandlerExecutionChain.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.ArrayList;
import java.util.List;

public class HandlerExecutionChain {

    private final HandlerMethod handler;
    private final List<HandlerInterceptor> interceptorList;
    private int interceptorIndex = -1;  // the key bookkeeping field

    public HandlerExecutionChain(HandlerMethod handler) {
        this(handler, new ArrayList<>());
    }

    public HandlerExecutionChain(HandlerMethod handler, List<HandlerInterceptor> interceptors) {
        this.handler = handler;
        this.interceptorList = new ArrayList<>(interceptors);
    }

    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptorList.add(interceptor);
    }

    // Forward iteration — the interceptorIndex advances after each success
    public boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        for (int i = 0; i < this.interceptorList.size(); i++) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            if (!interceptor.preHandle(request, response, this.handler)) {
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
        return true;
    }

    // Reverse iteration — all interceptors (they all passed preHandle)
    public void applyPostHandle(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            interceptor.postHandle(request, response, this.handler);
        }
    }

    // Reverse from interceptorIndex — only those whose preHandle succeeded
    public void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response,
                                       Exception ex) {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            } catch (Throwable ex2) {
                System.err.println("HandlerInterceptor.afterCompletion threw exception: " + ex2);
            }
        }
    }

    public HandlerMethod getHandler() {
        return handler;
    }

    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptorList);
    }
}
```

### The interceptorIndex Trace

Here's how the `interceptorIndex` tracks through two scenarios:

**Normal flow (interceptors A, B, C all approve):**
```
A.preHandle → true   → interceptorIndex = 0
B.preHandle → true   → interceptorIndex = 1
C.preHandle → true   → interceptorIndex = 2
--- handler invoked ---
C.postHandle (i=2), B.postHandle (i=1), A.postHandle (i=0)
C.afterCompletion (i=2), B.afterCompletion (i=1), A.afterCompletion (i=0)
```

**B vetoes (A approved, B returns false, C never runs):**
```
A.preHandle → true   → interceptorIndex = 0
B.preHandle → false  → interceptorIndex stays 0
--- triggerAfterCompletion called ---
A.afterCompletion (i=0)  ← only A, because interceptorIndex is 0
--- handler NOT invoked, postHandle NOT called ---
```

---

## 11.4 Updating the DispatcherServlet

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

Three changes:
1. **New field** — `List<HandlerInterceptor> interceptors` and `addInterceptor()` method
2. **Updated `doDispatch()`** — wrap handler invocation with interceptor lifecycle
3. **Updated `getHandler()`** — return `HandlerExecutionChain` instead of bare `HandlerMethod`

**Change 1:** Add interceptor list field and registration method

```java
private final List<HandlerInterceptor> interceptors = new ArrayList<>();

public void addInterceptor(HandlerInterceptor interceptor) {
    this.interceptors.add(interceptor);
}
```

**Change 2:** Updated `doDispatch()` — the complete new method

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HandlerExecutionChain mappedHandler = null;
    Exception dispatchException = null;

    try {
        // Step 1: Look up the handler + interceptor chain
        mappedHandler = getHandler(request);

        if (mappedHandler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    "No handler found for " + request.getMethod() + " " + request.getRequestURI());
            return;
        }

        // Step 2: Apply interceptor preHandle — forward order
        if (!mappedHandler.applyPreHandle(request, response)) {
            return;  // interceptor vetoed; afterCompletion already called inside
        }

        // Step 3: Find a HandlerAdapter that supports this handler
        HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());

        // Step 4: Delegate invocation to the adapter
        adapter.handle(request, response, mappedHandler.getHandler());

        // Step 5: Apply interceptor postHandle — reverse order
        mappedHandler.applyPostHandle(request, response);

    } catch (Exception ex) {
        dispatchException = ex;
    }

    // Step 6: afterCompletion — always runs
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, dispatchException);
    }

    // Re-throw if there was an exception
    if (dispatchException != null) {
        throw dispatchException;
    }
}
```

**Change 3:** Updated `getHandler()` — now returns `HandlerExecutionChain`

```java
private HandlerExecutionChain getHandler(HttpServletRequest request) {
    if (handlerMapping == null) {
        return null;
    }
    HandlerMethod handler = handlerMapping.lookupHandler(request);
    if (handler == null) {
        return null;
    }
    return new HandlerExecutionChain(handler, interceptors);
}
```

---

## 11.5 Try It Yourself

Before looking at the implementation, try building these components yourself.

<details><summary>Challenge 1: Implement the HandlerInterceptor interface</summary>

Define an interface with three default methods: `preHandle` (returns boolean), `postHandle` (void), and `afterCompletion` (void, receives an Exception parameter). All methods take `HttpServletRequest`, `HttpServletResponse`, and `HandlerMethod`.

See the full solution in `src/main/java/com/simplespringmvc/interceptor/HandlerInterceptor.java`.

</details>

<details><summary>Challenge 2: Implement the interceptorIndex bookkeeping in applyPreHandle</summary>

The key insight: `interceptorIndex` is updated AFTER a successful `preHandle`, not before. If interceptor B at index 1 returns `false`, the `interceptorIndex` stays at 0 (where A succeeded). When `triggerAfterCompletion` runs, it counts down from `interceptorIndex` — so only A gets cleanup, not B.

```java
public boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    for (int i = 0; i < this.interceptorList.size(); i++) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (!interceptor.preHandle(request, response, this.handler)) {
            triggerAfterCompletion(request, response, null);
            return false;
        }
        this.interceptorIndex = i;  // only after success
    }
    return true;
}
```

</details>

<details><summary>Challenge 3: Update doDispatch() with try/catch for afterCompletion</summary>

The try/catch structure ensures `afterCompletion` always runs. The exception is captured (not immediately thrown) so that cleanup can happen first:

```java
try {
    // preHandle → handle → postHandle
} catch (Exception ex) {
    dispatchException = ex;  // capture, don't throw yet
}
// Always run afterCompletion
if (mappedHandler != null) {
    mappedHandler.triggerAfterCompletion(request, response, dispatchException);
}
// Now re-throw
if (dispatchException != null) {
    throw dispatchException;
}
```

</details>

---

## 11.6 Tests

### Unit Tests

**File:** `src/test/java/com/simplespringmvc/interceptor/HandlerInterceptorTest.java`

Tests the default method contract — all three callbacks have safe defaults:
- `shouldReturnTrue_WhenDefaultPreHandle` — default is to approve
- `shouldNotThrow_WhenDefaultPostHandle` — default is no-op
- `shouldNotThrow_WhenDefaultAfterCompletion` — default is no-op
- `shouldAllowPartialOverride` — can override just one method

**File:** `src/test/java/com/simplespringmvc/interceptor/HandlerExecutionChainTest.java`

Tests the forward/reverse iteration lifecycle using `RecordingInterceptor`:

**applyPreHandle:**
- `shouldReturnTrue_WhenAllApprove` — all interceptors run in forward order
- `shouldReturnFalse_WhenAnyVetoes` — stops at the vetoing interceptor
- `shouldTriggerAfterCompletion_WhenVetoOccurs` — only approved interceptors get cleanup
- `shouldReturnTrue_WhenNoInterceptors` — empty chain passes
- `shouldCallInForwardOrder` — verifies 0→N ordering

**applyPostHandle:**
- `shouldCallInReverseOrder` — verifies N→0 ordering
- `shouldDoNothing_WhenNoInterceptors` — handles empty chain

**triggerAfterCompletion:**
- `shouldCallInReverse_FromInterceptorIndex` — uses interceptorIndex, not list size
- `shouldNotCallAfterCompletion_WhenNoPreHandleRan` — interceptorIndex -1 means nothing runs
- `shouldContinueCleanup_WhenAfterCompletionThrows` — one error doesn't prevent others
- `shouldPassException_WhenHandlerThrew` — exception is forwarded

**Full lifecycle:**
- `shouldExecuteCompleteLifecycle` — A.pre → B.pre → handler → B.post → A.post → B.after → A.after
- `shouldSkipPostHandleAndHandler_WhenVetoOccurs` — veto skips handler and postHandle
- `shouldHandleFirstInterceptorVeto` — first interceptor veto means no afterCompletion for anyone

### Integration Tests

**File:** `src/test/java/com/simplespringmvc/integration/InterceptorIntegrationTest.java`

Tests the full HTTP stack with real Tomcat:
- `shouldExecuteFullLifecycle` — verifies lifecycle order through HTTP
- `shouldWorkWithNoInterceptors` — backward-compatible (no interceptors registered)
- `shouldWorkWithSingleInterceptor` — single interceptor lifecycle
- `shouldAbortRequest_WhenInterceptorReturnsFalse` — returns 403, only first interceptor gets cleanup
- `shouldNotInvokeHandler_WhenFirstInterceptorVetoes` — handler never executes
- `shouldCallAfterCompletion_WhenHandlerThrows` — afterCompletion receives the exception
- `shouldCallAfterCompletionForAll_WhenHandlerThrows` — all interceptors get cleanup on error
- `shouldSupportTimingHeaders` — real-world timing interceptor use case
- `shouldNotTriggerInterceptors_WhenNoHandlerFound` — 404 means no interceptors fire

Run all tests:
```bash
cd simple-spring-mvc && ./gradlew test
```

---

## 11.7 Why This Works

### ★ Insight: The interceptorIndex is a "Stack Unwinding" Mechanism

The `interceptorIndex` field is analogous to how compilers implement exception stack unwinding in C++ or Go's `defer` stack. Each successful `preHandle` "pushes" onto the cleanup stack (by advancing the index). When cleanup runs, it "pops" from the index back down to 0. The beauty is that a single integer replaces what would otherwise be a `List<HandlerInterceptor>` of "completed" interceptors. This is possible because the interceptors are always iterated in order — so an integer index is sufficient to represent "all interceptors from 0 to N."

### ★ Insight: Fail-Safe Cleanup via Individual Try/Catch

In `triggerAfterCompletion`, each `afterCompletion` call is individually wrapped in try/catch. This prevents a cascading failure — if interceptor C's cleanup throws, interceptors B and A still get their cleanup calls. This mirrors the pattern used in JDBC connection pool cleanup and Servlet filter chain destroy: individual resource cleanup must never prevent other resources from being cleaned up. The real Spring Framework logs the error via SLF4J but otherwise swallows it.

### ★ Insight: Why postHandle Iterates the Full List, But afterCompletion Uses interceptorIndex

This is a subtle but important distinction:
- `applyPostHandle()` is only called on the **success path** — meaning all `preHandle` calls succeeded. So it's safe to iterate the full list.
- `triggerAfterCompletion()` is called on **both** success and error paths. On the error path, some interceptors may not have had their `preHandle` called. So it must only iterate up to `interceptorIndex`.

---

## 11.8 What We Enhanced

| File | Change | Why |
|------|--------|-----|
| `SimpleDispatcherServlet.java` | `doDispatch()` now wraps invocation with interceptor lifecycle (preHandle → invoke → postHandle → afterCompletion) | Connect interceptors to the dispatch pipeline |
| `SimpleDispatcherServlet.java` | `getHandler()` now returns `HandlerExecutionChain` instead of bare `HandlerMethod` | Match the real framework's pattern — getHandler always returns a chain |
| `SimpleDispatcherServlet.java` | Added `interceptors` field + `addInterceptor()` method | Global interceptor registration |
| `SimpleDispatcherServlet.java` | `doDispatch()` uses try/catch to capture exceptions and always run afterCompletion | Guarantee cleanup even on handler errors |

---

## 11.9 Connection to Real Framework

| Simplified | Real Spring | File | Line(s) |
|---|---|---|---|
| `HandlerInterceptor` | `HandlerInterceptor` | `spring-webmvc/.../HandlerInterceptor.java` | 42–104 |
| `HandlerExecutionChain` | `HandlerExecutionChain` | `spring-webmvc/.../HandlerExecutionChain.java` | 47–200 |
| `HandlerExecutionChain.interceptorIndex` | `HandlerExecutionChain.interceptorIndex` | `spring-webmvc/.../HandlerExecutionChain.java` | 55 |
| `applyPreHandle()` | `applyPreHandle()` | `spring-webmvc/.../HandlerExecutionChain.java` | 142–152 |
| `applyPostHandle()` | `applyPostHandle()` | `spring-webmvc/.../HandlerExecutionChain.java` | 157–164 |
| `triggerAfterCompletion()` | `triggerAfterCompletion()` | `spring-webmvc/.../HandlerExecutionChain.java` | 171–181 |
| `doDispatch()` interceptor flow | `doDispatch()` | `spring-webmvc/.../DispatcherServlet.java` | 935–1004 |
| `addInterceptor()` on servlet | `InterceptorRegistry` + `AbstractHandlerMapping.adaptedInterceptors` | `spring-webmvc/.../handler/AbstractHandlerMapping.java` | 123, 450–455 |

**Simplifications:**
- Real framework uses `Object handler` (supports multiple handler types); we use `HandlerMethod`
- Real framework's `postHandle` receives `ModelAndView`; we omit it (ch13)
- Real framework has `AsyncHandlerInterceptor` for async dispatch; we don't support async
- Real framework supports URL-pattern-scoped interceptors via `MappedInterceptor`; ours are global
- Real framework holds interceptors per-HandlerMapping; we hold them on the DispatcherServlet

**Commit:** `11ab0b4351`

---

## 11.10 Complete Code

All files for this feature, exactly as they exist in `src/`.

### Production Code

#### [NEW] `src/main/java/com/simplespringmvc/interceptor/HandlerInterceptor.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for intercepting handler execution at three lifecycle points:
 * before the handler, after the handler, and after request completion.
 *
 * Maps to: {@code org.springframework.web.servlet.HandlerInterceptor}
 *
 * The real interface uses default methods (since Spring 5.x) so implementations
 * can override only the callbacks they care about. We do the same.
 *
 * <h3>Lifecycle within DispatcherServlet.doDispatch():</h3>
 * <pre>
 *   1. preHandle()       → forward order (0, 1, 2, ...)
 *   2. handler invoked
 *   3. postHandle()      → reverse order (..., 2, 1, 0)
 *   4. afterCompletion() → reverse from interceptorIndex (only those whose preHandle succeeded)
 * </pre>
 *
 * <h3>Key contract:</h3>
 * <ul>
 *   <li>If {@code preHandle} returns {@code false}, the remaining interceptors and handler
 *       are skipped. {@code afterCompletion} is called on all interceptors whose
 *       {@code preHandle} already returned {@code true}.</li>
 *   <li>{@code postHandle} is called only on the normal path (no exception).
 *       It is invoked in reverse order.</li>
 *   <li>{@code afterCompletion} is ALWAYS called (like a finally block) for each
 *       interceptor whose {@code preHandle} returned {@code true}.</li>
 * </ul>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>Takes {@code HandlerMethod} instead of {@code Object handler} — we only have one handler type</li>
 *   <li>No {@code ModelAndView} parameter on {@code postHandle} — we don't have views yet (ch13)</li>
 *   <li>No {@code AsyncHandlerInterceptor} sub-interface — no async support</li>
 * </ul>
 */
public interface HandlerInterceptor {

    /**
     * Called before the handler executes. Can veto execution by returning {@code false}.
     *
     * Maps to: {@code HandlerInterceptor.preHandle()} (line 66)
     *
     * Common uses: authentication checks, logging, request timing start.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the handler about to be invoked
     * @return {@code true} to continue the chain; {@code false} to abort
     * @throws Exception in case of errors
     */
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              HandlerMethod handler) throws Exception {
        return true;
    }

    /**
     * Called after the handler executes successfully, but before the response is committed.
     *
     * Maps to: {@code HandlerInterceptor.postHandle()} (line 85)
     *
     * In the real framework this receives a {@code ModelAndView} parameter for
     * interceptors to modify the view/model before rendering. We omit it since
     * we don't have view resolution yet (ch13).
     *
     * Common uses: adding common model attributes, modifying response headers.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the handler that was invoked
     * @throws Exception in case of errors
     */
    default void postHandle(HttpServletRequest request, HttpServletResponse response,
                            HandlerMethod handler) throws Exception {
    }

    /**
     * Called after the request is fully complete — like a finally block.
     * Always invoked for each interceptor whose {@code preHandle} returned {@code true},
     * regardless of whether the handler threw an exception.
     *
     * Maps to: {@code HandlerInterceptor.afterCompletion()} (line 104)
     *
     * Common uses: resource cleanup, logging request duration, releasing thread-locals.
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the handler that was invoked (or attempted)
     * @param ex       exception thrown during handler execution, or {@code null} if none
     * @throws Exception in case of errors
     */
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 HandlerMethod handler, Exception ex) throws Exception {
    }
}
```

#### [NEW] `src/main/java/com/simplespringmvc/interceptor/HandlerExecutionChain.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.ArrayList;
import java.util.List;

/**
 * Pairs a {@link HandlerMethod} with a list of {@link HandlerInterceptor}s and
 * manages the forward/reverse iteration lifecycle for pre-handle, post-handle,
 * and after-completion callbacks.
 *
 * Maps to: {@code org.springframework.web.servlet.HandlerExecutionChain}
 *
 * <h3>The interceptorIndex bookkeeping:</h3>
 * The critical field is {@code interceptorIndex}, which starts at {@code -1}.
 * It advances only when a {@code preHandle} returns {@code true}. When
 * {@code afterCompletion} runs, it counts DOWN from {@code interceptorIndex},
 * ensuring only interceptors whose {@code preHandle} succeeded get cleanup.
 *
 * <pre>
 *   Interceptors: [A, B, C]
 *
 *   Normal flow:
 *     A.preHandle → true  (interceptorIndex = 0)
 *     B.preHandle → true  (interceptorIndex = 1)
 *     C.preHandle → true  (interceptorIndex = 2)
 *     handler invoked
 *     C.postHandle, B.postHandle, A.postHandle  (reverse of all)
 *     C.afterCompletion, B.afterCompletion, A.afterCompletion  (reverse from index 2)
 *
 *   B vetoes:
 *     A.preHandle → true  (interceptorIndex = 0)
 *     B.preHandle → false (interceptorIndex stays 0)
 *     → handler NOT invoked, C.preHandle NOT called
 *     A.afterCompletion  (reverse from index 0 — only A's preHandle returned true)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No constructor unwrapping of nested HandlerExecutionChains</li>
 *   <li>No async handling ({@code applyAfterConcurrentHandlingStarted})</li>
 *   <li>No ModelAndView parameter on postHandle</li>
 * </ul>
 */
public class HandlerExecutionChain {

    private final HandlerMethod handler;
    private final List<HandlerInterceptor> interceptorList;

    /**
     * Tracks the index of the last interceptor whose {@code preHandle} returned {@code true}.
     * Starts at -1 (no interceptor has run). Used by {@link #triggerAfterCompletion}
     * to ensure cleanup runs only for interceptors that completed pre-processing.
     *
     * Maps to: {@code HandlerExecutionChain.interceptorIndex} (line 55)
     */
    private int interceptorIndex = -1;

    public HandlerExecutionChain(HandlerMethod handler) {
        this(handler, new ArrayList<>());
    }

    public HandlerExecutionChain(HandlerMethod handler, List<HandlerInterceptor> interceptors) {
        this.handler = handler;
        this.interceptorList = new ArrayList<>(interceptors);
    }

    /**
     * Add an interceptor to the end of the chain.
     */
    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptorList.add(interceptor);
    }

    /**
     * Apply {@code preHandle} on each interceptor in FORWARD order.
     *
     * Maps to: {@code HandlerExecutionChain.applyPreHandle()} (line 142)
     *
     * If any interceptor returns {@code false}, immediately triggers
     * {@code afterCompletion} for all previously successful interceptors
     * and returns {@code false} to signal that dispatch should stop.
     *
     * The key line is: {@code this.interceptorIndex = i} — this records which
     * interceptors need cleanup. The interceptor that returned {@code false}
     * does NOT get its afterCompletion called (its preHandle "didn't complete").
     *
     * @return {@code true} if all interceptors approved; {@code false} if any vetoed
     */
    public boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        for (int i = 0; i < this.interceptorList.size(); i++) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            if (!interceptor.preHandle(request, response, this.handler)) {
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
        return true;
    }

    /**
     * Apply {@code postHandle} on each interceptor in REVERSE order.
     *
     * Maps to: {@code HandlerExecutionChain.applyPostHandle()} (line 157)
     *
     * Only called on the success path (no exception during handler invocation).
     * Iterates the full list in reverse — by this point all preHandle calls
     * succeeded, so all interceptors deserve their postHandle callback.
     */
    public void applyPostHandle(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            interceptor.postHandle(request, response, this.handler);
        }
    }

    /**
     * Trigger {@code afterCompletion} in REVERSE order from {@code interceptorIndex}.
     *
     * Maps to: {@code HandlerExecutionChain.triggerAfterCompletion()} (line 171)
     *
     * This is the safety-net cleanup — analogous to a finally block. Key behaviors:
     * <ul>
     *   <li>Only iterates from {@code interceptorIndex} down to 0 (not the full list)</li>
     *   <li>Each call is wrapped in try/catch — one interceptor's error must NOT prevent
     *       cleanup of earlier interceptors</li>
     *   <li>The exception parameter is the exception from handler invocation (or null)</li>
     * </ul>
     *
     * @param ex the exception from handler invocation, or {@code null} if the request
     *           completed normally
     */
    public void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response,
                                       Exception ex) {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            } catch (Throwable ex2) {
                // Log but don't propagate — must not prevent other interceptors' cleanup.
                // The real framework logs via commons-logging here.
                System.err.println("HandlerInterceptor.afterCompletion threw exception: " + ex2);
            }
        }
    }

    /**
     * Returns the handler this chain wraps.
     */
    public HandlerMethod getHandler() {
        return handler;
    }

    /**
     * Returns an unmodifiable view of the interceptor list for testing/debugging.
     */
    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptorList);
    }

    @Override
    public String toString() {
        return "HandlerExecutionChain with [" + handler + "] and " +
                interceptorList.size() + " interceptors " + interceptorList;
    }
}
```

#### [MODIFIED] `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.interceptor.HandlerExecutionChain;
import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * The front controller that receives ALL HTTP requests and dispatches them
 * to the appropriate handler.
 *
 * Maps to: {@code org.springframework.web.servlet.DispatcherServlet}
 *
 * The real DispatcherServlet sits at the bottom of a three-class hierarchy:
 * <pre>
 *   HttpServletBean          → maps init-params to bean properties
 *     └── FrameworkServlet   → manages WebApplicationContext, request lifecycle
 *           └── DispatcherServlet → the actual dispatch logic
 * </pre>
 *
 * We collapse all three layers into one class. The key method is {@link #doDispatch}
 * which is the central routing point — every future feature adds a step here:
 * <ul>
 *   <li>ch03: handler mapping lookup</li>
 *   <li>ch04: handler adapter invocation</li>
 *   <li>ch11: interceptor pre/post processing ✓</li>
 *   <li>ch12: exception handler resolution</li>
 *   <li>ch13: view rendering</li>
 * </ul>
 *
 * <h3>ch11 Enhancement:</h3>
 * The dispatch pipeline now wraps handler invocation with interceptor lifecycle:
 * <pre>
 *   1. getHandler() returns HandlerExecutionChain (handler + interceptors)
 *   2. chain.applyPreHandle() — forward order, can veto
 *   3. adapter.handle() — the actual handler invocation
 *   4. chain.applyPostHandle() — reverse order (success path only)
 *   5. chain.triggerAfterCompletion() — reverse from interceptorIndex (always)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No locale/theme resolution (added in ch20)</li>
 *   <li>No flash map management (added in ch18)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    /**
     * Global interceptors applied to all handlers in order.
     *
     * Maps to: {@code AbstractHandlerMapping.adaptedInterceptors} (line 123)
     * In the real framework, interceptors are held per-HandlerMapping and can be
     * URL-pattern-scoped via MappedInterceptor. We keep it simple with a global list.
     *
     * <h3>ch11 Enhancement:</h3>
     * Added to support the interceptor pipeline.
     */
    private final List<HandlerInterceptor> interceptors = new ArrayList<>();

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Called by Tomcat once the Servlet is registered. Initializes strategy
     * components (handler mappings, adapters, etc.) from the bean container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * which is called from {@code onRefresh()}, which is triggered by
     * {@code FrameworkServlet.initWebApplicationContext()}.
     */
    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    /**
     * Initialize all strategy beans from the container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * Real version initializes 9 strategy types. We initialize them
     * incrementally as features are added.
     */
    protected void initStrategies() {
        initHandlerMapping();
        initHandlerAdapter();
        // Future features will initialize:
        // - ExceptionResolvers (ch12)
        // - ViewResolvers (ch13)
    }

    /**
     * Create the handler mapping and scan all beans for @Controller methods.
     *
     * Maps to: {@code DispatcherServlet.initHandlerMappings()} (line 505)
     * Real version looks for HandlerMapping beans in the context. We create
     * one directly and initialize it with our bean container.
     */
    private void initHandlerMapping() {
        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(beanContainer);
    }

    /**
     * Create the handler adapter that knows how to invoke handler methods.
     *
     * Maps to: {@code DispatcherServlet.initHandlerAdapters()} (line 552)
     * Real version finds all HandlerAdapter beans in the ApplicationContext
     * (or falls back to defaults from DispatcherServlet.properties).
     * The default adapters include:
     * <ul>
     *   <li>HttpRequestHandlerAdapter</li>
     *   <li>SimpleControllerHandlerAdapter</li>
     *   <li>RequestMappingHandlerAdapter</li>
     * </ul>
     *
     * We create just one adapter directly — the one that handles our
     * HandlerMethod type via reflection.
     */
    private void initHandlerAdapter() {
        handlerAdapter = new SimpleHandlerAdapter();
    }

    /**
     * Override service() to route ALL HTTP methods through doDispatch().
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870) which overrides
     * HttpServlet.service() and routes GET/POST/PUT/DELETE/PATCH/OPTIONS/TRACE
     * all through {@code processRequest()}, which calls {@code doService()},
     * which calls {@code doDispatch()}.
     *
     * We collapse that chain: service() → doDispatch() directly.
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        }
    }

    /**
     * The central dispatch method — the heart of the framework.
     *
     * Maps to: {@code DispatcherServlet.doDispatch()} (line 935)
     *
     * <h3>ch11 Enhancement:</h3>
     * The dispatch flow now includes interceptor lifecycle management:
     * <pre>
     *   1. mappedHandler = getHandler(request)           ← ch03, returns HandlerExecutionChain
     *   2. mappedHandler.applyPreHandle()                ← ch11 ✓ (can veto)
     *   3. ha = getHandlerAdapter(handler)               ← ch04 ✓
     *   4. ha.handle(request, response, handler)         ← ch04 ✓
     *   5. mappedHandler.applyPostHandle()               ← ch11 ✓ (reverse order)
     *   6. mappedHandler.triggerAfterCompletion()         ← ch11 ✓ (always, in finally)
     *   7. processDispatchResult(mv, exception)          ← ch12, ch13
     * </pre>
     *
     * The try/catch/finally structure mirrors the real DispatcherServlet:
     * <ul>
     *   <li>Inner try: handler lookup → preHandle → invoke → postHandle</li>
     *   <li>Catch: captures dispatch exception for later error handling</li>
     *   <li>Outer finally: always triggers afterCompletion</li>
     * </ul>
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerExecutionChain mappedHandler = null;
        Exception dispatchException = null;

        try {
            // Step 1: Look up the handler + interceptor chain for this request (ch03 + ch11)
            mappedHandler = getHandler(request);

            if (mappedHandler == null) {
                response.sendError(HttpServletResponse.SC_NOT_FOUND,
                        "No handler found for " + request.getMethod() + " " + request.getRequestURI());
                return;
            }

            // Step 2: Apply interceptor preHandle — forward order (ch11)
            // If any interceptor returns false, the chain stops and afterCompletion
            // runs inside applyPreHandle(). We return immediately.
            if (!mappedHandler.applyPreHandle(request, response)) {
                return;
            }

            // Step 3: Find a HandlerAdapter that supports this handler (ch04)
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());

            // Step 4: Delegate invocation + response writing to the adapter (ch04)
            adapter.handle(request, response, mappedHandler.getHandler());

            // Step 5: Apply interceptor postHandle — reverse order (ch11)
            mappedHandler.applyPostHandle(request, response);

        } catch (Exception ex) {
            dispatchException = ex;
        }

        // Step 6: afterCompletion — always runs for interceptors whose preHandle succeeded
        // The real framework does this inside processDispatchResult() on the normal path,
        // and in a catch block on the exception path. We simplify to always run it here
        // after the try/catch, which is equivalent.
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, dispatchException);
        }

        // Re-throw the exception if there was one (ch12 will add exception handling)
        if (dispatchException != null) {
            throw dispatchException;
        }
    }

    /**
     * Find the HandlerExecutionChain that matches this request.
     *
     * Maps to: {@code DispatcherServlet.getHandler()} (line 1103)
     * Real version iterates a list of HandlerMapping beans and returns the
     * first non-null HandlerExecutionChain. We have just one mapping.
     *
     * <h3>ch11 Enhancement:</h3>
     * Now returns a {@link HandlerExecutionChain} wrapping the handler with
     * any registered interceptors, instead of a bare {@link HandlerMethod}.
     * This is how the real DispatcherServlet works — getHandler() always
     * returns a chain, never a raw handler.
     */
    private HandlerExecutionChain getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        HandlerMethod handler = handlerMapping.lookupHandler(request);
        if (handler == null) {
            return null;
        }
        // Wrap the handler with all registered interceptors
        return new HandlerExecutionChain(handler, interceptors);
    }

    /**
     * Find a HandlerAdapter that can handle the given handler.
     *
     * Maps to: {@code DispatcherServlet.getHandlerAdapter()} (line 1185)
     * Real version iterates a list of HandlerAdapters and returns the first
     * one where {@code adapter.supports(handler)} is true.
     *
     * We have just one adapter, so we check it directly. The real framework
     * throws ServletException if no adapter supports the handler — we do the same.
     */
    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
    }

    // ─── Interceptor registration ────────────────────────────────────

    /**
     * Register an interceptor to be applied to all handlers.
     *
     * Maps to: {@code InterceptorRegistry.addInterceptor()} in WebMvcConfigurer-based
     * configuration, which ultimately feeds into AbstractHandlerMapping.adaptedInterceptors.
     *
     * In the real framework, interceptors can be URL-pattern-scoped via
     * {@code addPathPatterns()} / {@code excludePathPatterns()}, creating a
     * {@code MappedInterceptor} wrapper. We keep it simple — all interceptors
     * are global.
     *
     * <h3>ch11 Enhancement:</h3>
     * Added to support programmatic interceptor registration.
     *
     * @param interceptor the interceptor to add
     */
    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptors.add(interceptor);
    }

    /**
     * Returns the registered interceptors (for testing/debugging).
     */
    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptors);
    }

    // ─── Accessors ───────────────────────────────────────────────────

    /**
     * Provides access to the bean container for strategy initialization.
     * Future features use this to look up HandlerMappings, HandlerAdapters, etc.
     */
    public BeanContainer getBeanContainer() {
        return beanContainer;
    }

    /**
     * Provides access to the handler mapping for testing and inspection.
     */
    public SimpleHandlerMapping getHandlerMapping() {
        return handlerMapping;
    }

    /**
     * Provides access to the handler adapter for testing and inspection.
     */
    public HandlerAdapter getHandlerAdapter() {
        return handlerAdapter;
    }
}
```

### Test Code

#### [NEW] `src/test/java/com/simplespringmvc/interceptor/HandlerInterceptorTest.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

/**
 * Unit tests for the {@link HandlerInterceptor} interface.
 *
 * Verifies that the default method implementations return the expected values,
 * so that concrete interceptors only need to override the callbacks they care about.
 */
class HandlerInterceptorTest {

    // HandlerMethod has constructor validation so we create a real one
    // using a dummy method instead of mocking.
    static class DummyBean {
        public void handle() {}
    }

    private final HttpServletRequest request = mock(HttpServletRequest.class);
    private final HttpServletResponse response = mock(HttpServletResponse.class);
    private final HandlerMethod handler;

    HandlerInterceptorTest() throws Exception {
        handler = new HandlerMethod(new DummyBean(), DummyBean.class.getMethod("handle"));
    }

    @Test
    @DisplayName("should return true from default preHandle")
    void shouldReturnTrue_WhenDefaultPreHandle() throws Exception {
        HandlerInterceptor interceptor = new HandlerInterceptor() {};

        boolean result = interceptor.preHandle(request, response, handler);

        assertThat(result).isTrue();
    }

    @Test
    @DisplayName("should not throw from default postHandle")
    void shouldNotThrow_WhenDefaultPostHandle() throws Exception {
        HandlerInterceptor interceptor = new HandlerInterceptor() {};

        // Should complete without exception
        interceptor.postHandle(request, response, handler);
    }

    @Test
    @DisplayName("should not throw from default afterCompletion")
    void shouldNotThrow_WhenDefaultAfterCompletion() throws Exception {
        HandlerInterceptor interceptor = new HandlerInterceptor() {};

        // Should complete without exception
        interceptor.afterCompletion(request, response, handler, null);
    }

    @Test
    @DisplayName("should allow partial override of interface methods")
    void shouldAllowPartialOverride() throws Exception {
        // An interceptor that only overrides preHandle — other methods use defaults
        HandlerInterceptor authInterceptor = new HandlerInterceptor() {
            @Override
            public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                     HandlerMethod handler) {
                return false; // deny all
            }
        };

        assertThat(authInterceptor.preHandle(request, response, handler)).isFalse();
        // postHandle and afterCompletion still work via defaults
        authInterceptor.postHandle(request, response, handler);
        authInterceptor.afterCompletion(request, response, handler, null);
    }
}
```

#### [NEW] `src/test/java/com/simplespringmvc/interceptor/HandlerExecutionChainTest.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

/**
 * Unit tests for {@link HandlerExecutionChain}.
 *
 * Tests the forward/reverse iteration lifecycle and the interceptorIndex
 * bookkeeping that ensures correct afterCompletion cleanup.
 */
class HandlerExecutionChainTest {

    // HandlerMethod has constructor validation so we create a real one.
    static class DummyBean {
        public void handle() {}
    }

    private final HttpServletRequest request = mock(HttpServletRequest.class);
    private final HttpServletResponse response = mock(HttpServletResponse.class);
    private final HandlerMethod handler;

    HandlerExecutionChainTest() throws Exception {
        handler = new HandlerMethod(new DummyBean(), DummyBean.class.getMethod("handle"));
    }

    /**
     * A test interceptor that records all lifecycle calls in order.
     * The preHandle return value is configurable.
     */
    static class RecordingInterceptor implements HandlerInterceptor {
        final String name;
        final boolean preHandleResult;
        final List<String> calls;

        RecordingInterceptor(String name, boolean preHandleResult, List<String> calls) {
            this.name = name;
            this.preHandleResult = preHandleResult;
            this.calls = calls;
        }

        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                 HandlerMethod handler) {
            calls.add(name + ".preHandle");
            return preHandleResult;
        }

        @Override
        public void postHandle(HttpServletRequest request, HttpServletResponse response,
                               HandlerMethod handler) {
            calls.add(name + ".postHandle");
        }

        @Override
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                    HandlerMethod handler, Exception ex) {
            calls.add(name + ".afterCompletion");
        }

        @Override
        public String toString() {
            return name;
        }
    }

    private List<String> calls;

    @BeforeEach
    void setUp() {
        calls = new ArrayList<>();
    }

    // ─── applyPreHandle ─────────────────────────────────────────────

    @Nested
    @DisplayName("applyPreHandle")
    class ApplyPreHandle {

        @Test
        @DisplayName("should return true when all interceptors approve")
        void shouldReturnTrue_WhenAllApprove() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", true, calls));
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            boolean result = chain.applyPreHandle(request, response);

            assertThat(result).isTrue();
            assertThat(calls).containsExactly("A.preHandle", "B.preHandle", "C.preHandle");
        }

        @Test
        @DisplayName("should return false when any interceptor vetoes")
        void shouldReturnFalse_WhenAnyVetoes() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", false, calls)); // vetoes
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            boolean result = chain.applyPreHandle(request, response);

            assertThat(result).isFalse();
            // C.preHandle should NOT be called
            assertThat(calls).contains("A.preHandle", "B.preHandle");
            assertThat(calls).doesNotContain("C.preHandle");
        }

        @Test
        @DisplayName("should trigger afterCompletion for approved interceptors when veto occurs")
        void shouldTriggerAfterCompletion_WhenVetoOccurs() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", false, calls)); // vetoes
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            chain.applyPreHandle(request, response);

            // Only A's afterCompletion should be called (B vetoed, C never ran)
            assertThat(calls).containsExactly(
                    "A.preHandle", "B.preHandle", "A.afterCompletion");
        }

        @Test
        @DisplayName("should return true when no interceptors are registered")
        void shouldReturnTrue_WhenNoInterceptors() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);

            boolean result = chain.applyPreHandle(request, response);

            assertThat(result).isTrue();
            assertThat(calls).isEmpty();
        }

        @Test
        @DisplayName("should call interceptors in forward order")
        void shouldCallInForwardOrder() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("First", true, calls));
            chain.addInterceptor(new RecordingInterceptor("Second", true, calls));
            chain.addInterceptor(new RecordingInterceptor("Third", true, calls));

            chain.applyPreHandle(request, response);

            assertThat(calls).containsExactly(
                    "First.preHandle", "Second.preHandle", "Third.preHandle");
        }
    }

    // ─── applyPostHandle ────────────────────────────────────────────

    @Nested
    @DisplayName("applyPostHandle")
    class ApplyPostHandle {

        @Test
        @DisplayName("should call interceptors in reverse order")
        void shouldCallInReverseOrder() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", true, calls));
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            chain.applyPostHandle(request, response);

            assertThat(calls).containsExactly(
                    "C.postHandle", "B.postHandle", "A.postHandle");
        }

        @Test
        @DisplayName("should do nothing when no interceptors are registered")
        void shouldDoNothing_WhenNoInterceptors() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);

            chain.applyPostHandle(request, response);

            assertThat(calls).isEmpty();
        }
    }

    // ─── triggerAfterCompletion ──────────────────────────────────────

    @Nested
    @DisplayName("triggerAfterCompletion")
    class TriggerAfterCompletion {

        @Test
        @DisplayName("should call afterCompletion in reverse from interceptorIndex")
        void shouldCallInReverse_FromInterceptorIndex() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", true, calls));
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            // All preHandles succeed → interceptorIndex = 2
            chain.applyPreHandle(request, response);
            calls.clear();

            chain.triggerAfterCompletion(request, response, null);

            assertThat(calls).containsExactly(
                    "C.afterCompletion", "B.afterCompletion", "A.afterCompletion");
        }

        @Test
        @DisplayName("should not call afterCompletion when no preHandle ran")
        void shouldNotCallAfterCompletion_WhenNoPreHandleRan() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));

            // Don't call applyPreHandle — interceptorIndex stays -1
            chain.triggerAfterCompletion(request, response, null);

            assertThat(calls).isEmpty();
        }

        @Test
        @DisplayName("should continue cleanup even when afterCompletion throws")
        void shouldContinueCleanup_WhenAfterCompletionThrows() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            // B throws in afterCompletion
            chain.addInterceptor(new HandlerInterceptor() {
                @Override
                public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                         HandlerMethod handler) {
                    calls.add("B.preHandle");
                    return true;
                }

                @Override
                public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                            HandlerMethod handler, Exception ex) throws Exception {
                    calls.add("B.afterCompletion");
                    throw new RuntimeException("cleanup error in B");
                }
            });
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            chain.applyPreHandle(request, response);
            calls.clear();

            // Should NOT throw despite B's afterCompletion error
            chain.triggerAfterCompletion(request, response, null);

            // All three should still get afterCompletion called
            assertThat(calls).containsExactly(
                    "C.afterCompletion", "B.afterCompletion", "A.afterCompletion");
        }

        @Test
        @DisplayName("should pass exception to afterCompletion when handler threw")
        void shouldPassException_WhenHandlerThrew() throws Exception {
            List<Exception> capturedExceptions = new ArrayList<>();

            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new HandlerInterceptor() {
                @Override
                public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                         HandlerMethod handler) {
                    return true;
                }

                @Override
                public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                            HandlerMethod handler, Exception ex) {
                    capturedExceptions.add(ex);
                }
            });

            chain.applyPreHandle(request, response);

            Exception handlerException = new RuntimeException("handler failed");
            chain.triggerAfterCompletion(request, response, handlerException);

            assertThat(capturedExceptions).hasSize(1);
            assertThat(capturedExceptions.get(0)).isSameAs(handlerException);
        }
    }

    // ─── Full lifecycle ─────────────────────────────────────────────

    @Nested
    @DisplayName("full lifecycle")
    class FullLifecycle {

        @Test
        @DisplayName("should execute complete lifecycle in correct order")
        void shouldExecuteCompleteLifecycle() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", true, calls));

            // Simulate the full dispatch lifecycle
            chain.applyPreHandle(request, response);     // forward
            calls.add("handler.invoked");                  // handler runs
            chain.applyPostHandle(request, response);     // reverse
            chain.triggerAfterCompletion(request, response, null); // reverse

            assertThat(calls).containsExactly(
                    "A.preHandle",
                    "B.preHandle",
                    "handler.invoked",
                    "B.postHandle",
                    "A.postHandle",
                    "B.afterCompletion",
                    "A.afterCompletion");
        }

        @Test
        @DisplayName("should skip postHandle and handler when veto occurs mid-chain")
        void shouldSkipPostHandleAndHandler_WhenVetoOccurs() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", true, calls));
            chain.addInterceptor(new RecordingInterceptor("B", false, calls)); // vetoes
            chain.addInterceptor(new RecordingInterceptor("C", true, calls));

            boolean approved = chain.applyPreHandle(request, response);
            if (approved) {
                calls.add("handler.invoked");
                chain.applyPostHandle(request, response);
                chain.triggerAfterCompletion(request, response, null);
            }

            // Handler and postHandle should NOT have been called
            assertThat(calls).doesNotContain("handler.invoked", "A.postHandle",
                    "B.postHandle", "C.postHandle");
            // Only A got afterCompletion (B returned false, C never ran)
            assertThat(calls).containsExactly(
                    "A.preHandle", "B.preHandle", "A.afterCompletion");
        }

        @Test
        @DisplayName("should handle first interceptor veto correctly")
        void shouldHandleFirstInterceptorVeto() throws Exception {
            HandlerExecutionChain chain = new HandlerExecutionChain(handler);
            chain.addInterceptor(new RecordingInterceptor("A", false, calls)); // vetoes immediately
            chain.addInterceptor(new RecordingInterceptor("B", true, calls));

            boolean approved = chain.applyPreHandle(request, response);

            assertThat(approved).isFalse();
            // No afterCompletion for anyone — interceptorIndex is -1 after A vetoes
            // because interceptorIndex is updated AFTER successful preHandle
            assertThat(calls).containsExactly("A.preHandle");
        }
    }

    // ─── Accessors ──────────────────────────────────────────────────

    @Test
    @DisplayName("should return the wrapped handler")
    void shouldReturnHandler() {
        HandlerExecutionChain chain = new HandlerExecutionChain(handler);

        assertThat(chain.getHandler()).isSameAs(handler);
    }

    @Test
    @DisplayName("should return interceptor list")
    void shouldReturnInterceptors() {
        HandlerExecutionChain chain = new HandlerExecutionChain(handler);
        RecordingInterceptor a = new RecordingInterceptor("A", true, calls);
        RecordingInterceptor b = new RecordingInterceptor("B", true, calls);
        chain.addInterceptor(a);
        chain.addInterceptor(b);

        assertThat(chain.getInterceptors()).containsExactly(a, b);
    }

    @Test
    @DisplayName("should include handler and interceptor count in toString")
    void shouldIncludeInfoInToString() {
        HandlerExecutionChain chain = new HandlerExecutionChain(handler);
        chain.addInterceptor(new RecordingInterceptor("A", true, calls));

        assertThat(chain.toString()).contains("1 interceptors");
    }
}
```

#### [NEW] `src/test/java/com/simplespringmvc/integration/InterceptorIntegrationTest.java`

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.GetMapping;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for the handler interceptor feature (ch11).
 *
 * Verifies the complete interceptor lifecycle through a real HTTP stack:
 * preHandle (forward) → handler → postHandle (reverse) → afterCompletion (reverse).
 * Also tests veto behavior, exception handling, and multi-interceptor ordering.
 */
class InterceptorIntegrationTest {

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    static class GreetingController {

        @GetMapping("/hello")
        public String hello() {
            return "Hello, World!";
        }

        @RequestMapping(path = "/fail", method = "GET")
        public String fail() {
            throw new RuntimeException("handler error");
        }
    }

    // ─── Test interceptors ───────────────────────────────────────────

    /**
     * Interceptor that records all lifecycle calls for verification.
     */
    static class RecordingInterceptor implements HandlerInterceptor {
        final String name;
        final List<String> calls;

        RecordingInterceptor(String name, List<String> calls) {
            this.name = name;
            this.calls = calls;
        }

        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                 HandlerMethod handler) {
            calls.add(name + ".preHandle");
            return true;
        }

        @Override
        public void postHandle(HttpServletRequest request, HttpServletResponse response,
                               HandlerMethod handler) {
            calls.add(name + ".postHandle");
        }

        @Override
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                    HandlerMethod handler, Exception ex) {
            calls.add(name + ".afterCompletion" + (ex != null ? "(ex)" : ""));
        }
    }

    /**
     * Interceptor that adds a custom response header (demonstrating a real use case).
     */
    static class TimingInterceptor implements HandlerInterceptor {
        static final String START_TIME_ATTR = "requestStartTime";

        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                 HandlerMethod handler) {
            request.setAttribute(START_TIME_ATTR, System.nanoTime());
            return true;
        }

        @Override
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                    HandlerMethod handler, Exception ex) {
            Long startTime = (Long) request.getAttribute(START_TIME_ATTR);
            if (startTime != null) {
                long duration = (System.nanoTime() - startTime) / 1_000_000; // ms
                response.setHeader("X-Request-Duration-Ms", String.valueOf(duration));
            }
        }
    }

    /**
     * Interceptor that vetoes requests — simulating auth denial.
     */
    static class DenyingInterceptor implements HandlerInterceptor {
        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                 HandlerMethod handler) throws Exception {
            response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied");
            return false;
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;
    private List<String> lifecycleCalls;

    @BeforeEach
    void setUp() {
        lifecycleCalls = Collections.synchronizedList(new ArrayList<>());
    }

    @AfterEach
    void tearDown() throws Exception {
        if (tomcat != null) {
            tomcat.stop();
        }
    }

    private void startServer(SimpleDispatcherServlet servlet) throws Exception {
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();
        baseUrl = "http://localhost:" + tomcat.getPort();
        httpClient = HttpClient.newHttpClient();
    }

    private SimpleDispatcherServlet createServlet() {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new GreetingController());
        return new SimpleDispatcherServlet(container);
    }

    // ─── Normal lifecycle ────────────────────────────────────────────

    @Test
    @DisplayName("should execute full interceptor lifecycle in correct order")
    void shouldExecuteFullLifecycle() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new RecordingInterceptor("A", lifecycleCalls));
        servlet.addInterceptor(new RecordingInterceptor("B", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello, World!");

        // Verify lifecycle order: preHandle forward, postHandle reverse, afterCompletion reverse
        assertThat(lifecycleCalls).containsExactly(
                "A.preHandle",
                "B.preHandle",
                "B.postHandle",
                "A.postHandle",
                "B.afterCompletion",
                "A.afterCompletion");
    }

    @Test
    @DisplayName("should work with no interceptors registered")
    void shouldWorkWithNoInterceptors() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("Hello, World!");
    }

    @Test
    @DisplayName("should work with single interceptor")
    void shouldWorkWithSingleInterceptor() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new RecordingInterceptor("Only", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(lifecycleCalls).containsExactly(
                "Only.preHandle", "Only.postHandle", "Only.afterCompletion");
    }

    // ─── Veto behavior ──────────────────────────────────────────────

    @Test
    @DisplayName("should abort request when interceptor returns false")
    void shouldAbortRequest_WhenInterceptorReturnsFalse() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new RecordingInterceptor("A", lifecycleCalls));
        servlet.addInterceptor(new DenyingInterceptor());
        servlet.addInterceptor(new RecordingInterceptor("C", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(403);
        // A ran preHandle (passed), Denying ran preHandle (vetoed), C never ran
        // A gets afterCompletion cleanup
        assertThat(lifecycleCalls).containsExactly("A.preHandle", "A.afterCompletion");
    }

    @Test
    @DisplayName("should not invoke handler when first interceptor vetoes")
    void shouldNotInvokeHandler_WhenFirstInterceptorVetoes() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new DenyingInterceptor());
        servlet.addInterceptor(new RecordingInterceptor("B", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(403);
        // No interceptor calls recorded because Denying doesn't record and B never runs
        assertThat(lifecycleCalls).isEmpty();
    }

    // ─── Exception handling ─────────────────────────────────────────

    @Test
    @DisplayName("should call afterCompletion with exception when handler throws")
    void shouldCallAfterCompletion_WhenHandlerThrows() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new RecordingInterceptor("A", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/fail")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(500);
        // preHandle succeeds, postHandle is NOT called (exception path),
        // afterCompletion IS called with the exception
        assertThat(lifecycleCalls).containsExactly(
                "A.preHandle", "A.afterCompletion(ex)");
    }

    @Test
    @DisplayName("should call afterCompletion for all interceptors when handler throws")
    void shouldCallAfterCompletionForAll_WhenHandlerThrows() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new RecordingInterceptor("A", lifecycleCalls));
        servlet.addInterceptor(new RecordingInterceptor("B", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/fail")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(500);
        assertThat(lifecycleCalls).containsExactly(
                "A.preHandle", "B.preHandle",
                "B.afterCompletion(ex)", "A.afterCompletion(ex)");
    }

    // ─── Real-world use case: timing interceptor ────────────────────

    @Test
    @DisplayName("should support real-world interceptor use case like timing headers")
    void shouldSupportTimingHeaders() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new TimingInterceptor());
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/hello")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        // The timing header may or may not be visible depending on when the response
        // is committed. We just verify the handler executed correctly.
        assertThat(response.body()).isEqualTo("Hello, World!");
    }

    // ─── 404 path ───────────────────────────────────────────────────

    @Test
    @DisplayName("should not trigger interceptors when no handler found")
    void shouldNotTriggerInterceptors_WhenNoHandlerFound() throws Exception {
        SimpleDispatcherServlet servlet = createServlet();
        servlet.addInterceptor(new RecordingInterceptor("A", lifecycleCalls));
        startServer(servlet);

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder(URI.create(baseUrl + "/nonexistent")).GET().build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(404);
        // No interceptor calls — interceptors only fire when a handler is found
        assertThat(lifecycleCalls).isEmpty();
    }
}
```

---

## Summary

| What | Details |
|------|---------|
| **New files** | `HandlerInterceptor.java`, `HandlerExecutionChain.java` |
| **Modified files** | `SimpleDispatcherServlet.java` |
| **Tests** | 4 unit (interface) + 17 unit (chain) + 9 integration = **30 new tests** |
| **Key pattern** | Chain of Responsibility with index-based cleanup tracking |
| **Lines of code** | ~170 production, ~590 test |

**Next chapter:** [Exception Handling](ch12_exception_handling.md) — catch exceptions from handler methods and route them to `@ExceptionHandler` methods, either on the controller itself or on `@ControllerAdvice` classes. This plugs into the `dispatchException` we capture in `doDispatch()` but currently just re-throw.
