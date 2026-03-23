# Chapter 27: Async Request Handling (DeferredResult, Callable)

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---|---|---|
| Controller methods execute synchronously -- the Servlet thread is blocked until the handler returns and the response is written | Long-running operations (database queries, remote API calls) tie up Servlet threads, limiting the server's ability to handle concurrent requests | Allow controller methods to return `Callable<T>`, `DeferredResult<T>`, or `CompletionStage<T>` to free the Servlet thread while a background thread computes the result |

---

## 27.1 The Integration Point

There are two integration points where the existing dispatch pipeline gains async awareness.

### Integration Point 1: SimpleHandlerAdapter.handle() -- detecting re-dispatch

When the Servlet container re-dispatches after async processing, the same handler is matched (same URL). The adapter must detect this re-dispatch and process the saved concurrent result instead of invoking the controller again.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`

```java
@Override
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                           Object handler) throws Exception {
    HandlerMethod handlerMethod = (HandlerMethod) handler;

    // ch27: Check if this is a re-dispatch after async processing.
    WebAsyncManager asyncManager = WebAsyncManager.getAsyncManager(request);
    if (asyncManager.hasConcurrentResult()) {
        Object result = asyncManager.getConcurrentResult();
        asyncManager.clearConcurrentResult();

        // If the async computation failed, rethrow the exception.
        if (result instanceof Throwable t) {
            if (t instanceof Exception ex) {
                throw ex;
            }
            throw new RuntimeException(t);
        }

        // Wrap the handler method so getReturnType() returns the actual
        // result type instead of Callable/DeferredResult.
        HandlerMethod wrappedMethod = handlerMethod.wrapConcurrentResult(result);
        return returnValueHandlers.handleReturnValue(
                result, wrappedMethod, request, response);
    }

    // ... normal synchronous path continues below
```

This maps to `RequestMappingHandlerAdapter.invokeHandlerMethod()` (line 885-901) which checks `asyncManager.hasConcurrentResult()` and creates a `ConcurrentResultHandlerMethod` wrapper.

Three things happen in this block:

1. **Retrieve and clear the result.** The `WebAsyncManager` stored the result from the async computation (T2). We grab it and reset the state machine so the manager is ready for future requests.

2. **Rethrow Throwables.** If the async computation threw an exception, the adapter rethrows it so the exception resolver chain in `doDispatch()` handles it the same way as a synchronous exception.

3. **Wrap the handler method.** `wrapConcurrentResult()` creates a `ConcurrentResultHandlerMethod` that overrides `getReturnType()` to return the actual result type (e.g., `String.class`) instead of the declared type (e.g., `Callable.class`). This prevents `CallableReturnValueHandler` from matching again, which would create an infinite async loop.

### Integration Point 2: SimpleDispatcherServlet.doDispatch() -- async lifecycle check

After the handler adapter returns, the dispatcher must check whether async processing was started. If so, it calls `afterConcurrentHandlingStarted` on interceptors instead of `postHandle`/`afterCompletion`, and returns immediately.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
// Step 3: Find a HandlerAdapter and invoke the handler.
HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
mv = adapter.handle(processedRequest, response, mappedHandler.getHandler());

// ch27: Check if WebAsyncManager started async processing
WebAsyncManager asyncManager =
        WebAsyncManager.getAsyncManager(processedRequest);
if (asyncManager.isConcurrentHandlingStarted()) {
    if (mappedHandler != null) {
        mappedHandler.applyAfterConcurrentHandlingStarted(
                processedRequest, response);
    }
    return;
}

// ch22: If async processing started by other means (SSE emitter)
if (processedRequest.isAsyncStarted()) {
    return;
}

// Step 4: Apply interceptor postHandle -- reverse order
mappedHandler.applyPostHandle(processedRequest, response);
```

This maps to `DispatcherServlet.doDispatch()` (line 965) where the `asyncManager.isConcurrentHandlingStarted()` check replaces the simple `request.isAsyncStarted()` check. The previous `request.isAsyncStarted()` check (from ch22, for SSE) is still present as a fallback for direct async contexts that bypass the `WebAsyncManager`.

This connects the async return value pipeline to the re-dispatch lifecycle. We need to build:

- **DeferredResult** -- the async result container with thread-safe rendezvous
- **WebAsyncManager** -- the async lifecycle coordinator with state machine
- **AsyncHandlerInterceptor** -- extension for async-aware interceptors
- **CallableReturnValueHandler** -- starts async for Callable returns
- **DeferredResultReturnValueHandler** -- starts async for DeferredResult/CompletionStage returns
- **ConcurrentResultHandlerMethod** -- wrapper that changes the apparent return type on re-dispatch

Two key design decisions:

1. **Using ConcurrentResultHandlerMethod to change the apparent return type prevents infinite async loops on re-dispatch.** Without it, a `Callable<String>` handler would match `CallableReturnValueHandler` again on re-dispatch (because the declared return type is still `Callable`), starting async processing a second time, ad infinitum.

2. **Introducing `getReturnType()` on HandlerMethod creates the indirection needed for all return value handlers to work correctly with async results.** All return value handlers must call `handlerMethod.getReturnType()` instead of `handlerMethod.getMethod().getReturnType()`. The base implementation delegates to `Method.getReturnType()`, but `ConcurrentResultHandlerMethod` overrides it.

---

## 27.2 DeferredResult -- The Async Result Container

`DeferredResult<T>` is a container that gives the application complete control over threading. The controller creates a `DeferredResult`, hands it off to whatever async mechanism it likes (message listener, scheduled task, remote service callback), and returns immediately. When the result is ready, any thread calls `setResult()`.

**Creating:** `src/main/java/com/simplespringmvc/async/DeferredResult.java`

The core challenge is coordinating two threads that race to complete a handshake:

- **T1 (Servlet thread)** calls `setResultHandler()` to register the dispatch callback
- **T2 (Application thread)** calls `setResult()` when the result is ready

Whichever arrives second triggers the actual dispatch. This is the **rendezvous pattern**.

```java
private static final Object RESULT_NONE = new Object();

private volatile Object result = RESULT_NONE;
private volatile boolean expired;
private DeferredResultHandler resultHandler;
```

A sentinel `RESULT_NONE` is used instead of `null` because `null` is a valid result -- a controller might intentionally return null. The `volatile` keyword on `result` ensures cross-thread visibility: T2 writes, T1 (or T3 on re-dispatch) reads.

The `setResultInternal()` method uses double-checked locking:

```java
private boolean setResultInternal(Object result) {
    // Fast path: already set or expired -- no synchronization needed
    if (isSetOrExpired()) {
        return false;
    }

    DeferredResultHandler handlerToUse;
    synchronized (this) {
        // Re-check under lock (double-checked locking)
        if (isSetOrExpired()) {
            return false;
        }
        this.result = result;
        handlerToUse = this.resultHandler;
    }

    // Call handler OUTSIDE synchronized block to avoid deadlock
    if (handlerToUse != null) {
        handlerToUse.handleResult(result);
    }
    return true;
}
```

The critical detail: `handlerToUse.handleResult(result)` happens **outside** the synchronized block. The real framework does this to avoid holding the `DeferredResult` monitor while the handler acquires Servlet container locks -- a potential deadlock scenario. Maps to `DeferredResult.setResultInternal()` (line 239-256).

The `setResultHandler()` method mirrors this pattern -- if a result was already set before the handler was registered, it calls the handler immediately (outside the lock):

```java
public void setResultHandler(DeferredResultHandler resultHandler) {
    Object resultToHandle;
    synchronized (this) {
        if (this.result != RESULT_NONE) {
            resultToHandle = this.result;
        } else {
            this.resultHandler = resultHandler;
            return;
        }
    }
    // Call handler outside lock
    resultHandler.handleResult(resultToHandle);
}
```

Maps to `DeferredResult.setResultHandler()` (line 190). The `DeferredResultHandler` is a functional interface that `WebAsyncManager` implements via a lambda pointing to `setConcurrentResultAndDispatch()`.

The lifecycle callbacks (`onTimeout`, `onError`, `onCompletion`) and their corresponding handlers are called by `WebAsyncManager` through the Servlet `AsyncListener`:

```java
boolean handleTimeout() {
    if (this.timeoutCallback != null) {
        this.timeoutCallback.run();
    }
    if (this.timeoutResult != null) {
        return setResultInternal(this.timeoutResult.get());
    }
    return false;
}

void handleCompletion() {
    this.expired = true;
    if (this.completionCallback != null) {
        this.completionCallback.run();
    }
}
```

Once `handleCompletion()` marks the `DeferredResult` as expired, no new results are accepted -- `isSetOrExpired()` returns true and `setResultInternal()` exits on the fast path.

---

## 27.3 WebAsyncManager -- The Lifecycle Coordinator

`WebAsyncManager` is the central coordinator for async request processing. One instance per request, stored as a request attribute, it bridges between controller return values (`Callable`/`DeferredResult`) and the Servlet container's async API.

**Creating:** `src/main/java/com/simplespringmvc/async/WebAsyncManager.java`

### The State Machine

```java
private enum State { NOT_STARTED, ASYNC_PROCESSING, RESULT_SET }

private final AtomicReference<State> state = new AtomicReference<>(State.NOT_STARTED);
private volatile Object concurrentResult = RESULT_NONE;
```

The lifecycle:

```
NOT_STARTED --> ASYNC_PROCESSING --> RESULT_SET --> (re-dispatch clears back to NOT_STARTED)
```

Transitions use `AtomicReference.compareAndSet()` to ensure exactly one thread can move the state forward. Maps to `WebAsyncManager.state` (line 92).

### Per-request Singleton

```java
public static WebAsyncManager getAsyncManager(HttpServletRequest request) {
    WebAsyncManager manager =
            (WebAsyncManager) request.getAttribute(WEB_ASYNC_MANAGER_ATTRIBUTE);
    if (manager == null) {
        manager = new WebAsyncManager();
        request.setAttribute(WEB_ASYNC_MANAGER_ATTRIBUTE, manager);
    }
    return manager;
}
```

Maps to `WebAsyncUtils.getAsyncManager(ServletRequest)` (line 37). The request attribute ensures the same manager is used throughout a request's lifecycle, including across the initial dispatch and re-dispatch.

### Callable Processing

```java
public void startCallableProcessing(Callable<?> callable,
                                     HttpServletRequest request,
                                     HttpServletResponse response) {
    if (!state.compareAndSet(State.NOT_STARTED, State.ASYNC_PROCESSING)) {
        throw new IllegalStateException(
                "Async processing already started for this request");
    }

    asyncContext = request.startAsync(request, response);
    asyncContext.setTimeout(0);

    asyncContext.addListener(new AsyncListener() { /* timeout/error/complete */ });

    taskExecutor.submit(() -> {
        try {
            Object result = callable.call();
            setConcurrentResultAndDispatch(result);
        } catch (Throwable ex) {
            setConcurrentResultAndDispatch(ex);
        }
    });
}
```

Maps to `WebAsyncManager.startCallableProcessing()` (line 186). The CAS transition from `NOT_STARTED` to `ASYNC_PROCESSING` ensures only one async operation per request. The `Callable` is submitted to a shared cached thread pool, and on completion (success or failure), `setConcurrentResultAndDispatch()` saves the result and triggers re-dispatch.

### DeferredResult Processing

```java
public void startDeferredResultProcessing(DeferredResult<?> deferredResult,
                                           HttpServletRequest request,
                                           HttpServletResponse response) {
    if (!state.compareAndSet(State.NOT_STARTED, State.ASYNC_PROCESSING)) {
        throw new IllegalStateException(
                "Async processing already started for this request");
    }

    asyncContext = request.startAsync(request, response);

    if (deferredResult.getTimeoutValue() != null) {
        asyncContext.setTimeout(deferredResult.getTimeoutValue());
    } else {
        asyncContext.setTimeout(0);
    }

    asyncContext.addListener(new AsyncListener() { /* delegates to DeferredResult lifecycle */ });

    deferredResult.setResultHandler(this::setConcurrentResultAndDispatch);
}
```

Maps to `WebAsyncManager.startDeferredResultProcessing()` (line 277). The key difference from Callable: the framework does not submit work to a thread pool. Instead, it wires the `DeferredResult`'s result handler to point at `setConcurrentResultAndDispatch()`. When the application thread eventually calls `deferredResult.setResult(value)`, the handler fires, which saves the result and dispatches.

### The Dispatch Trigger

```java
private synchronized void setConcurrentResultAndDispatch(Object result) {
    if (!state.compareAndSet(State.ASYNC_PROCESSING, State.RESULT_SET)) {
        return;
    }
    this.concurrentResult = result;
    asyncContext.dispatch();
}
```

Maps to `WebAsyncManager.setConcurrentResultAndDispatch()` (line 340). The `synchronized` block plus CAS ensures only one thread can set the result. `asyncContext.dispatch()` tells the Servlet container to re-dispatch the request -- the container calls `DispatcherServlet.service()` again with `DispatcherType.ASYNC`.

---

## 27.4 AsyncHandlerInterceptor

When a handler starts async processing, the normal interceptor lifecycle (`postHandle`/`afterCompletion`) should not run on the initial Servlet thread -- the response is not ready yet. Instead, async-aware interceptors get a callback to clean up thread-locals before the thread is released.

**Creating:** `src/main/java/com/simplespringmvc/interceptor/AsyncHandlerInterceptor.java`

```java
public interface AsyncHandlerInterceptor extends HandlerInterceptor {

    default void afterConcurrentHandlingStarted(HttpServletRequest request,
                                                 HttpServletResponse response,
                                                 HandlerMethod handler) throws Exception {
    }
}
```

Maps to `org.springframework.web.servlet.AsyncHandlerInterceptor` (line 49). The lifecycle with async handlers:

```
Initial request (T1):
  preHandle() --> handler returns Callable/DeferredResult
  --> afterConcurrentHandlingStarted() (NOT postHandle/afterCompletion)
  --> T1 released

Re-dispatch (T3, after async completes):
  preHandle() --> handler adapter processes saved result
  --> postHandle() --> afterCompletion()
```

This allows interceptors to clean up thread-locals on T1 and re-establish them on T3.

---

## 27.5 Return Value Handlers

### CallableReturnValueHandler

Detects `Callable` return types and delegates to `WebAsyncManager.startCallableProcessing()`.

**Creating:** `src/main/java/com/simplespringmvc/adapter/CallableReturnValueHandler.java`

```java
public class CallableReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return Callable.class.isAssignableFrom(handlerMethod.getReturnType());
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        if (returnValue == null) {
            return null;
        }

        Callable<?> callable = (Callable<?>) returnValue;
        WebAsyncManager asyncManager = WebAsyncManager.getAsyncManager(request);
        asyncManager.startCallableProcessing(callable, request, response);

        return null; // response completed via async re-dispatch
    }
}
```

Maps to `CallableMethodReturnValueHandler` (line 47-55). The handler returns `null` because the response will be completed asynchronously via re-dispatch. The `supportsReturnType` check uses `handlerMethod.getReturnType()` (not `handlerMethod.getMethod().getReturnType()`), which is essential for the re-dispatch mechanism to work correctly.

### DeferredResultReturnValueHandler

Detects `DeferredResult` and `CompletionStage` return types. `CompletionStage` (and its common implementation `CompletableFuture`) is transparently supported by wrapping it in a `DeferredResult`.

**Creating:** `src/main/java/com/simplespringmvc/adapter/DeferredResultReturnValueHandler.java`

```java
@Override
public boolean supportsReturnType(HandlerMethod handlerMethod) {
    Class<?> type = handlerMethod.getReturnType();
    return DeferredResult.class.isAssignableFrom(type)
            || CompletionStage.class.isAssignableFrom(type);
}
```

The `CompletionStage` adaptation bridges to `DeferredResult` via `whenComplete`:

```java
private DeferredResult<Object> adaptCompletionStage(CompletionStage<?> completionStage) {
    DeferredResult<Object> result = new DeferredResult<>();
    completionStage.whenComplete((value, ex) -> {
        if (ex != null) {
            if (ex instanceof CompletionException && ex.getCause() != null) {
                ex = ex.getCause();
            }
            result.setErrorResult(ex);
        } else {
            result.setResult(value);
        }
    });
    return result;
}
```

Maps to `DeferredResultMethodReturnValueHandler.adaptCompletionStage()` (line 93). `CompletionException` is unwrapped to get the actual cause, matching how the real framework handles it.

---

## 27.6 HandlerMethod Enhancement -- ConcurrentResultHandlerMethod

The `HandlerMethod` class gained two additions: a `getReturnType()` method and a `ConcurrentResultHandlerMethod` inner class.

**Modifying:** `src/main/java/com/simplespringmvc/mapping/HandlerMethod.java`

```java
public Class<?> getReturnType() {
    return method.getReturnType();
}

public HandlerMethod wrapConcurrentResult(Object result) {
    return new ConcurrentResultHandlerMethod(this, result);
}

private static class ConcurrentResultHandlerMethod extends HandlerMethod {

    private final Class<?> resultType;

    ConcurrentResultHandlerMethod(HandlerMethod original, Object result) {
        super(original.getBean(), original.getMethod());
        this.resultType = (result != null) ? result.getClass() : Void.class;
    }

    @Override
    public Class<?> getReturnType() {
        return resultType;
    }
}
```

Maps to `ServletInvocableHandlerMethod.ConcurrentResultHandlerMethod` (line 231).

**Why is this needed?** Consider a concrete example:

1. A controller method is declared as `public Callable<String> search() { ... }`
2. On the initial request, `CallableReturnValueHandler.supportsReturnType()` checks `handlerMethod.getReturnType()` -- it returns `Callable.class`, so the handler matches.
3. The `Callable` executes on a background thread and produces `"search results"`.
4. The container re-dispatches. The same handler method is matched (same URL `/search`). Its declared return type is still `Callable`.
5. **Without the wrapper:** `CallableReturnValueHandler` matches again, starts a new async operation, which re-dispatches again -- infinite loop.
6. **With the wrapper:** `SimpleHandlerAdapter` wraps the handler in `ConcurrentResultHandlerMethod` with the result `"search results"`. Now `getReturnType()` returns `String.class`. `CallableReturnValueHandler` does not match. `ViewNameReturnValueHandler` or `ResponseBodyReturnValueHandler` matches instead, and the result is written to the response.

---

## 27.7 HandlerExecutionChain Enhancement

The execution chain gained a method to notify async-aware interceptors when concurrent processing starts.

**Modifying:** `src/main/java/com/simplespringmvc/interceptor/HandlerExecutionChain.java`

```java
public void applyAfterConcurrentHandlingStarted(HttpServletRequest request,
                                                  HttpServletResponse response) {
    for (int i = this.interceptorIndex; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (interceptor instanceof AsyncHandlerInterceptor asyncInterceptor) {
            try {
                asyncInterceptor.afterConcurrentHandlingStarted(
                        request, response, this.handler);
            } catch (Throwable ex) {
                System.err.println(
                        "AsyncHandlerInterceptor.afterConcurrentHandlingStarted threw: " + ex);
            }
        }
    }
}
```

Maps to `HandlerExecutionChain.applyAfterConcurrentHandlingStarted()` (line 193). This method iterates interceptors in reverse order from `interceptorIndex` (only interceptors whose `preHandle` succeeded). Non-async interceptors are silently skipped. Each call is wrapped in try/catch so one interceptor's failure does not prevent others from cleaning up -- the same safety pattern used in `triggerAfterCompletion`.

---

## 27.8 Return Value Handler Updates

Three existing return value handlers were updated to use `handlerMethod.getReturnType()` instead of `handlerMethod.getMethod().getReturnType()`. This is necessary so that `ConcurrentResultHandlerMethod`'s override takes effect on re-dispatch.

### SseEmitterReturnValueHandler

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SseEmitterReturnValueHandler.java`

```java
@Override
public boolean supportsReturnType(HandlerMethod handlerMethod) {
    return SseEmitter.class.isAssignableFrom(handlerMethod.getReturnType());
}
```

### ViewNameReturnValueHandler

**Modifying:** `src/main/java/com/simplespringmvc/adapter/ViewNameReturnValueHandler.java`

```java
@Override
public boolean supportsReturnType(HandlerMethod handlerMethod) {
    Class<?> returnType = handlerMethod.getReturnType();
    return CharSequence.class.isAssignableFrom(returnType);
}
```

### ModelAndViewReturnValueHandler

**Modifying:** `src/main/java/com/simplespringmvc/adapter/ModelAndViewReturnValueHandler.java`

```java
@Override
public boolean supportsReturnType(HandlerMethod handlerMethod) {
    return ModelAndView.class.isAssignableFrom(handlerMethod.getReturnType());
}
```

All three now call `handlerMethod.getReturnType()` which is overridable, rather than going directly to `handlerMethod.getMethod().getReturnType()` which always returns the declared Java method return type. Without this change, a `Callable<String>` method that produces a `String` result on re-dispatch would not match `ViewNameReturnValueHandler` (because `getMethod().getReturnType()` still returns `Callable`).

---

## 27.9 Try It Yourself

Before looking at the tests, try implementing these two challenges:

### Challenge 1: Implement the DeferredResult rendezvous

Given the `DeferredResult` class with `setResult(T)` and `setResultHandler(DeferredResultHandler)`, implement the thread coordination so that:
- If the result handler is registered first and the result arrives later, the handler is called
- If the result arrives first and the handler is registered later, the handler is called immediately
- Only the first result is accepted
- The handler callback happens outside the synchronized block

**Hint:** Use a `volatile` field for the result with a sentinel value (not `null`), and a `synchronized` block around the state check-and-set. Grab a reference to the handler inside the lock, call it outside. Both `setResultInternal()` and `setResultHandler()` follow this same pattern.

### Challenge 2: Handle the re-dispatch in the adapter

Given that `WebAsyncManager.hasConcurrentResult()` returns true on re-dispatch, write the code in `SimpleHandlerAdapter.handle()` that:
- Retrieves the result from the manager
- Clears the manager state
- Rethrows if the result is a `Throwable`
- Otherwise processes the result through the normal return value handler pipeline

**Hint:** You need a `ConcurrentResultHandlerMethod` wrapper that changes `getReturnType()` to the actual result type. Call `handlerMethod.wrapConcurrentResult(result)` and pass the wrapped method to `returnValueHandlers.handleReturnValue()`.

---

## 27.10 Tests

### DeferredResult Unit Tests

These tests verify the rendezvous pattern -- both orderings of result and handler arrival.

**File:** `src/test/java/com/simplespringmvc/async/DeferredResultTest.java`

```java
@Test
void shouldAcceptResult_WhenNotYetSet() {
    DeferredResult<String> dr = new DeferredResult<>();

    boolean accepted = dr.setResult("hello");

    assertThat(accepted).isTrue();
    assertThat(dr.hasResult()).isTrue();
    assertThat(dr.getResult()).isEqualTo("hello");
}

@Test
void shouldRejectSecondResult_WhenAlreadySet() {
    DeferredResult<String> dr = new DeferredResult<>();
    dr.setResult("first");

    boolean accepted = dr.setResult("second");

    assertThat(accepted).isFalse();
    assertThat(dr.getResult()).isEqualTo("first");
}
```

The first test verifies basic result setting. The second verifies the "first writer wins" semantics -- once a result is set, subsequent calls return false.

```java
@Test
void shouldCallHandler_WhenResultArrrivesAfterHandler() {
    DeferredResult<String> dr = new DeferredResult<>();
    AtomicReference<Object> handledResult = new AtomicReference<>();

    dr.setResultHandler(handledResult::set);       // Handler arrives first
    dr.setResult("async value");                    // Then result arrives

    assertThat(handledResult.get()).isEqualTo("async value");
}

@Test
void shouldCallHandler_WhenResultArrivesBeforeHandler() {
    DeferredResult<String> dr = new DeferredResult<>();
    AtomicReference<Object> handledResult = new AtomicReference<>();

    dr.setResult("early result");                   // Result arrives first
    dr.setResultHandler(handledResult::set);         // Then handler arrives

    assertThat(handledResult.get()).isEqualTo("early result");
}
```

These two tests verify both sides of the rendezvous. The handler is called regardless of which side arrives first.

```java
@Test
void shouldCoordinateAcrossThreads_WhenResultSetFromAnotherThread() throws Exception {
    DeferredResult<String> dr = new DeferredResult<>();
    CountDownLatch latch = new CountDownLatch(1);
    AtomicReference<Object> handledResult = new AtomicReference<>();

    dr.setResultHandler(result -> {
        handledResult.set(result);
        latch.countDown();
    });

    Thread worker = new Thread(() -> dr.setResult("from worker"));
    worker.start();

    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
    assertThat(handledResult.get()).isEqualTo("from worker");
}
```

This test verifies the cross-thread coordination using a `CountDownLatch` to synchronize the test thread with the worker thread.

### WebAsyncManager Unit Tests

**File:** `src/test/java/com/simplespringmvc/async/WebAsyncManagerTest.java`

```java
@Test
void shouldReturnSameInstance_WhenCalledTwiceForSameRequest() {
    WebAsyncManager first = WebAsyncManager.getAsyncManager(request);
    WebAsyncManager second = WebAsyncManager.getAsyncManager(request);

    assertThat(first).isSameAs(second);
}
```

Verifies the per-request singleton semantics -- the same manager is returned for the same request.

```java
@Test
void shouldStartAsyncAndExecuteCallable_WhenCallableReturnsValue() throws Exception {
    WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
    Callable<String> callable = () -> "async result";

    manager.startCallableProcessing(callable, request, response);

    assertThat(manager.isConcurrentHandlingStarted()).isTrue();

    Thread.sleep(500); // Give the background thread time to execute

    assertThat(manager.hasConcurrentResult()).isTrue();
    assertThat(manager.getConcurrentResult()).isEqualTo("async result");
}
```

Verifies the Callable processing lifecycle: start async, execute on a background thread, save the result.

```java
@Test
void shouldSaveResult_WhenDeferredResultIsSet() throws Exception {
    WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
    DeferredResult<String> dr = new DeferredResult<>();

    manager.startDeferredResultProcessing(dr, request, response);
    dr.setResult("deferred value");

    assertThat(manager.hasConcurrentResult()).isTrue();
    assertThat(manager.getConcurrentResult()).isEqualTo("deferred value");
}
```

Verifies the DeferredResult processing lifecycle: start async, wire the handler, set result -- the result flows through to the manager.

### CallableReturnValueHandler Tests

**File:** `src/test/java/com/simplespringmvc/adapter/CallableReturnValueHandlerTest.java`

```java
@Test
void shouldNotMatchOnReDispatch_WhenWrappedInConcurrentResultHandlerMethod() throws Exception {
    HandlerMethod hm = new HandlerMethod(new TestController(),
            TestController.class.getMethod("callableMethod"));

    HandlerMethod wrapped = hm.wrapConcurrentResult("async result");

    assertThat(handler.supportsReturnType(wrapped)).isFalse();
}
```

The most important test: verifies that `CallableReturnValueHandler` does NOT match on re-dispatch when the handler method is wrapped in `ConcurrentResultHandlerMethod`. This prevents the infinite async loop.

### DeferredResultReturnValueHandler Tests

**File:** `src/test/java/com/simplespringmvc/adapter/DeferredResultReturnValueHandlerTest.java`

```java
@Test
void shouldSupportCompletionStageReturnType() throws Exception {
    HandlerMethod hm = new HandlerMethod(new TestController(),
            TestController.class.getMethod("completionStageMethod"));

    assertThat(handler.supportsReturnType(hm)).isTrue();
}

@Test
void shouldSupportCompletableFutureReturnType() throws Exception {
    HandlerMethod hm = new HandlerMethod(new TestController(),
            TestController.class.getMethod("completableFutureMethod"));

    assertThat(handler.supportsReturnType(hm)).isTrue();
}
```

Verifies that both `CompletionStage` and its subclass `CompletableFuture` are supported transparently.

### Integration Tests

**File:** `src/test/java/com/simplespringmvc/integration/AsyncRequestHandlingIntegrationTest.java`

The integration tests run against a real embedded Tomcat server, testing the full lifecycle.

```java
@Test
void shouldReturnAsyncResult_WhenCallableCompletesSuccessfully() throws Exception {
    HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/async/callable"))
            .header("Accept", "text/plain")
            .GET().build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("callable result");
}
```

This test covers the full Callable flow: HTTP request hits Tomcat, `DispatcherServlet` dispatches to the handler, handler returns a `Callable`, `CallableReturnValueHandler` starts async processing, the Callable executes on a background thread, `WebAsyncManager` saves the result and dispatches, `SimpleHandlerAdapter` detects the re-dispatch and processes the saved result, the response is written.

```java
@Test
void shouldReturnAsyncResult_WhenDeferredResultSetFromAnotherThread() throws Exception {
    CompletableFuture<HttpResponse<String>> futureResponse = CompletableFuture.supplyAsync(() -> {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(baseUrl + "/async/deferred-delayed"))
                    .header("Accept", "text/plain")
                    .GET().build();
            return client.send(request, HttpResponse.BodyHandlers.ofString());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    });

    Thread.sleep(500);

    if (pendingResult != null) {
        pendingResult.setResult("delayed result");
    }

    HttpResponse<String> response = futureResponse.get();

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).isEqualTo("delayed result");
}
```

This test demonstrates the key value proposition of `DeferredResult`: the HTTP request blocks waiting for a response while the test thread (simulating an external event like a message listener) completes the `DeferredResult` from a completely different thread. The controller uses a static `pendingResult` field that the test populates after a delay.

```java
@Test
void shouldReturnResult_WhenCompletableFutureCompletes() throws Exception {
    HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/async/completable-future"))
            .header("Accept", "application/json")
            .GET().build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).contains("\"message\"");
    assertThat(response.body()).contains("future result");
}
```

Verifies end-to-end `CompletionStage` support -- the handler returns a `CompletableFuture`, which is adapted to a `DeferredResult` transparently.

---

## 27.11 Why This Works

> **The Rendezvous Pattern.** `DeferredResult` coordinates two threads that race to complete a handshake. The `volatile` field provides cross-thread visibility (T2's write is seen by T1), and the `synchronized` block provides atomicity (check-then-set is atomic). The handler callback happens outside the lock to avoid deadlock with Servlet container locks. This same volatile+synchronized pattern is used in `java.util.concurrent.FutureTask` and throughout the real Spring framework.

> **The Return Type Indirection.** `ConcurrentResultHandlerMethod` exists because on re-dispatch, the same URL maps to the same controller method. The Java method's declared return type is still `Callable<String>`, but the framework needs return value handlers to see `String.class` instead. By overriding `getReturnType()`, the wrapper changes what handlers see without changing the actual Java method. This is why all return value handlers must call `handlerMethod.getReturnType()` instead of `handlerMethod.getMethod().getReturnType()` -- the indirection is the feature.

> **The Two-Phase Lifecycle.** The same `DispatcherServlet.doDispatch()` code handles both the initial request and the re-dispatch. On the initial request: the handler returns, `WebAsyncManager` starts async processing, the dispatcher detects `isConcurrentHandlingStarted()`, calls `afterConcurrentHandlingStarted` on interceptors, and returns. On the re-dispatch: the handler is matched again (same URL), but `SimpleHandlerAdapter.handle()` detects `hasConcurrentResult()`, retrieves the saved result, wraps the handler, and processes through the normal return value handler pipeline. The dispatcher never sees `isConcurrentHandlingStarted()` on re-dispatch because the result was cleared. The same code path, two completely different behaviors.

---

## 27.12 What We Enhanced

| Aspect | Before (ch22 SSE-only) | After (ch27 Full Async) |
|---|---|---|
| **Async handling** | SSE only, no re-dispatch -- `SseEmitterReturnValueHandler` uses `AsyncContext` directly and completes in-place | `Callable`/`DeferredResult`/`CompletionStage` with full re-dispatch lifecycle through `WebAsyncManager` |
| **Interceptor lifecycle** | No async callback -- interceptors skipped entirely when `request.isAsyncStarted()` | `afterConcurrentHandlingStarted()` called on `AsyncHandlerInterceptor` implementations |
| **Return type checking** | `getMethod().getReturnType()` -- always returns the declared Java method return type | `getReturnType()` with override support via `ConcurrentResultHandlerMethod` |
| **Handler adapter** | Always invokes the controller method | Detects re-dispatch via `WebAsyncManager.hasConcurrentResult()`, retrieves saved result, wraps handler, processes through return value handlers |

---

## 27.13 Connection to Real Framework

| Simplified | Real Spring | Location |
|---|---|---|
| `DeferredResult` | `org.springframework.web.context.request.async.DeferredResult` | `DeferredResult.java:58-327` |
| `DeferredResult.setResultInternal()` | `DeferredResult.setResultInternal()` | `DeferredResult.java:239-256` |
| `DeferredResult.DeferredResultHandler` | `DeferredResult.DeferredResultHandler` | `DeferredResult.java:327` |
| `WebAsyncManager` | `org.springframework.web.context.request.async.WebAsyncManager` | `WebAsyncManager.java:86-340` |
| `WebAsyncManager.getAsyncManager()` | `WebAsyncUtils.getAsyncManager()` | `WebAsyncUtils.java:37` |
| `WebAsyncManager.startCallableProcessing()` | `WebAsyncManager.startCallableProcessing()` | `WebAsyncManager.java:186-242` |
| `WebAsyncManager.startDeferredResultProcessing()` | `WebAsyncManager.startDeferredResultProcessing()` | `WebAsyncManager.java:277-328` |
| `WebAsyncManager.setConcurrentResultAndDispatch()` | `WebAsyncManager.setConcurrentResultAndDispatch()` | `WebAsyncManager.java:340` |
| `AsyncHandlerInterceptor` | `org.springframework.web.servlet.AsyncHandlerInterceptor` | `AsyncHandlerInterceptor.java:49` |
| `CallableReturnValueHandler` | `CallableMethodReturnValueHandler` | `CallableMethodReturnValueHandler.java:47-55` |
| `DeferredResultReturnValueHandler` | `DeferredResultMethodReturnValueHandler` | `DeferredResultMethodReturnValueHandler.java:48-93` |
| `ConcurrentResultHandlerMethod` | `ServletInvocableHandlerMethod.ConcurrentResultHandlerMethod` | `ServletInvocableHandlerMethod.java:231` |
| `HandlerMethod.getReturnType()` | `HandlerMethod.getReturnType()` returns a `MethodParameter` | `HandlerMethod.java` |
| `HandlerExecutionChain.applyAfterConcurrentHandlingStarted()` | `HandlerExecutionChain.applyAfterConcurrentHandlingStarted()` | `HandlerExecutionChain.java:193` |

Key differences from the real framework:
- No `CallableProcessingInterceptor` / `DeferredResultProcessingInterceptor` chains
- No `WebAsyncTask` (custom executor per Callable)
- No `StandardServletAsyncWebRequest` wrapper -- uses `AsyncContext` directly
- Uses a shared cached thread pool instead of configurable `AsyncTaskExecutor`
- No `ModelAndViewContainer` context saving/restoring across async boundaries
- No `ListenableFuture` support (deprecated in Spring 6)

---

## 27.14 Complete Code

### DeferredResult.java [NEW]

`src/main/java/com/simplespringmvc/async/DeferredResult.java`

```java
package com.simplespringmvc.async;

import java.util.function.Consumer;
import java.util.function.Supplier;

/**
 * A container for producing an asynchronous result from any thread.
 *
 * Maps to: {@code org.springframework.web.context.request.async.DeferredResult}
 *
 * Unlike {@link java.util.concurrent.Callable} (where the framework submits work
 * to a thread pool), {@code DeferredResult} gives complete control over threading
 * to the application. The controller creates a DeferredResult, hands it off to
 * whatever async mechanism it likes (message listener, scheduled task, another
 * service), and returns immediately. When the result is ready, any thread calls
 * {@link #setResult(Object)} to complete the request.
 *
 * <h3>Thread coordination — the rendezvous:</h3>
 * Two threads race to complete the handshake:
 * <ul>
 *   <li><b>T1 (Servlet thread)</b> calls {@link #setResultHandler} to register
 *       the callback that dispatches the result back to the container</li>
 *   <li><b>T2 (Application thread)</b> calls {@link #setResult} when the result
 *       is ready</li>
 * </ul>
 * Whichever arrives second triggers the actual dispatch. This rendezvous uses
 * {@code volatile} fields for visibility and {@code synchronized} blocks for
 * atomicity. The critical call ({@code handler.handleResult}) happens OUTSIDE
 * the synchronized block to avoid deadlock with Servlet container locks.
 *
 * <h3>How the real framework does it (DeferredResult.java:108-166):</h3>
 * <pre>
 *   volatile Object result = RESULT_NONE;     // sentinel, not null
 *   volatile boolean expired;
 *   DeferredResultHandler resultHandler;       // set by WebAsyncManager
 *
 *   setResult(value):
 *     if already set or expired → return false
 *     synchronized(this):
 *       store result, grab handler reference
 *     if handler != null → handler.handleResult(result)   // outside lock!
 *
 *   setResultHandler(handler):
 *     synchronized(this):
 *       if result already set → grab it
 *       else → store handler
 *     if result was set → handler.handleResult(result)     // outside lock!
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No DeferredResultProcessingInterceptor chain</li>
 *   <li>No LifecycleInterceptor inner class — timeout/error handled directly</li>
 *   <li>No ModelAndViewContainer context saving</li>
 * </ul>
 *
 * @param <T> the result type
 */
public class DeferredResult<T> {

    /**
     * Sentinel object meaning "no result set yet".
     *
     * Maps to: {@code DeferredResult.RESULT_NONE} (line 58)
     * Using a sentinel instead of null because null is a valid result
     * (a controller might intentionally return null).
     */
    private static final Object RESULT_NONE = new Object();

    private final Long timeoutValue;
    private final Supplier<?> timeoutResult;

    /**
     * The actual result. Volatile for cross-thread visibility — T2 writes,
     * T1 (or T3 on re-dispatch) reads.
     *
     * Maps to: {@code DeferredResult.result} (line 84)
     */
    private volatile Object result = RESULT_NONE;

    /**
     * Set to true when the Servlet async lifecycle completes (timeout,
     * error, or normal completion). Once expired, no new results are accepted.
     *
     * Maps to: {@code DeferredResult.expired} (line 87)
     */
    private volatile boolean expired;

    /**
     * The callback that bridges this DeferredResult to the WebAsyncManager's
     * dispatch mechanism. Set by WebAsyncManager.startDeferredResultProcessing().
     *
     * Maps to: {@code DeferredResult.resultHandler} (line 81)
     */
    private DeferredResultHandler resultHandler;

    private Runnable timeoutCallback;
    private Consumer<Throwable> errorCallback;
    private Runnable completionCallback;

    /**
     * Create a DeferredResult with no timeout and no timeout fallback.
     */
    public DeferredResult() {
        this(null, null);
    }

    /**
     * Create a DeferredResult with a timeout but no fallback result.
     *
     * @param timeoutValue timeout in milliseconds, or null for no timeout
     */
    public DeferredResult(Long timeoutValue) {
        this(timeoutValue, null);
    }

    /**
     * Create a DeferredResult with a timeout and a fallback result supplier.
     *
     * Maps to: {@code DeferredResult(Long, Supplier)} (line 104)
     *
     * @param timeoutValue  timeout in milliseconds, or null for no timeout
     * @param timeoutResult supplier for a fallback result on timeout
     */
    public DeferredResult(Long timeoutValue, Supplier<?> timeoutResult) {
        this.timeoutValue = timeoutValue;
        this.timeoutResult = timeoutResult;
    }

    /**
     * Returns the configured timeout value.
     */
    public Long getTimeoutValue() {
        return timeoutValue;
    }

    // ─── Result setting ──────────────────────────────────────────────

    /**
     * Set the result value and trigger re-dispatch if the handler is ready.
     *
     * Maps to: {@code DeferredResult.setResult(T)} (line 203)
     *
     * Can be called from any thread. Returns true if the result was accepted,
     * false if the DeferredResult was already set or expired.
     *
     * @param result the result value
     * @return true if the result was accepted
     */
    public boolean setResult(T result) {
        return setResultInternal(result);
    }

    /**
     * Set an error result (typically a Throwable) and trigger re-dispatch.
     *
     * Maps to: {@code DeferredResult.setErrorResult(Object)} (line 230)
     *
     * On re-dispatch, the handler adapter will detect the Throwable and
     * rethrow it, routing it through the exception resolver chain.
     *
     * @param errorResult the error (usually a Throwable)
     * @return true if accepted
     */
    public boolean setErrorResult(Object errorResult) {
        return setResultInternal(errorResult);
    }

    /**
     * The shared implementation for setResult/setErrorResult.
     *
     * Maps to: {@code DeferredResult.setResultInternal(Object)} (line 239)
     *
     * Uses double-checked locking:
     * <ol>
     *   <li>Quick volatile read of isSetOrExpired() — fast path, no lock</li>
     *   <li>If not set: synchronized block to atomically store result and grab handler</li>
     *   <li>Call handler OUTSIDE the lock to avoid deadlock with Servlet locks</li>
     * </ol>
     */
    private boolean setResultInternal(Object result) {
        // Fast path: already set or expired — no synchronization needed
        if (isSetOrExpired()) {
            return false;
        }

        DeferredResultHandler handlerToUse;
        synchronized (this) {
            // Re-check under lock (double-checked locking)
            if (isSetOrExpired()) {
                return false;
            }
            this.result = result;
            handlerToUse = this.resultHandler;
        }

        // Call handler OUTSIDE synchronized block.
        // Maps to: DeferredResult.setResultInternal() line 256
        // The real framework does this to avoid holding the DeferredResult monitor
        // while the handler acquires Servlet container locks (potential deadlock).
        if (handlerToUse != null) {
            handlerToUse.handleResult(result);
        }
        return true;
    }

    /**
     * Register the callback that bridges to WebAsyncManager's dispatch.
     * Called by WebAsyncManager.startDeferredResultProcessing().
     *
     * Maps to: {@code DeferredResult.setResultHandler(DeferredResultHandler)} (line 190)
     *
     * If a result was already set (race: application set result before
     * WebAsyncManager registered the handler), immediately calls the handler.
     *
     * @param resultHandler the dispatch callback
     */
    public void setResultHandler(DeferredResultHandler resultHandler) {
        Object resultToHandle;
        synchronized (this) {
            if (this.result != RESULT_NONE) {
                // Result arrived first — grab it
                resultToHandle = this.result;
            } else {
                // No result yet — store handler for later
                this.resultHandler = resultHandler;
                return;
            }
        }
        // Call handler outside lock (same deadlock avoidance as setResultInternal)
        resultHandler.handleResult(resultToHandle);
    }

    // ─── State queries ──────────────────────────────────────────────

    /**
     * Check if the result is set or the DeferredResult has expired.
     *
     * Maps to: {@code DeferredResult.isSetOrExpired()} (line 162)
     */
    public boolean isSetOrExpired() {
        return (this.result != RESULT_NONE || this.expired);
    }

    /**
     * Check if a result has been set.
     */
    public boolean hasResult() {
        return this.result != RESULT_NONE;
    }

    /**
     * Get the result value, or null if no result has been set.
     */
    public Object getResult() {
        Object resultToCheck = this.result;
        return (resultToCheck != RESULT_NONE ? resultToCheck : null);
    }

    // ─── Lifecycle callbacks ─────────────────────────────────────────

    /**
     * Register a callback to run when the async request times out.
     *
     * Maps to: {@code DeferredResult.onTimeout(Runnable)} (line 140)
     *
     * @param callback the timeout callback
     */
    public void onTimeout(Runnable callback) {
        this.timeoutCallback = callback;
    }

    /**
     * Register a callback to run when an error occurs.
     *
     * @param callback the error callback
     */
    public void onError(Consumer<Throwable> callback) {
        this.errorCallback = callback;
    }

    /**
     * Register a callback to run when the async lifecycle completes
     * (normal, timeout, or error).
     *
     * @param callback the completion callback
     */
    public void onCompletion(Runnable callback) {
        this.completionCallback = callback;
    }

    // ─── Lifecycle hooks (called by WebAsyncManager) ─────────────────

    /**
     * Handle timeout: run user callback, then try to set the timeout result.
     *
     * Maps to: {@code DeferredResult.LifecycleInterceptor.handleTimeout()} (line 294)
     *
     * @return true if a timeout result was successfully set
     */
    boolean handleTimeout() {
        if (this.timeoutCallback != null) {
            this.timeoutCallback.run();
        }
        if (this.timeoutResult != null) {
            return setResultInternal(this.timeoutResult.get());
        }
        return false;
    }

    /**
     * Handle error: run user callback.
     *
     * Maps to: {@code DeferredResult.LifecycleInterceptor.handleError()} (line 306)
     */
    void handleError(Throwable ex) {
        if (this.errorCallback != null) {
            this.errorCallback.accept(ex);
        }
    }

    /**
     * Handle completion: mark as expired, run user callback.
     *
     * Maps to: {@code DeferredResult.LifecycleInterceptor.afterCompletion()} (line 315)
     */
    void handleCompletion() {
        this.expired = true;
        if (this.completionCallback != null) {
            this.completionCallback.run();
        }
    }

    // ─── Inner interface ──────────────────────────────────────────────

    /**
     * Callback interface bridging the DeferredResult to the dispatch mechanism.
     *
     * Maps to: {@code DeferredResult.DeferredResultHandler} (line 327)
     *
     * WebAsyncManager provides a lambda that calls setConcurrentResultAndDispatch().
     */
    @FunctionalInterface
    public interface DeferredResultHandler {
        void handleResult(Object result);
    }
}
```

### WebAsyncManager.java [NEW]

`src/main/java/com/simplespringmvc/async/WebAsyncManager.java`

```java
package com.simplespringmvc.async;

import jakarta.servlet.AsyncContext;
import jakarta.servlet.AsyncEvent;
import jakarta.servlet.AsyncListener;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Coordinates the async request processing lifecycle for Callable and
 * DeferredResult returns.
 *
 * Maps to: {@code org.springframework.web.context.request.async.WebAsyncManager}
 *
 * One instance per request, stored as a request attribute (lazily created).
 * Bridges between controller return values (Callable/DeferredResult) and
 * the Servlet container's async API.
 *
 * <h3>State machine (WebAsyncManager.java:92):</h3>
 * <pre>
 *   NOT_STARTED → ASYNC_PROCESSING → RESULT_SET → (re-dispatch clears back to NOT_STARTED)
 * </pre>
 * Transitions use {@link AtomicReference#compareAndSet} to ensure exactly
 * one thread can set the result.
 *
 * <h3>Threading model:</h3>
 * <pre>
 *   T1 (Servlet thread):  controller returns Callable/DeferredResult
 *                          → startAsync() on the Servlet request
 *                          → T1 is released back to the thread pool
 *
 *   T2 (App/Executor):    Callable completes or DeferredResult.setResult() called
 *                          → setConcurrentResultAndDispatch() saves result
 *                          → asyncContext.dispatch() re-enters the container
 *
 *   T3 (Servlet thread):  DispatcherServlet.doDispatch() runs again
 *                          → SimpleHandlerAdapter detects hasConcurrentResult()
 *                          → processes the saved result synchronously
 * </pre>
 *
 * <h3>How the real framework does it (WebAsyncManager.java:170-340):</h3>
 * <pre>
 *   startCallableProcessing():
 *     1. CAS NOT_STARTED → ASYNC_PROCESSING
 *     2. asyncWebRequest.startAsync()
 *     3. Register timeout/error/completion handlers
 *     4. Submit Callable to taskExecutor
 *     5. On completion → setConcurrentResultAndDispatch(result)
 *
 *   startDeferredResultProcessing():
 *     1. CAS NOT_STARTED → ASYNC_PROCESSING
 *     2. asyncWebRequest.startAsync()
 *     3. Register timeout/error/completion handlers
 *     4. deferredResult.setResultHandler(this::setConcurrentResultAndDispatch)
 *     5. When app calls setResult() → handler fires → setConcurrentResultAndDispatch
 *
 *   setConcurrentResultAndDispatch():
 *     synchronized:
 *       1. CAS ASYNC_PROCESSING → RESULT_SET
 *       2. Store result
 *       3. asyncWebRequest.dispatch()
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No CallableProcessingInterceptor / DeferredResultProcessingInterceptor chains</li>
 *   <li>No WebAsyncTask (custom executor per Callable)</li>
 *   <li>No StandardServletAsyncWebRequest wrapper — uses AsyncContext directly</li>
 *   <li>Uses a shared cached thread pool instead of configurable AsyncTaskExecutor</li>
 *   <li>No concurrent result context (ModelAndViewContainer) saving</li>
 * </ul>
 */
public class WebAsyncManager {

    /**
     * Sentinel meaning "no concurrent result has been set".
     *
     * Maps to: {@code WebAsyncManager.RESULT_NONE} (line 86)
     */
    private static final Object RESULT_NONE = new Object();

    /**
     * Request attribute key for the per-request WebAsyncManager instance.
     *
     * Maps to: {@code WebAsyncUtils.WEB_ASYNC_MANAGER_ATTRIBUTE}
     */
    public static final String WEB_ASYNC_MANAGER_ATTRIBUTE =
            WebAsyncManager.class.getName() + ".WEB_ASYNC_MANAGER";

    /**
     * The three-state machine controlling the async lifecycle.
     *
     * Maps to: {@code WebAsyncManager.state} (line 92)
     * The real version also uses an AtomicReference<State>.
     */
    private enum State { NOT_STARTED, ASYNC_PROCESSING, RESULT_SET }

    private final AtomicReference<State> state = new AtomicReference<>(State.NOT_STARTED);

    /**
     * The saved result from the async computation. Volatile for cross-thread
     * visibility: T2 writes, T3 reads.
     *
     * Maps to: {@code WebAsyncManager.concurrentResult} (line 95)
     */
    private volatile Object concurrentResult = RESULT_NONE;

    /**
     * The Servlet AsyncContext for this async request.
     */
    private AsyncContext asyncContext;

    /**
     * Shared thread pool for Callable execution. Daemon threads so they
     * don't prevent JVM shutdown.
     *
     * Maps to: {@code WebAsyncManager.taskExecutor} (line 100)
     * The real version defaults to SimpleAsyncTaskExecutor and is configurable.
     */
    private static final ExecutorService taskExecutor = Executors.newCachedThreadPool(r -> {
        Thread t = new Thread(r, "simple-mvc-async");
        t.setDaemon(true);
        return t;
    });

    // ─── Static factory ──────────────────────────────────────────────

    /**
     * Get or create the WebAsyncManager for the current request.
     *
     * Maps to: {@code WebAsyncUtils.getAsyncManager(ServletRequest)} (line 37)
     * which stores the manager as a request attribute for per-request singleton
     * semantics.
     *
     * @param request the current HTTP request
     * @return the WebAsyncManager for this request
     */
    public static WebAsyncManager getAsyncManager(HttpServletRequest request) {
        WebAsyncManager manager =
                (WebAsyncManager) request.getAttribute(WEB_ASYNC_MANAGER_ATTRIBUTE);
        if (manager == null) {
            manager = new WebAsyncManager();
            request.setAttribute(WEB_ASYNC_MANAGER_ATTRIBUTE, manager);
        }
        return manager;
    }

    // ─── Callable processing ─────────────────────────────────────────

    /**
     * Start async processing for a Callable return value.
     *
     * Maps to: {@code WebAsyncManager.startCallableProcessing()} (line 186)
     *
     * <ol>
     *   <li>CAS transition NOT_STARTED → ASYNC_PROCESSING</li>
     *   <li>Start Servlet async mode</li>
     *   <li>Register AsyncListener for timeout/error/completion</li>
     *   <li>Submit the Callable to the thread pool</li>
     *   <li>On completion (success or failure), call setConcurrentResultAndDispatch()</li>
     * </ol>
     *
     * @param callable the callable to execute asynchronously
     * @param request  the current HTTP request
     * @param response the current HTTP response
     */
    public void startCallableProcessing(Callable<?> callable,
                                         HttpServletRequest request,
                                         HttpServletResponse response) {
        if (!state.compareAndSet(State.NOT_STARTED, State.ASYNC_PROCESSING)) {
            throw new IllegalStateException(
                    "Async processing already started for this request");
        }

        // Start Servlet async mode.
        // Maps to: StandardServletAsyncWebRequest.startAsync() (line 75)
        asyncContext = request.startAsync(request, response);
        asyncContext.setTimeout(0); // no timeout by default

        // Register lifecycle listener.
        // Maps to: WebAsyncManager registers timeout/error/completion handlers
        // on StandardServletAsyncWebRequest (line 207-226).
        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) {
                setConcurrentResultAndDispatch(
                        new RuntimeException("Async Callable processing timed out"));
            }

            @Override
            public void onError(AsyncEvent event) {
                Throwable cause = event.getThrowable();
                if (cause == null) {
                    cause = new RuntimeException("Async processing error");
                }
                setConcurrentResultAndDispatch(cause);
            }

            @Override
            public void onComplete(AsyncEvent event) {
                // Async lifecycle complete — nothing to do
            }

            @Override
            public void onStartAsync(AsyncEvent event) {
                // Re-dispatch starts a new async context — not used
            }
        });

        // Submit the Callable to the thread pool.
        // Maps to: WebAsyncManager line 231-242 where the callable is submitted
        // to the taskExecutor.
        taskExecutor.submit(() -> {
            try {
                Object result = callable.call();
                setConcurrentResultAndDispatch(result);
            } catch (Throwable ex) {
                setConcurrentResultAndDispatch(ex);
            }
        });
    }

    // ─── DeferredResult processing ───────────────────────────────────

    /**
     * Start async processing for a DeferredResult return value.
     *
     * Maps to: {@code WebAsyncManager.startDeferredResultProcessing()} (line 277)
     *
     * <ol>
     *   <li>CAS transition NOT_STARTED → ASYNC_PROCESSING</li>
     *   <li>Start Servlet async mode</li>
     *   <li>Configure timeout from DeferredResult</li>
     *   <li>Register AsyncListener for timeout/error/completion that delegates
     *       to DeferredResult's lifecycle hooks</li>
     *   <li>Wire the DeferredResult's resultHandler to call
     *       setConcurrentResultAndDispatch()</li>
     * </ol>
     *
     * @param deferredResult the deferred result container
     * @param request        the current HTTP request
     * @param response       the current HTTP response
     */
    public void startDeferredResultProcessing(DeferredResult<?> deferredResult,
                                               HttpServletRequest request,
                                               HttpServletResponse response) {
        if (!state.compareAndSet(State.NOT_STARTED, State.ASYNC_PROCESSING)) {
            throw new IllegalStateException(
                    "Async processing already started for this request");
        }

        asyncContext = request.startAsync(request, response);

        // Apply timeout from the DeferredResult configuration
        if (deferredResult.getTimeoutValue() != null) {
            asyncContext.setTimeout(deferredResult.getTimeoutValue());
        } else {
            asyncContext.setTimeout(0); // no timeout
        }

        // Register lifecycle listener that delegates to DeferredResult hooks.
        // Maps to: WebAsyncManager line 302-326
        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) {
                // Let DeferredResult handle timeout (run callback, try timeout result)
                if (!deferredResult.handleTimeout()) {
                    // No timeout result configured — set an error result
                    setConcurrentResultAndDispatch(
                            new RuntimeException("DeferredResult processing timed out"));
                }
            }

            @Override
            public void onError(AsyncEvent event) {
                Throwable cause = event.getThrowable();
                if (cause == null) {
                    cause = new RuntimeException("Async processing error");
                }
                deferredResult.handleError(cause);
                setConcurrentResultAndDispatch(cause);
            }

            @Override
            public void onComplete(AsyncEvent event) {
                deferredResult.handleCompletion();
            }

            @Override
            public void onStartAsync(AsyncEvent event) {
                // not used
            }
        });

        // Wire the DeferredResult to this manager's dispatch mechanism.
        // Maps to: WebAsyncManager line 328:
        //   deferredResult.setResultHandler(result -> {
        //       result = interceptorChain.applyPostProcess(request, response, result);
        //       setConcurrentResultAndDispatch(result);
        //   });
        //
        // When the application thread calls deferredResult.setResult(value),
        // the handler fires, which saves the result and dispatches.
        deferredResult.setResultHandler(this::setConcurrentResultAndDispatch);
    }

    // ─── Result coordination ─────────────────────────────────────────

    /**
     * Save the async result and re-dispatch the request to the container.
     *
     * Maps to: {@code WebAsyncManager.setConcurrentResultAndDispatch()} (line 340)
     *
     * The synchronized block + CAS ensures only one thread can set the result:
     * <ul>
     *   <li>CAS ASYNC_PROCESSING → RESULT_SET (fails if already set)</li>
     *   <li>Store the result</li>
     *   <li>Call asyncContext.dispatch() to re-enter the Servlet container</li>
     * </ul>
     *
     * @param result the async computation result (or Throwable on failure)
     */
    private synchronized void setConcurrentResultAndDispatch(Object result) {
        if (!state.compareAndSet(State.ASYNC_PROCESSING, State.RESULT_SET)) {
            return; // already set, or not in ASYNC_PROCESSING state
        }
        this.concurrentResult = result;

        // Re-dispatch the request. The Servlet container will call
        // DispatcherServlet.service() again with DispatcherType.ASYNC.
        // Maps to: StandardServletAsyncWebRequest.dispatch() (line 91)
        asyncContext.dispatch();
    }

    // ─── State queries (used by SimpleHandlerAdapter on re-dispatch) ──

    /**
     * Whether async processing has been started (and possibly completed).
     *
     * Maps to: {@code WebAsyncManager.isConcurrentHandlingStarted()} (line 146)
     * Returns true from the moment startAsync is called until the result
     * is cleared on re-dispatch.
     */
    public boolean isConcurrentHandlingStarted() {
        return state.get() != State.NOT_STARTED;
    }

    /**
     * Whether a concurrent result has been saved and is ready for processing.
     *
     * Maps to: {@code WebAsyncManager.hasConcurrentResult()} (line 154)
     */
    public boolean hasConcurrentResult() {
        return concurrentResult != RESULT_NONE;
    }

    /**
     * Get the saved concurrent result.
     *
     * Maps to: {@code WebAsyncManager.getConcurrentResult()} (line 162)
     */
    public Object getConcurrentResult() {
        return concurrentResult;
    }

    /**
     * Clear the concurrent result and reset the state machine for the next
     * request lifecycle.
     *
     * Maps to: {@code WebAsyncManager.clearConcurrentResult()} (line 170)
     * Called by SimpleHandlerAdapter after retrieving the result on re-dispatch.
     */
    public void clearConcurrentResult() {
        concurrentResult = RESULT_NONE;
        state.set(State.NOT_STARTED);
    }
}
```

### AsyncHandlerInterceptor.java [NEW]

`src/main/java/com/simplespringmvc/interceptor/AsyncHandlerInterceptor.java`

```java
package com.simplespringmvc.interceptor;

import com.simplespringmvc.mapping.HandlerMethod;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Extension of {@link HandlerInterceptor} with a callback for when a handler
 * starts concurrent (async) processing.
 *
 * Maps to: {@code org.springframework.web.servlet.AsyncHandlerInterceptor}
 *
 * <h3>Lifecycle with async handlers:</h3>
 * <pre>
 *   Initial request (T1):
 *     preHandle() → handler returns Callable/DeferredResult
 *     → afterConcurrentHandlingStarted() (NOT postHandle/afterCompletion)
 *     → T1 released
 *
 *   Re-dispatch (T3, after async completes):
 *     preHandle() → handler adapter processes saved result
 *     → postHandle() → afterCompletion()
 * </pre>
 *
 * This allows interceptors to:
 * <ul>
 *   <li>Clean up thread-locals on T1 (the initial Servlet thread)</li>
 *   <li>Re-establish them on T3 (the re-dispatch thread)</li>
 *   <li>Distinguish initial vs re-dispatch via {@code request.getDispatcherType()}
 *       (REQUEST vs ASYNC)</li>
 * </ul>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>Uses typed {@code HandlerMethod} instead of {@code Object handler}</li>
 * </ul>
 */
public interface AsyncHandlerInterceptor extends HandlerInterceptor {

    /**
     * Called instead of {@code postHandle} and {@code afterCompletion} when
     * the handler starts concurrent processing.
     *
     * Maps to: {@code AsyncHandlerInterceptor.afterConcurrentHandlingStarted()}
     * (line 49)
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     * @param handler  the handler that started async processing
     * @throws Exception in case of errors
     */
    default void afterConcurrentHandlingStarted(HttpServletRequest request,
                                                 HttpServletResponse response,
                                                 HandlerMethod handler) throws Exception {
    }
}
```

### CallableReturnValueHandler.java [NEW]

`src/main/java/com/simplespringmvc/adapter/CallableReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.async.WebAsyncManager;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.concurrent.Callable;

/**
 * Handles return values of type {@link Callable} by starting async processing
 * and submitting the Callable to a thread pool.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.CallableMethodReturnValueHandler}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>Controller method returns a {@code Callable<T>}</li>
 *   <li>This handler detects the Callable return type via {@link #supportsReturnType}</li>
 *   <li>Gets the per-request {@link WebAsyncManager}</li>
 *   <li>Calls {@code startCallableProcessing()} which:
 *       <ul>
 *         <li>Starts Servlet async mode (request.startAsync())</li>
 *         <li>Submits the Callable to a thread pool</li>
 *       </ul>
 *   </li>
 *   <li>Returns null — the Servlet thread (T1) is released</li>
 *   <li>When the Callable completes (on T2), WebAsyncManager saves the result
 *       and calls asyncContext.dispatch() to re-enter the container</li>
 *   <li>On re-dispatch (T3), SimpleHandlerAdapter detects the saved result
 *       (via WebAsyncManager.hasConcurrentResult()) and processes it through
 *       the normal return value handler pipeline using a
 *       ConcurrentResultHandlerMethod wrapper</li>
 * </ol>
 *
 * <h3>Why Callable doesn't match on re-dispatch:</h3>
 * On re-dispatch, the same handler method is matched (same URL). Its declared
 * return type is still {@code Callable<T>}. However, SimpleHandlerAdapter
 * wraps it in a {@link HandlerMethod#wrapConcurrentResult(Object)} that changes
 * {@code getReturnType()} to the actual result type. Since this handler checks
 * {@code handlerMethod.getReturnType()} (not {@code getMethod().getReturnType()}),
 * it won't match on re-dispatch.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No WebAsyncTask support (custom executor per Callable)</li>
 *   <li>No ModelAndViewContainer context saving/restoring</li>
 * </ul>
 */
public class CallableReturnValueHandler implements HandlerMethodReturnValueHandler {

    /**
     * Supports methods whose return type is {@link Callable}.
     *
     * Maps to: {@code CallableMethodReturnValueHandler.supportsReturnType()} (line 47)
     *
     * Uses {@code handlerMethod.getReturnType()} which is overridden by
     * ConcurrentResultHandlerMethod on re-dispatch to return the actual result type.
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return Callable.class.isAssignableFrom(handlerMethod.getReturnType());
    }

    /**
     * Start async processing for the Callable return value.
     *
     * Maps to: {@code CallableMethodReturnValueHandler.handleReturnValue()} (line 55)
     *
     * @return null because the response will be completed asynchronously via re-dispatch
     */
    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        if (returnValue == null) {
            return null;
        }

        Callable<?> callable = (Callable<?>) returnValue;
        WebAsyncManager asyncManager = WebAsyncManager.getAsyncManager(request);
        asyncManager.startCallableProcessing(callable, request, response);

        return null; // response completed via async re-dispatch
    }
}
```

### DeferredResultReturnValueHandler.java [NEW]

`src/main/java/com/simplespringmvc/adapter/DeferredResultReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.async.DeferredResult;
import com.simplespringmvc.async.WebAsyncManager;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.concurrent.CompletionException;
import java.util.concurrent.CompletionStage;

/**
 * Handles return values of type {@link DeferredResult} and {@link CompletionStage}
 * by starting async processing and registering a result handler.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.DeferredResultMethodReturnValueHandler}
 *
 * <h3>DeferredResult vs Callable:</h3>
 * <ul>
 *   <li><b>Callable:</b> The framework controls the thread — submits to a pool</li>
 *   <li><b>DeferredResult:</b> The application controls the thread — any thread
 *       can call setResult() at any time</li>
 * </ul>
 *
 * <h3>CompletionStage support:</h3>
 * {@link CompletionStage} (and its common implementation {@link java.util.concurrent.CompletableFuture})
 * is transparently supported by wrapping it in a DeferredResult. The CompletionStage's
 * {@code whenComplete} callback bridges to {@code setResult()}/{@code setErrorResult()}.
 *
 * Maps to: {@code DeferredResultMethodReturnValueHandler.adaptCompletionStage()} (line 93)
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No ListenableFuture support (deprecated in Spring 6)</li>
 *   <li>No ModelAndViewContainer context saving</li>
 * </ul>
 */
public class DeferredResultReturnValueHandler implements HandlerMethodReturnValueHandler {

    /**
     * Supports methods whose return type is DeferredResult or CompletionStage.
     *
     * Maps to: {@code DeferredResultMethodReturnValueHandler.supportsReturnType()} (line 48)
     *
     * Uses {@code handlerMethod.getReturnType()} which is overridden by
     * ConcurrentResultHandlerMethod on re-dispatch.
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        Class<?> type = handlerMethod.getReturnType();
        return DeferredResult.class.isAssignableFrom(type)
                || CompletionStage.class.isAssignableFrom(type);
    }

    /**
     * Start async processing for the DeferredResult or CompletionStage.
     *
     * Maps to: {@code DeferredResultMethodReturnValueHandler.handleReturnValue()} (line 60)
     *
     * @return null because the response will be completed asynchronously via re-dispatch
     */
    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        if (returnValue == null) {
            return null;
        }

        DeferredResult<?> deferredResult;
        if (returnValue instanceof DeferredResult<?> dr) {
            deferredResult = dr;
        } else if (returnValue instanceof CompletionStage<?> cs) {
            deferredResult = adaptCompletionStage(cs);
        } else {
            throw new IllegalStateException(
                    "Unexpected return value type: " + returnValue.getClass());
        }

        WebAsyncManager asyncManager = WebAsyncManager.getAsyncManager(request);
        asyncManager.startDeferredResultProcessing(deferredResult, request, response);

        return null; // response completed via async re-dispatch
    }

    /**
     * Adapt a CompletionStage to a DeferredResult by wiring the
     * whenComplete callback.
     *
     * Maps to: {@code DeferredResultMethodReturnValueHandler.adaptCompletionStage()} (line 93)
     *
     * <h3>How it works:</h3>
     * <pre>
     *   CompletableFuture completes
     *     → whenComplete((value, ex) → ...)
     *       → if success: deferredResult.setResult(value)
     *       → if error:   deferredResult.setErrorResult(unwrappedCause)
     *         → DeferredResult.setResultHandler fires
     *           → WebAsyncManager.setConcurrentResultAndDispatch()
     *             → asyncContext.dispatch()
     * </pre>
     *
     * CompletionException is unwrapped to get the actual cause, matching
     * how the real framework handles it.
     */
    private DeferredResult<Object> adaptCompletionStage(CompletionStage<?> completionStage) {
        DeferredResult<Object> result = new DeferredResult<>();
        completionStage.whenComplete((value, ex) -> {
            if (ex != null) {
                // Unwrap CompletionException to get the actual cause
                if (ex instanceof CompletionException && ex.getCause() != null) {
                    ex = ex.getCause();
                }
                result.setErrorResult(ex);
            } else {
                result.setResult(value);
            }
        });
        return result;
    }
}
```

### HandlerMethod.java [MODIFIED]

`src/main/java/com/simplespringmvc/mapping/HandlerMethod.java`

```java
package com.simplespringmvc.mapping;

import java.lang.reflect.Method;

/**
 * Wraps a controller bean instance and a specific method on that bean,
 * representing a single endpoint that can handle HTTP requests.
 *
 * Maps to: {@code org.springframework.web.method.HandlerMethod}
 *
 * The real HandlerMethod is much richer: it supports lazy bean resolution
 * (by bean name through BeanFactory), method parameter introspection via
 * MethodParameter[], return type analysis, @ResponseStatus evaluation,
 * and validation flags. It extends AnnotatedMethod which itself wraps
 * a SynthesizingMethodParameter.
 *
 * Simplifications:
 * <ul>
 *   <li>Always holds the bean instance directly (no lazy resolution by name)</li>
 *   <li>No @ResponseStatus evaluation</li>
 *   <li>No validation flag support</li>
 * </ul>
 */
public class HandlerMethod {

    private final Object bean;
    private final Method method;
    private final MethodParameter[] parameters;

    public HandlerMethod(Object bean, Method method) {
        if (bean == null) {
            throw new IllegalArgumentException("Bean must not be null");
        }
        if (method == null) {
            throw new IllegalArgumentException("Method must not be null");
        }
        this.bean = bean;
        this.method = method;
        this.parameters = initMethodParameters();
    }

    /**
     * Build the MethodParameter[] array eagerly at construction time.
     *
     * Maps to: {@code HandlerMethod.initMethodParameters()} (line 189)
     * Real version creates SynthesizingMethodParameter instances.
     */
    private MethodParameter[] initMethodParameters() {
        int count = method.getParameterCount();
        MethodParameter[] result = new MethodParameter[count];
        for (int i = 0; i < count; i++) {
            result[i] = new MethodParameter(method, i);
        }
        return result;
    }

    /**
     * The controller bean instance that owns this method.
     */
    public Object getBean() {
        return bean;
    }

    /**
     * The method to invoke on the bean.
     */
    public Method getMethod() {
        return method;
    }

    /**
     * The type of the controller bean (convenience for annotation checks).
     */
    public Class<?> getBeanType() {
        return bean.getClass();
    }

    /**
     * The method parameters, wrapped as MethodParameter objects.
     *
     * Maps to: {@code HandlerMethod.getMethodParameters()} (line 247)
     * Real version returns MethodParameter[] with generic type info,
     * annotation synthesis, and nesting support.
     */
    public MethodParameter[] getMethodParameters() {
        return parameters;
    }

    /**
     * Returns the return type of the handler method.
     *
     * ch27: This method exists to support {@link ConcurrentResultHandlerMethod},
     * which overrides it to return the actual result type instead of
     * Callable/DeferredResult. All return value handlers should use this
     * method instead of {@code getMethod().getReturnType()}.
     *
     * Maps to: {@code HandlerMethod.getReturnType()} which returns a
     * MethodParameter with index -1 that provides the return type.
     *
     * @return the return type class
     */
    public Class<?> getReturnType() {
        return method.getReturnType();
    }

    /**
     * Create a wrapper HandlerMethod that changes the apparent return type
     * to match the actual async result. Used on re-dispatch so that return
     * value handlers see the unwrapped result type instead of Callable/DeferredResult.
     *
     * Maps to: {@code ServletInvocableHandlerMethod.wrapConcurrentResult()} and
     * the inner {@code ConcurrentResultHandlerMethod} class (line 231).
     *
     * <h3>Why this is needed:</h3>
     * On re-dispatch, the same handler method is matched (same URL). Its declared
     * return type is still {@code Callable<String>}. Without this wrapper, the
     * CallableReturnValueHandler would match again, creating an infinite loop.
     * The wrapper changes getReturnType() to return {@code String.class}, so
     * ResponseBody/ViewName handlers match instead.
     *
     * @param result the concurrent result (determines the new return type)
     * @return a wrapper HandlerMethod with the result's type as return type
     */
    public HandlerMethod wrapConcurrentResult(Object result) {
        return new ConcurrentResultHandlerMethod(this, result);
    }

    /**
     * A HandlerMethod wrapper that overrides the return type to match the
     * actual concurrent result, rather than the declared Callable/DeferredResult type.
     *
     * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation
     *          .ServletInvocableHandlerMethod.ConcurrentResultHandlerMethod}
     *
     * This is why the real Spring has the indirection of getReturnType() —
     * on re-dispatch, the handler method is the same (same URL, same controller),
     * but the "return type" needs to be the unwrapped result type so that
     * return value handlers (ResponseBody, ViewName, etc.) match correctly
     * instead of Callable/DeferredResult handlers matching again.
     */
    private static class ConcurrentResultHandlerMethod extends HandlerMethod {

        private final Class<?> resultType;

        ConcurrentResultHandlerMethod(HandlerMethod original, Object result) {
            super(original.getBean(), original.getMethod());
            this.resultType = (result != null) ? result.getClass() : Void.class;
        }

        @Override
        public Class<?> getReturnType() {
            return resultType;
        }
    }

    @Override
    public String toString() {
        return bean.getClass().getSimpleName() + "#" + method.getName();
    }
}
```

### HandlerExecutionChain.java [MODIFIED]

`src/main/java/com/simplespringmvc/interceptor/HandlerExecutionChain.java`

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
 * <h3>ch27 Enhancement — Async lifecycle:</h3>
 * Added {@link #applyAfterConcurrentHandlingStarted} for async handler support.
 * When the handler starts concurrent processing (Callable/DeferredResult), this
 * method is called instead of postHandle/afterCompletion. It notifies
 * {@link AsyncHandlerInterceptor}s so they can clean up thread-locals on the
 * initial Servlet thread before it's released.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No constructor unwrapping of nested HandlerExecutionChains</li>
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
     * Add an interceptor at position 0 — before all other interceptors.
     *
     * <h3>ch19 Enhancement:</h3>
     * Used to insert the {@code CorsInterceptor} at the front of the chain
     * so CORS validation happens before any other interceptors.
     *
     * Maps to: The real framework inserts the CorsInterceptor at index 0
     * in {@code AbstractHandlerMapping.getCorsHandlerExecutionChain()} (line 754).
     *
     * @param interceptor the interceptor to add at position 0
     */
    public void addInterceptorFirst(HandlerInterceptor interceptor) {
        this.interceptorList.add(0, interceptor);
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
     * Called instead of {@code applyPostHandle} and {@code triggerAfterCompletion}
     * when the handler starts concurrent (async) processing.
     *
     * ch27: Iterates interceptors in reverse order (from interceptorIndex down)
     * and calls {@link AsyncHandlerInterceptor#afterConcurrentHandlingStarted}
     * on each one that implements the async interface. Non-async interceptors
     * are silently skipped.
     *
     * Maps to: {@code HandlerExecutionChain.applyAfterConcurrentHandlingStarted()} (line 193)
     *
     * @param request  current HTTP request
     * @param response current HTTP response
     */
    public void applyAfterConcurrentHandlingStarted(HttpServletRequest request,
                                                      HttpServletResponse response) {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            if (interceptor instanceof AsyncHandlerInterceptor asyncInterceptor) {
                try {
                    asyncInterceptor.afterConcurrentHandlingStarted(
                            request, response, this.handler);
                } catch (Throwable ex) {
                    System.err.println(
                            "AsyncHandlerInterceptor.afterConcurrentHandlingStarted threw: " + ex);
                }
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

### SimpleHandlerAdapter.java [MODIFIED]

`src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.async.WebAsyncManager;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.convert.SimpleConversionService;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.converter.JacksonXmlMessageConverter;
import com.simplespringmvc.converter.Jaxb2XmlMessageConverter;
import com.simplespringmvc.converter.PlainTextMessageConverter;
import com.simplespringmvc.http.ContentNegotiationManager;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.session.DefaultSessionAttributeStore;
import com.simplespringmvc.session.SessionAttributeStore;
import com.simplespringmvc.session.SessionAttributesHandler;
import com.simplespringmvc.session.SimpleSessionStatus;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * The adapter that bridges the DispatcherServlet's generic handler invocation
 * to our specific HandlerMethod-based invocation pipeline.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 *
 * <h3>ch27 Enhancement:</h3>
 * <ul>
 *   <li>Detects re-dispatch after async processing (Callable/DeferredResult) and
 *       processes the saved concurrent result through the normal return value
 *       handler pipeline instead of re-invoking the controller</li>
 *   <li>Uses {@link HandlerMethod#wrapConcurrentResult(Object)} to change the
 *       apparent return type so async-specific return value handlers don't match again</li>
 *   <li>Registers {@link CallableReturnValueHandler} and
 *       {@link DeferredResultReturnValueHandler} for async return types</li>
 *   <li>If the async result is a Throwable, rethrows it so the exception resolver
 *       pipeline handles it normally</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Single adapter instance (not a chain of adapters)</li>
 *   <li>No ModelAndViewContainer — return value handlers return ModelAndView directly</li>
 *   <li>No ModelFactory — session attribute lifecycle is managed directly</li>
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

    /**
     * ch18: Cache of SessionAttributesHandler per controller class.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.sessionAttributesHandlerCache} (line 194)
     * The real version uses a ConcurrentHashMap keyed by controller type.
     */
    private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache =
            new ConcurrentHashMap<>(64);

    /**
     * ch18: Strategy for session attribute storage.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.sessionAttributeStore}
     */
    private final SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();

    public SimpleHandlerAdapter() {
        // ch14: Create the ContentNegotiationManager for Accept header parsing
        contentNegotiationManager = new ContentNegotiationManager();

        // ch14: Register converters in priority order.
        // ORDER MATTERS: JacksonMessageConverter first means JSON is the default
        // when Accept is */* (most common case). PlainTextMessageConverter handles
        // text/plain requests for String return values.
        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());
        // ch26: XML converters — JAXB first (for @XmlRootElement classes), then
        // Jackson XML (for any POJO). JAXB is more specific (annotation-gated),
        // so it should be checked first when both could handle a class.
        messageConverters.add(new Jaxb2XmlMessageConverter());
        messageConverters.add(new JacksonXmlMessageConverter());

        conversionService = new SimpleConversionService();

        // ch20: Servlet API + Locale resolver — handles HttpServletRequest, HttpServletResponse,
        // HttpSession, and Locale as method parameters. Comes FIRST because it matches by type,
        // not annotation, and should be checked before annotation-based resolvers.
        argumentResolvers.addResolver(new ServletRequestArgumentResolver());

        // ch18: Session-aware resolvers come FIRST — they match by annotation/type
        // and should take priority over generic resolvers.
        argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
        argumentResolvers.addResolver(new SessionStatusArgumentResolver());
        argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        // ch25: MultipartFileArgumentResolver BEFORE RequestParamArgumentResolver.
        // Both match @RequestParam, but MultipartFileArgumentResolver only matches
        // MultipartFile type. It must come first so that @RequestParam MultipartFile
        // parameters are resolved by the multipart resolver, not the query param resolver.
        argumentResolvers.addResolver(new MultipartFileArgumentResolver());
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers — they take priority
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

        // ch27: Async return value handlers — BEFORE SSE and ResponseBody.
        // These detect Callable/DeferredResult/CompletionStage and start async
        // processing. On re-dispatch, ConcurrentResultHandlerMethod changes
        // getReturnType() so these handlers won't match again.
        returnValueHandlers.addHandler(new CallableReturnValueHandler());
        returnValueHandlers.addHandler(new DeferredResultReturnValueHandler());

        // ch22: SseEmitterReturnValueHandler BEFORE ResponseBodyReturnValueHandler.
        // This is critical: @RestController has class-level @ResponseBody, so
        // ResponseBodyReturnValueHandler would match ALL methods on such classes.
        // By registering SseEmitterReturnValueHandler first, methods returning
        // SseEmitter are correctly handled by the SSE handler.
        // Maps to: RequestMappingHandlerAdapter.getDefaultReturnValueHandlers() (line 681)
        // where ResponseBodyEmitterReturnValueHandler comes before
        // RequestResponseBodyMethodProcessor.
        returnValueHandlers.addHandler(new SseEmitterReturnValueHandler());
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
        // ch26: XML converters
        messageConverters.add(new Jaxb2XmlMessageConverter());
        messageConverters.add(new JacksonXmlMessageConverter());

        conversionService = new SimpleConversionService();

        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        // ch20: Servlet API + Locale resolver
        argumentResolvers.addResolver(new ServletRequestArgumentResolver());
        // ch18: Session-aware resolvers
        argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
        argumentResolvers.addResolver(new SessionStatusArgumentResolver());
        argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        // ch25: MultipartFileArgumentResolver before RequestParamArgumentResolver
        argumentResolvers.addResolver(new MultipartFileArgumentResolver());
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

        // ch27: Async return value handlers
        returnValueHandlers.addHandler(new CallableReturnValueHandler());
        returnValueHandlers.addHandler(new DeferredResultReturnValueHandler());

        // ch22: SseEmitterReturnValueHandler first
        returnValueHandlers.addHandler(new SseEmitterReturnValueHandler());
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

    /**
     * Handle the request by invoking the handler method, managing session
     * attributes lifecycle around the invocation.
     *
     * <h3>ch27 Enhancement:</h3>
     * The handle() method now checks for re-dispatch after async processing
     * BEFORE invoking the handler. If a concurrent result is available:
     * <pre>
     *   1. Get concurrent result from WebAsyncManager
     *   2. Clear the result (reset state machine)
     *   3. If result is Throwable → rethrow (exception resolvers handle it)
     *   4. Wrap the handler in ConcurrentResultHandlerMethod (changes return type)
     *   5. Process through return value handlers as normal
     * </pre>
     *
     * This mirrors {@code RequestMappingHandlerAdapter.invokeHandlerMethod()} (line 885)
     * which checks {@code asyncManager.hasConcurrentResult()} and creates a
     * {@code ConcurrentResultHandlerMethod} wrapper.
     *
     * <h3>ch18 Enhancement:</h3>
     * The normal (non-re-dispatch) path wraps invocation with session attribute management:
     * <pre>
     *   1. Get/create SessionAttributesHandler for this controller
     *   2. Retrieve session attributes → store as request attribute
     *   3. Create SessionStatus → store as request attribute
     *   4. Resolve arguments (resolvers can find session attrs via request attribute)
     *   5. Invoke handler method
     *   6. Process return value → ModelAndView
     *   7. If SessionStatus.isComplete() → cleanup session attributes
     *      Else → store matching model attributes in session
     *   8. Return ModelAndView
     * </pre>
     */
    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                               Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // ch27: Check if this is a re-dispatch after async processing.
        // On re-dispatch, the Servlet container re-enters doDispatch() with
        // DispatcherType.ASYNC. The same handler is matched (same URL), but
        // instead of invoking the controller again, we process the saved result.
        //
        // Maps to: RequestMappingHandlerAdapter.invokeHandlerMethod() (line 888-901)
        WebAsyncManager asyncManager = WebAsyncManager.getAsyncManager(request);
        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            asyncManager.clearConcurrentResult();

            // If the async computation failed, rethrow the exception.
            // The exception resolver chain in doDispatch() will handle it.
            if (result instanceof Throwable t) {
                if (t instanceof Exception ex) {
                    throw ex;
                }
                throw new RuntimeException(t);
            }

            // Wrap the handler method so getReturnType() returns the actual
            // result type instead of Callable/DeferredResult. This prevents
            // CallableReturnValueHandler/DeferredResultReturnValueHandler from
            // matching again, and lets ResponseBody/ViewName handlers match.
            HandlerMethod wrappedMethod = handlerMethod.wrapConcurrentResult(result);
            return returnValueHandlers.handleReturnValue(
                    result, wrappedMethod, request, response);
        }

        // ch18: Step 1 — Get or create SessionAttributesHandler for this controller
        SessionAttributesHandler sessionAttrHandler =
                getSessionAttributesHandler(handlerMethod);

        // ch18: Step 2 — Retrieve session attributes and store as request attribute
        // so ModelAttributeArgumentResolver can find existing instances
        Map<String, Object> sessionAttrs = sessionAttrHandler.retrieveAttributes(request);
        request.setAttribute(ModelAttributeArgumentResolver.SESSION_ATTRIBUTES_KEY, sessionAttrs);

        // ch18: Step 3 — Create per-request SessionStatus
        SimpleSessionStatus sessionStatus = new SimpleSessionStatus();
        request.setAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE, sessionStatus);

        // ch20: Store response as request attribute for ServletRequestArgumentResolver
        request.setAttribute(ServletRequestArgumentResolver.RESPONSE_ATTRIBUTE, response);

        // Steps 4-6: Resolve arguments, invoke, process return value
        Object[] args = resolveArguments(handlerMethod, request);
        Object result = invokeHandlerMethod(handlerMethod, args);
        ModelAndView mv = returnValueHandlers.handleReturnValue(
                result, handlerMethod, request, response);

        // ch18: Step 7 — Session attributes lifecycle
        if (sessionAttrHandler.hasSessionAttributes()) {
            if (sessionStatus.isComplete()) {
                // SessionStatus.setComplete() was called — cleanup
                sessionAttrHandler.cleanupAttributes(request);
            } else {
                // Store matching model attributes back in session.
                // Two sources of attributes to store:
                // 1. Session attributes retrieved at start (may have been modified by reference)
                sessionAttrHandler.storeAttributes(request, sessionAttrs);
                // 2. New attributes in the ModelAndView model
                if (mv != null) {
                    sessionAttrHandler.storeAttributes(request, mv.getModel());
                }
            }
        }

        return mv;
    }

    /**
     * Get or create the SessionAttributesHandler for the given handler's controller class.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.getSessionAttributesHandler()} (line 976)
     *
     * Cached per controller type so @SessionAttributes metadata is only
     * extracted once.
     */
    private SessionAttributesHandler getSessionAttributesHandler(HandlerMethod handlerMethod) {
        Class<?> handlerType = handlerMethod.getBeanType();
        return this.sessionAttributesHandlerCache.computeIfAbsent(
                handlerType,
                type -> new SessionAttributesHandler(type, this.sessionAttributeStore));
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

    /**
     * Returns the session attributes handler cache (for testing/inspection).
     *
     * <h3>ch18 Enhancement:</h3>
     * Added for testing session attribute management.
     */
    public Map<Class<?>, SessionAttributesHandler> getSessionAttributesHandlerCache() {
        return sessionAttributesHandlerCache;
    }
}
```

### SimpleDispatcherServlet.java [MODIFIED]

`src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.RedirectAttributesArgumentResolver;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.async.WebAsyncManager;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.cors.CorsConfiguration;
import com.simplespringmvc.cors.CorsInterceptor;
import com.simplespringmvc.cors.CorsProcessor;
import com.simplespringmvc.cors.CorsRegistry;
import com.simplespringmvc.cors.CorsUtils;
import com.simplespringmvc.cors.DefaultCorsProcessor;
import com.simplespringmvc.exception.ExceptionHandlerExceptionResolver;
import com.simplespringmvc.exception.HandlerExceptionResolver;
import com.simplespringmvc.flash.FlashMap;
import com.simplespringmvc.flash.FlashMapManager;
import com.simplespringmvc.flash.SimpleRedirectAttributes;
import com.simplespringmvc.http.HttpMediaTypeNotAcceptableException;
import com.simplespringmvc.interceptor.HandlerExecutionChain;
import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.locale.AcceptHeaderLocaleResolver;
import com.simplespringmvc.locale.LocaleContextHolder;
import com.simplespringmvc.locale.LocaleResolver;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.multipart.MultipartHttpServletRequest;
import com.simplespringmvc.multipart.MultipartResolver;
import com.simplespringmvc.view.ModelAndView;
import com.simplespringmvc.view.View;
import com.simplespringmvc.view.ViewResolver;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Locale;
import java.util.Map;

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
 *   <li>ch11: interceptor pre/post processing</li>
 *   <li>ch12: exception handler resolution</li>
 *   <li>ch13: view resolution and rendering ✓</li>
 *   <li>ch18: flash map lifecycle and redirect handling ✓</li>
 *   <li>ch19: CORS processing in getHandler() ✓</li>
 *   <li>ch20: locale resolution lifecycle ✓</li>
 *   <li>ch22: async (SSE) skip — bypass postHandle/afterCompletion when async started ✓</li>
 *   <li>ch25: multipart resolution — wrap request before handler lookup, cleanup after ✓</li>
 *   <li>ch27: async lifecycle — WebAsyncManager check, afterConcurrentHandlingStarted ✓</li>
 * </ul>
 *
 * <h3>ch18 Enhancement:</h3>
 * The dispatch pipeline now includes flash map management and redirect support:
 * <pre>
 *   0. Flash map lifecycle: retrieve input flash map, create output flash map
 *   1. getHandler(request) → HandlerExecutionChain
 *   2. chain.applyPreHandle()
 *   3. ha.handle() → returns ModelAndView (null if response handled directly)
 *   4. chain.applyPostHandle()
 *   5. chain.triggerAfterCompletion() (always)
 *   6. If exception → processHandlerException()
 *   7. If ModelAndView non-null:
 *      7a. If view name starts with "redirect:" → handle redirect + save flash map
 *      7b. Otherwise → resolve and render view (merge input flash attrs into model)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No theme resolution</li>
 *   <li>Falls back to text/plain when view can't be resolved (real framework throws)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    /**
     * ch20: Request attribute under which the LocaleResolver is stored.
     * Used by {@code LocaleChangeInterceptor} to find the resolver.
     *
     * Maps to: {@code DispatcherServlet.LOCALE_RESOLVER_ATTRIBUTE} (line 178)
     */
    public static final String LOCALE_RESOLVER_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".LOCALE_RESOLVER";

    /**
     * Request attribute under which the input FlashMap is stored.
     *
     * Maps to: {@code DispatcherServlet.INPUT_FLASH_MAP_ATTRIBUTE} (line 221)
     */
    public static final String INPUT_FLASH_MAP_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".INPUT_FLASH_MAP";

    /**
     * Request attribute under which the output FlashMap is stored.
     *
     * Maps to: {@code DispatcherServlet.OUTPUT_FLASH_MAP_ATTRIBUTE} (line 228)
     */
    public static final String OUTPUT_FLASH_MAP_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".OUTPUT_FLASH_MAP";

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    /**
     * Global interceptors applied to all handlers in order.
     *
     * Maps to: {@code AbstractHandlerMapping.adaptedInterceptors} (line 123)
     */
    private final List<HandlerInterceptor> interceptors = new ArrayList<>();

    /**
     * Exception resolvers consulted when a handler throws an exception.
     *
     * Maps to: {@code DispatcherServlet.handlerExceptionResolvers} (line 191)
     */
    private final List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<>();

    /**
     * ViewResolvers that translate logical view names into View objects.
     *
     * Maps to: {@code DispatcherServlet.viewResolvers} (line 197)
     * Real version is populated from ViewResolver beans in the context
     * (or from DispatcherServlet.properties defaults). Iterated in order —
     * first resolver that returns a non-null View wins.
     *
     * <h3>ch13 Enhancement:</h3>
     * Added to support view resolution and rendering.
     */
    private final List<ViewResolver> viewResolvers = new ArrayList<>();

    /**
     * ch18: FlashMapManager for managing flash attributes across redirects.
     *
     * Maps to: {@code DispatcherServlet.flashMapManager} (line 200)
     * The real version is initialized from a FlashMapManager bean in the context,
     * defaulting to SessionFlashMapManager.
     */
    private FlashMapManager flashMapManager;

    /**
     * ch19: CORS processor for validating and writing CORS response headers.
     * Defaults to {@link DefaultCorsProcessor}.
     *
     * Maps to: {@code AbstractHandlerMapping.corsProcessor} (line 99)
     */
    private CorsProcessor corsProcessor = new DefaultCorsProcessor();

    /**
     * ch20: Locale resolver for determining the current locale.
     * Defaults to {@link AcceptHeaderLocaleResolver}.
     *
     * Maps to: {@code DispatcherServlet.localeResolver} (line 178)
     * The real version is initialized from a "localeResolver" bean in the context,
     * defaulting to AcceptHeaderLocaleResolver.
     */
    private LocaleResolver localeResolver = new AcceptHeaderLocaleResolver();

    /**
     * ch25: MultipartResolver for handling file uploads.
     * Null by default — multipart support is opt-in.
     *
     * Maps to: {@code DispatcherServlet.multipartResolver} (line 175)
     * The real version is initialized from a "multipartResolver" bean in the
     * ApplicationContext. If no bean is found, multipart support is disabled.
     * Spring Boot auto-configures a StandardServletMultipartResolver.
     */
    private MultipartResolver multipartResolver;

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Called by Tomcat once the Servlet is registered. Initializes strategy
     * components (handler mappings, adapters, etc.) from the bean container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
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
        initExceptionResolvers();
        // ch13: ViewResolvers are added programmatically via addViewResolver()
        // rather than initialized from the container. The real framework finds
        // ViewResolver beans in the ApplicationContext.
        // ch18: FlashMapManager is set programmatically via setFlashMapManager()
    }

    private void initHandlerMapping() {
        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(beanContainer);
    }

    private void initHandlerAdapter() {
        handlerAdapter = new SimpleHandlerAdapter();
    }

    private void initExceptionResolvers() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(beanContainer);
        exceptionResolvers.add(resolver);
    }

    /**
     * Override service() to route ALL HTTP methods through doDispatch(),
     * with locale lifecycle management wrapping the dispatch.
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870) and
     * {@code FrameworkServlet.processRequest()} (line 982)
     *
     * <h3>ch20 Enhancement:</h3>
     * The locale lifecycle mirrors {@code FrameworkServlet.processRequest()}:
     * <pre>
     *   1. Expose locale resolver as request attribute (for interceptors)
     *   2. Resolve locale from request → set in LocaleContextHolder
     *   3. doDispatch() — locale available to all code via LocaleContextHolder
     *   4. finally: reset LocaleContextHolder (cleanup thread-local)
     * </pre>
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            // ch20: Locale lifecycle — expose resolver and set thread-local locale
            if (localeResolver != null) {
                request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, localeResolver);
                Locale locale = localeResolver.resolveLocale(request);
                LocaleContextHolder.setLocale(locale);
            }

            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        } finally {
            // ch20: Reset locale context to prevent thread-local leaks.
            // Maps to: FrameworkServlet.resetContextHolders() (line 1084)
            LocaleContextHolder.resetLocale();
        }
    }

    /**
     * The central dispatch method — the heart of the framework.
     *
     * Maps to: {@code DispatcherServlet.doDispatch()} (line 935)
     *
     * <h3>ch18 Enhancement:</h3>
     * The dispatch flow now includes flash map management:
     * <pre>
     *   0. Flash map lifecycle:
     *      - Retrieve input flash map (from previous redirect)
     *      - Create output flash map (for possible redirect in this request)
     *   1. mappedHandler = getHandler(request)
     *   2. mappedHandler.applyPreHandle()
     *   3. mv = ha.handle(request, response, handler) ← now manages session attrs
     *   4. mappedHandler.applyPostHandle()
     *   5. mappedHandler.triggerAfterCompletion() (always)
     *   6. processHandlerException() if exception
     *   7. If view name starts with "redirect:" → handleRedirect() + save flash map
     *      Otherwise → render view (merge input flash attrs into model)
     * </pre>
     *
     * The flash map lifecycle mirrors the real DispatcherServlet.doService()
     * (lines 850-857) which sets up INPUT/OUTPUT flash maps as request attributes
     * before doDispatch() is called.
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerExecutionChain mappedHandler = null;
        ModelAndView mv = null;
        Exception dispatchException = null;
        // ch25: Track whether we wrapped the request for multipart cleanup
        boolean multipartRequestParsed = false;
        HttpServletRequest processedRequest = request;

        // ch25: Step 0a — Check for multipart request and wrap if needed.
        // This must happen BEFORE handler lookup so argument resolvers
        // see the MultipartHttpServletRequest wrapper.
        //
        // Maps to: DispatcherServlet.checkMultipart() (line 1105)
        if (multipartResolver != null && multipartResolver.isMultipart(request)) {
            processedRequest = multipartResolver.resolveMultipart(request);
            multipartRequestParsed = (processedRequest != request);
        }

        // ch18: Step 0b — Flash map lifecycle: retrieve input, create output
        // ch25: Use processedRequest for flash maps (the original or multipart-wrapped)
        FlashMap inputFlashMap = null;
        if (flashMapManager != null) {
            inputFlashMap = flashMapManager.retrieveAndUpdate(processedRequest, response);
            if (inputFlashMap != null) {
                processedRequest.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE,
                        Collections.unmodifiableMap(inputFlashMap));
            }
            processedRequest.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        }

        try {
            // Step 1: Look up the handler + interceptor chain for this request
            // ch25: Use processedRequest — the wrapped request preserves
            // getRequestURI() and getMethod() via HttpServletRequestWrapper.
            mappedHandler = getHandler(processedRequest);

            if (mappedHandler == null) {
                response.sendError(HttpServletResponse.SC_NOT_FOUND,
                        "No handler found for " + processedRequest.getMethod()
                                + " " + processedRequest.getRequestURI());
                return;
            }

            // Step 2: Apply interceptor preHandle — forward order
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Step 3: Find a HandlerAdapter and invoke the handler.
            // ch18: handle() now manages session attributes lifecycle internally.
            // ch25: Pass processedRequest so argument resolvers see the
            // MultipartHttpServletRequest wrapper and can extract files.
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
            mv = adapter.handle(processedRequest, response, mappedHandler.getHandler());

            // ch27: Check if WebAsyncManager started async processing
            // (Callable/DeferredResult). If so, call afterConcurrentHandlingStarted
            // on interceptors instead of postHandle/afterCompletion.
            //
            // Maps to: DispatcherServlet.doDispatch() (line 965):
            //   if (asyncManager.isConcurrentHandlingStarted()) {
            //       mappedHandler.applyAfterConcurrentHandlingStarted(req, res);
            //       return;
            //   }
            WebAsyncManager asyncManager =
                    WebAsyncManager.getAsyncManager(processedRequest);
            if (asyncManager.isConcurrentHandlingStarted()) {
                if (mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(
                            processedRequest, response);
                }
                return;
            }

            // ch22: If async processing started by other means (SSE emitter),
            // skip postHandle, afterCompletion, and view rendering.
            // SSE uses AsyncContext directly without WebAsyncManager.
            if (processedRequest.isAsyncStarted()) {
                return;
            }

            // Step 4: Apply interceptor postHandle — reverse order
            mappedHandler.applyPostHandle(processedRequest, response);

        } catch (HttpMediaTypeNotAcceptableException ex) {
            // ch14: The client's Accept header doesn't match any type the server can produce.
            response.sendError(HttpServletResponse.SC_NOT_ACCEPTABLE, ex.getMessage());
            if (mappedHandler != null) {
                mappedHandler.triggerAfterCompletion(processedRequest, response, null);
            }
            return;
        } catch (Exception ex) {
            dispatchException = ex;
        }

        // Step 5: afterCompletion — always runs
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(processedRequest, response, dispatchException);
        }

        // Step 6: Exception handling
        if (dispatchException != null) {
            Object handler = (mappedHandler != null) ? mappedHandler.getHandler() : null;
            if (!processHandlerException(processedRequest, response, handler, dispatchException)) {
                throw dispatchException;
            }
            // Exception was handled — don't render a view
            return;
        }

        // Step 7 (ch13/ch18): View resolution, rendering, or redirect.
        if (mv != null && mv.getViewName() != null) {
            String viewName = mv.getViewName();

            // ch18: Handle "redirect:" prefix
            if (viewName.startsWith("redirect:")) {
                String redirectUrl = viewName.substring("redirect:".length());
                handleRedirect(redirectUrl, processedRequest, response);
            } else {
                // ch18: Merge input flash attributes into model for view access
                if (inputFlashMap != null && !inputFlashMap.isEmpty()) {
                    mv.getModel().putAll(inputFlashMap);
                }
                render(mv, processedRequest, response);
            }
        }
    }

    /**
     * Handle a redirect by saving flash attributes and sending the redirect response.
     *
     * <h3>ch18 Enhancement:</h3>
     * Maps to the redirect handling in {@code DispatcherServlet.processDispatchResult()}
     * and {@code RequestContextUtils.saveOutputFlashMap()} (line 218).
     *
     * The flow:
     * <ol>
     *   <li>Get the output FlashMap (created at request start)</li>
     *   <li>Copy flash attributes from RedirectAttributes (if a handler used it)</li>
     *   <li>Set the target request path on the FlashMap</li>
     *   <li>Save the FlashMap via FlashMapManager</li>
     *   <li>Send the HTTP redirect response</li>
     * </ol>
     *
     * @param redirectUrl the target URL to redirect to
     * @param request     current HTTP request
     * @param response    current HTTP response
     */
    private void handleRedirect(String redirectUrl, HttpServletRequest request,
                                 HttpServletResponse response) throws IOException {
        if (flashMapManager != null) {
            FlashMap outputFlashMap =
                    (FlashMap) request.getAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE);

            // Copy flash attributes from RedirectAttributes if the handler used it
            SimpleRedirectAttributes redirectAttrs = (SimpleRedirectAttributes)
                    request.getAttribute(RedirectAttributesArgumentResolver.REDIRECT_ATTRIBUTES_ATTRIBUTE);
            if (redirectAttrs != null && !redirectAttrs.getFlashAttributes().isEmpty()) {
                outputFlashMap.putAll(redirectAttrs.getFlashAttributes());
            }

            // Set target path so the FlashMap matches the redirect target
            outputFlashMap.setTargetRequestPath(redirectUrl);

            // Save to session for the next request to retrieve
            flashMapManager.saveOutputFlashMap(outputFlashMap, request, response);
        }

        response.sendRedirect(redirectUrl);
    }

    /**
     * Resolve the view name and render the view with model data.
     *
     * Maps to: {@code DispatcherServlet.render()} (line 1267)
     *
     * The real render() method:
     * <pre>
     *   1. Determine locale (via LocaleResolver)
     *   2. Resolve view name → View object (via ViewResolver chain)
     *   3. Set response status if configured
     *   4. Call view.render(model, request, response)
     * </pre>
     *
     * <h3>ch20 Enhancement:</h3>
     * Now resolves the locale from the LocaleResolver (step 1), sets it on
     * the response (so response headers reflect the locale), and passes it
     * to the ViewResolver chain for locale-specific template selection.
     *
     * @param mv       the ModelAndView holding view name and model data
     * @param request  current HTTP request
     * @param response current HTTP response
     */
    private void render(ModelAndView mv, HttpServletRequest request,
                        HttpServletResponse response) throws Exception {
        String viewName = mv.getViewName();

        // ch20: Step 1 — Determine locale from LocaleContextHolder
        // (set at request start in service(), possibly changed by LocaleChangeInterceptor)
        Locale locale = LocaleContextHolder.getLocale();
        response.setLocale(locale);

        // Step 2: Resolve view name via ViewResolver chain (now with locale)
        View view = resolveViewName(viewName, locale);

        if (view != null) {
            // Step 4: Delegate to the View object for rendering.
            view.render(mv.getModel(), request, response);
        } else {
            // Fallback: write the view name directly as text/plain.
            response.setContentType("text/plain");
            response.setCharacterEncoding("UTF-8");
            response.getWriter().write(viewName);
            response.getWriter().flush();
        }
    }

    /**
     * Resolve a view name to a View object by iterating the ViewResolver chain.
     *
     * Maps to: {@code DispatcherServlet.resolveViewName()} (line 1339)
     * → iterates {@code viewResolvers} and returns the first non-null result.
     *
     * <h3>ch20 Enhancement:</h3>
     * Now passes the locale to each ViewResolver for locale-specific
     * template selection (e.g., hello_fr.html for French).
     *
     * @param viewName the logical view name to resolve
     * @param locale   the locale for locale-specific view selection
     * @return the View object, or null if no resolver can resolve it
     */
    private View resolveViewName(String viewName, Locale locale) throws Exception {
        for (ViewResolver resolver : viewResolvers) {
            View view = resolver.resolveViewName(viewName, locale);
            if (view != null) {
                return view;
            }
        }
        return null;
    }

    /**
     * Iterate exception resolvers to handle the given exception.
     *
     * Maps to: {@code DispatcherServlet.processHandlerException()} (line 1207)
     */
    private boolean processHandlerException(HttpServletRequest request, HttpServletResponse response,
                                            Object handler, Exception ex) {
        for (HandlerExceptionResolver resolver : exceptionResolvers) {
            if (resolver.resolveException(request, response, handler, ex)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Look up the handler for this request and build the execution chain.
     *
     * <h3>ch19 Enhancement:</h3>
     * After resolving the handler and building the initial chain, checks for
     * CORS configuration. If this is a CORS request (or preflight), a
     * {@link CorsInterceptor} is inserted at position 0 in the chain.
     *
     * For preflight requests where no handler matches (because there's no
     * OPTIONS mapping), we create a no-op handler and rely on the CORS
     * interceptor to write the response headers.
     *
     * Maps to: {@code AbstractHandlerMapping.getHandler()} (line 542)
     * — specifically the CORS block at lines 569-588.
     */
    private HandlerExecutionChain getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        HandlerMethod handler = handlerMapping.lookupHandler(request);

        // ch19: Handle preflight requests that have no explicit OPTIONS handler
        if (handler == null && CorsUtils.isPreFlightRequest(request)) {
            // Look up the handler for the *actual* method in the preflight
            // Access-Control-Request-Method header to get its CORS config
            HandlerMethod actualHandler = findPreflightHandler(request);
            if (actualHandler != null) {
                // Create a no-op handler for the preflight — the CorsInterceptor
                // will write the CORS headers, and this handler does nothing.
                HandlerMethod noOpHandler = createNoOpHandler();
                HandlerExecutionChain chain = new HandlerExecutionChain(noOpHandler, interceptors);
                applyCorsProcessing(chain, actualHandler, request);
                return chain;
            }
            // No CORS config found — fall through to return null (404)
            return null;
        }

        if (handler == null) {
            return null;
        }

        HandlerExecutionChain chain = new HandlerExecutionChain(handler, interceptors);

        // ch19: Apply CORS processing for actual CORS requests
        if (CorsUtils.isCorsRequest(request)
                || handlerMapping.hasCorsConfiguration(handler)) {
            applyCorsProcessing(chain, handler, request);
        }

        return chain;
    }

    /**
     * For a preflight request, find the handler that would handle the
     * actual request method (from Access-Control-Request-Method header).
     *
     * Maps to: The real framework does this implicitly because handler
     * mappings register CORS configs keyed by HandlerMethod, and the
     * handler is found by the requested method, not OPTIONS.
     */
    private HandlerMethod findPreflightHandler(HttpServletRequest request) {
        String actualMethod = request.getHeader("Access-Control-Request-Method");
        if (actualMethod == null) {
            return null;
        }
        return handlerMapping.lookupHandler(request.getRequestURI(), actualMethod);
    }

    /**
     * Create a no-op handler for preflight requests. The handler method
     * does nothing — all CORS work is done by the CorsInterceptor.
     *
     * Maps to: {@code AbstractHandlerMapping.PreFlightHttpRequestHandler} (line 795)
     */
    private HandlerMethod createNoOpHandler() {
        try {
            // Use a method that exists on Object — toString() as a no-op target
            return new HandlerMethod(this, getClass().getMethod("noOpPreflightHandler"));
        } catch (NoSuchMethodException e) {
            throw new IllegalStateException("Cannot create no-op preflight handler", e);
        }
    }

    /**
     * No-op method used as the handler for preflight requests.
     * Must be public so reflection can invoke it.
     */
    public String noOpPreflightHandler() {
        return null; // intentionally does nothing
    }

    /**
     * Merge handler-level and global CORS configs, then insert a CorsInterceptor
     * at position 0 in the execution chain.
     *
     * Maps to: {@code AbstractHandlerMapping.getCorsHandlerExecutionChain()} (line 746)
     *
     * The merging strategy: global config is the base, handler config is overlaid:
     * {@code globalConfig.combine(handlerConfig)}
     */
    private void applyCorsProcessing(HandlerExecutionChain chain,
                                      HandlerMethod handler,
                                      HttpServletRequest request) {
        // Get handler-level config (from @CrossOrigin)
        CorsConfiguration handlerConfig = handlerMapping.getCorsConfiguration(handler);

        // Get global config (from CorsRegistry)
        CorsConfiguration globalConfig = handlerMapping.getGlobalCorsConfiguration(
                request.getRequestURI());

        // Merge: global is base, handler overlays
        CorsConfiguration mergedConfig;
        if (globalConfig != null && handlerConfig != null) {
            mergedConfig = globalConfig.combine(handlerConfig);
        } else if (globalConfig != null) {
            mergedConfig = globalConfig;
        } else if (handlerConfig != null) {
            mergedConfig = handlerConfig;
        } else {
            return; // no CORS config at all
        }

        // Validate
        mergedConfig.validateAllowCredentials();

        // Insert CorsInterceptor at position 0
        chain.addInterceptorFirst(new CorsInterceptor(mergedConfig, corsProcessor));
    }

    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
    }

    // ─── FlashMapManager registration ────────────────────────────────

    /**
     * Set the FlashMapManager for managing flash attributes across redirects.
     *
     * Maps to: In the real framework, the FlashMapManager is initialized from
     * a FlashMapManager bean in the ApplicationContext (defaulting to
     * SessionFlashMapManager). We use programmatic registration for simplicity.
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support flash attributes and the Post/Redirect/Get pattern.
     *
     * @param flashMapManager the manager to use
     */
    public void setFlashMapManager(FlashMapManager flashMapManager) {
        this.flashMapManager = flashMapManager;
    }

    /**
     * Returns the configured FlashMapManager (for testing/inspection).
     */
    public FlashMapManager getFlashMapManager() {
        return flashMapManager;
    }

    // ─── ViewResolver registration ──────────────────────────────────

    /**
     * Register a ViewResolver to resolve view names to View objects.
     *
     * Maps to: In the real framework, ViewResolvers are registered as beans in
     * the ApplicationContext and auto-detected by DispatcherServlet.initViewResolvers().
     * We use programmatic registration for simplicity.
     *
     * <h3>ch13 Enhancement:</h3>
     * Added to support view resolution.
     *
     * @param viewResolver the resolver to add
     */
    public void addViewResolver(ViewResolver viewResolver) {
        this.viewResolvers.add(viewResolver);
    }

    /**
     * Returns the registered ViewResolvers (for testing/inspection).
     */
    public List<ViewResolver> getViewResolvers() {
        return List.copyOf(viewResolvers);
    }

    // ─── Interceptor registration ────────────────────────────────────

    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptors.add(interceptor);
    }

    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptors);
    }

    // ─── CORS registration (ch19) ─────────────────────────────────

    /**
     * Apply global CORS configuration from a {@link CorsRegistry}.
     *
     * Maps to: In the real framework, this is done via
     * {@code WebMvcConfigurer.addCorsMappings(CorsRegistry)}, which eventually
     * calls {@code AbstractHandlerMapping.setCorsConfigurations()}.
     *
     * @param corsRegistry the registry with global CORS mappings
     */
    public void setCorsRegistry(CorsRegistry corsRegistry) {
        if (handlerMapping != null) {
            handlerMapping.setCorsConfigurations(corsRegistry.getCorsConfigurations());
        }
    }

    /**
     * Set a custom CORS processor. Defaults to {@link DefaultCorsProcessor}.
     *
     * Maps to: {@code AbstractHandlerMapping.setCorsProcessor(CorsProcessor)} (line 237)
     *
     * @param corsProcessor the processor to use
     */
    public void setCorsProcessor(CorsProcessor corsProcessor) {
        this.corsProcessor = corsProcessor;
    }

    /**
     * Returns the configured CorsProcessor (for testing/inspection).
     */
    public CorsProcessor getCorsProcessor() {
        return corsProcessor;
    }

    // ─── Locale resolver registration (ch20) ──────────────────────────

    /**
     * Set the LocaleResolver for this dispatcher.
     *
     * Maps to: In the real framework, the LocaleResolver is initialized from
     * a "localeResolver" bean in the ApplicationContext (defaulting to
     * AcceptHeaderLocaleResolver). We use programmatic registration.
     *
     * @param localeResolver the resolver to use
     */
    public void setLocaleResolver(LocaleResolver localeResolver) {
        this.localeResolver = localeResolver;
    }

    /**
     * Returns the configured LocaleResolver.
     */
    public LocaleResolver getLocaleResolver() {
        return localeResolver;
    }

    // ─── Multipart resolver registration (ch25) ────────────────────

    /**
     * Set the MultipartResolver for handling file uploads.
     *
     * Maps to: In the real framework, the MultipartResolver is initialized from
     * a "multipartResolver" bean in the ApplicationContext. Spring Boot
     * auto-configures a StandardServletMultipartResolver.
     *
     * <h3>ch25 Enhancement:</h3>
     * Added to support multipart file upload handling. When set, the
     * DispatcherServlet wraps multipart requests before handler lookup.
     *
     * @param multipartResolver the resolver to use (null to disable)
     */
    public void setMultipartResolver(MultipartResolver multipartResolver) {
        this.multipartResolver = multipartResolver;
    }

    /**
     * Returns the configured MultipartResolver (for testing/inspection).
     */
    public MultipartResolver getMultipartResolver() {
        return multipartResolver;
    }

    // ─── Accessors ───────────────────────────────────────────────────

    public BeanContainer getBeanContainer() {
        return beanContainer;
    }

    public SimpleHandlerMapping getHandlerMapping() {
        return handlerMapping;
    }

    public HandlerAdapter getHandlerAdapter() {
        return handlerAdapter;
    }

    public List<HandlerExceptionResolver> getExceptionResolvers() {
        return List.copyOf(exceptionResolvers);
    }
}
```

### SseEmitterReturnValueHandler.java [MODIFIED]

`src/main/java/com/simplespringmvc/adapter/SseEmitterReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.http.MediaType;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.sse.SseEmitter;
import com.simplespringmvc.view.ModelAndView;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.AsyncContext;
import jakarta.servlet.AsyncEvent;
import jakarta.servlet.AsyncListener;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.io.PrintWriter;
import java.util.ArrayList;
import java.util.List;
import java.util.Collection;
import java.util.function.Consumer;

/**
 * Handles {@link SseEmitter} return values by starting Servlet async processing
 * and initializing the emitter with a handler that writes SSE events to the
 * HTTP response.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ResponseBodyEmitterReturnValueHandler}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   1. Detect SseEmitter return type
 *   2. Call emitter.extendResponse() → sets text/event-stream
 *   3. Disable ETag caching (ShallowEtagHeaderFilter)
 *   4. Wrap response in StreamingServletServerHttpResponse (absorbs header changes)
 *   5. Create DeferredResult with emitter's timeout
 *   6. Start async via WebAsyncManager.startDeferredResultProcessing()
 *   7. Create DefaultSseEmitterHandler (uses message converters for writing)
 *   8. Call emitter.initialize(handler)
 * </pre>
 *
 * We simplify by using Servlet {@link AsyncContext} directly (no WebAsyncManager
 * or DeferredResult layer) and using Jackson ObjectMapper directly for JSON
 * serialization (no message converter iteration with StreamingServletServerHttpResponse).
 *
 * <h3>How the Handler writes SSE events:</h3>
 * The emitter's {@link SseEmitter.SseEventBuilder} produces a {@code Set<DataWithMediaType>}
 * where protocol text ("id:", "event:", "data:") is plain String and data payloads
 * are objects to serialize. The handler writes Strings directly and serializes objects
 * to JSON using Jackson.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No WebAsyncManager / DeferredResult — uses AsyncContext directly</li>
 *   <li>No StreamingServletServerHttpResponse wrapper</li>
 *   <li>No message converter iteration — uses Jackson directly for non-String data</li>
 *   <li>No reactive type support</li>
 *   <li>No ResponseEntity{@literal <SseEmitter>} unwrapping</li>
 *   <li>No SSE fragment/view rendering</li>
 * </ul>
 */
public class SseEmitterReturnValueHandler implements HandlerMethodReturnValueHandler {

    /**
     * Check whether the handler method returns an {@link SseEmitter}.
     *
     * Maps to: {@code ResponseBodyEmitterReturnValueHandler.supportsReturnType()}
     * which checks if the return type is {@code ResponseBodyEmitter} or subclass,
     * or if it's wrapped in {@code ResponseEntity}.
     */
    /**
     * ch27: Uses handlerMethod.getReturnType() instead of getMethod().getReturnType()
     * so that ConcurrentResultHandlerMethod (on async re-dispatch) can override the
     * apparent return type to the actual result type.
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return SseEmitter.class.isAssignableFrom(handlerMethod.getReturnType());
    }

    /**
     * Start async processing and initialize the emitter.
     *
     * Maps to: {@code ResponseBodyEmitterReturnValueHandler.handleReturnValue()} (line 181)
     *
     * The flow:
     * <ol>
     *   <li>Set SSE response headers (Content-Type, Cache-Control)</li>
     *   <li>Start Servlet async mode via {@code request.startAsync()}</li>
     *   <li>Configure timeout on the AsyncContext</li>
     *   <li>Create the {@link SseHandler} that bridges emitter sends to the response</li>
     *   <li>Register an {@link AsyncListener} for timeout/error/completion lifecycle</li>
     *   <li>Initialize the emitter (flushes early sends, wires callbacks)</li>
     * </ol>
     *
     * Returns null because the response is handled asynchronously — there is
     * no ModelAndView to render.
     */
    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        if (returnValue == null) {
            return null;
        }

        SseEmitter emitter = (SseEmitter) returnValue;

        // Set SSE response headers.
        // Maps to: SseEmitter.extendResponse() (line 69) which sets text/event-stream.
        response.setContentType("text/event-stream");
        response.setCharacterEncoding("UTF-8");
        // Disable caching — SSE streams must not be cached
        response.setHeader("Cache-Control", "no-cache");
        // Keep-alive signals the client to maintain the connection
        response.setHeader("Connection", "keep-alive");

        // Start Servlet async mode.
        // Maps to: WebAsyncManager.startDeferredResultProcessing() → asyncWebRequest.startAsync()
        // We call the Servlet API directly, skipping the WebAsyncManager/DeferredResult layer.
        AsyncContext asyncContext;
        try {
            asyncContext = request.startAsync(request, response);
        } catch (IllegalStateException ex) {
            // Async not supported — servlet registration needs setAsyncSupported(true)
            emitter.initializeWithError(ex);
            throw ex;
        }

        // Configure timeout
        if (emitter.getTimeout() != null) {
            asyncContext.setTimeout(emitter.getTimeout());
        } else {
            asyncContext.setTimeout(0); // no timeout by default
        }

        // Create the handler that bridges emitter sends to the HTTP response
        SseHandler sseHandler = new SseHandler(response, asyncContext);

        // Register async lifecycle listeners.
        // Maps to: WebAsyncManager registers timeout/error/completion handlers on
        // the StandardServletAsyncWebRequest, which adds an AsyncListener.
        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) {
                // Timeout occurred — notify the handler's callbacks then complete
                sseHandler.fireTimeoutCallbacks();
                emitter.completeWithError(
                        new IOException("SSE connection timed out"));
            }

            @Override
            public void onError(AsyncEvent event) {
                Throwable cause = event.getThrowable();
                if (cause == null) {
                    cause = new IOException("Async processing error");
                }
                sseHandler.fireErrorCallbacks(cause);
                emitter.completeWithError(cause);
            }

            @Override
            public void onComplete(AsyncEvent event) {
                sseHandler.fireCompletionCallbacks();
            }

            @Override
            public void onStartAsync(AsyncEvent event) {
                // not used
            }
        });

        // Initialize the emitter — flushes any early sends and wires callbacks.
        // Maps to: ResponseBodyEmitterReturnValueHandler line 229:
        //   emitter.initialize(emitterHandler)
        try {
            emitter.initialize(sseHandler);
        } catch (Throwable ex) {
            emitter.initializeWithError(ex);
            throw ex;
        }

        return null; // response handled asynchronously
    }

    // ─── Inner Handler ──────────────────────────────────────────────

    /**
     * Bridges {@link SseEmitter} sends to the actual HTTP response output.
     *
     * Maps to: {@code ResponseBodyEmitterReturnValueHandler.DefaultSseEmitterHandler} (line 268)
     *
     * The real version iterates message converters to find one that can write
     * each data item. We simplify: Strings are written directly to the PrintWriter;
     * non-String objects are serialized to JSON using Jackson. This avoids the
     * need for a StreamingServletServerHttpResponse that absorbs Content-Type changes.
     */
    static class SseHandler implements SseEmitter.Handler {

        private final HttpServletResponse response;
        private final AsyncContext asyncContext;
        private final ObjectMapper objectMapper = new ObjectMapper();
        private final List<Runnable> timeoutCallbacks = new ArrayList<>();
        private final List<Consumer<Throwable>> errorCallbacks = new ArrayList<>();
        private final List<Runnable> completionCallbacks = new ArrayList<>();
        private volatile boolean asyncComplete;

        SseHandler(HttpServletResponse response, AsyncContext asyncContext) {
            this.response = response;
            this.asyncContext = asyncContext;
        }

        /**
         * Write a batch of SSE data items to the response and flush.
         *
         * Maps to: {@code DefaultSseEmitterHandler.send(Set)} (line 295)
         * The real version iterates converters per item; we write directly.
         *
         * For a typical SSE event like {@code event().id("1").name("msg").data(user)},
         * the items are:
         * <pre>
         *   1. ("id:1\nevent:msg\ndata:", TEXT_PLAIN)  → write as string
         *   2. (User{name="John"}, null)               → serialize to JSON
         *   3. ("\n\n", TEXT_PLAIN)                     → write as string
         * </pre>
         */
        @Override
        public void send(Collection<SseEmitter.DataWithMediaType> items) throws IOException {
            PrintWriter writer = response.getWriter();
            for (SseEmitter.DataWithMediaType item : items) {
                if (item.data() instanceof String str) {
                    writer.write(str);
                } else {
                    // Serialize non-String data to JSON.
                    // The real framework iterates message converters and uses
                    // StreamingServletServerHttpResponse to prevent Content-Type changes.
                    // We simplify by using Jackson directly.
                    writer.write(objectMapper.writeValueAsString(item.data()));
                }
            }
            writer.flush();
            if (writer.checkError()) {
                throw new IOException("Client disconnected");
            }
        }

        /**
         * Complete the async context normally.
         *
         * Maps to: {@code DefaultSseEmitterHandler.complete()} (line 318)
         * The real version flushes, then calls {@code deferredResult.setResult(null)},
         * which triggers async dispatch back to DispatcherServlet.
         * We simply complete the AsyncContext directly.
         */
        @Override
        public void complete() {
            if (asyncComplete) {
                return;
            }
            asyncComplete = true;
            try {
                response.getWriter().flush();
            } catch (IOException ignored) {
                // Client may have already disconnected
            }
            asyncContext.complete();
        }

        /**
         * Complete the async context with an error.
         *
         * Maps to: {@code DefaultSseEmitterHandler.completeWithError()} (line 330)
         */
        @Override
        public void completeWithError(Throwable failure) {
            if (asyncComplete) {
                return;
            }
            asyncComplete = true;
            asyncContext.complete();
        }

        @Override
        public void onTimeout(Runnable callback) {
            timeoutCallbacks.add(callback);
        }

        @Override
        public void onError(Consumer<Throwable> callback) {
            errorCallbacks.add(callback);
        }

        @Override
        public void onCompletion(Runnable callback) {
            completionCallbacks.add(callback);
        }

        // Callback dispatch methods called by the AsyncListener

        void fireTimeoutCallbacks() {
            for (Runnable cb : timeoutCallbacks) {
                cb.run();
            }
        }

        void fireErrorCallbacks(Throwable cause) {
            for (Consumer<Throwable> cb : errorCallbacks) {
                cb.accept(cause);
            }
        }

        void fireCompletionCallbacks() {
            for (Runnable cb : completionCallbacks) {
                cb.run();
            }
        }
    }
}
```

### ViewNameReturnValueHandler.java [MODIFIED]

`src/main/java/com/simplespringmvc/adapter/ViewNameReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Handles String return values by interpreting them as logical view names.
 * Produces a {@link ModelAndView} for the DispatcherServlet to render.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler}
 *
 * The real ViewNameMethodReturnValueHandler:
 * <ul>
 *   <li>Supports both {@code void.class} and {@code CharSequence} return types</li>
 *   <li>Handles "redirect:" and "forward:" prefixes specially</li>
 *   <li>Sets the view name on a {@code ModelAndViewContainer}</li>
 *   <li>For null returns, does nothing (response may already be committed)</li>
 * </ul>
 *
 * This handler is the bridge between the return value handler pattern and view
 * resolution. When a @Controller method returns a String like "hello", this
 * handler creates a ModelAndView("hello") that the DispatcherServlet will
 * resolve via ViewResolver → View → render.
 *
 * <h3>Ordering:</h3>
 * This handler must appear AFTER {@code ResponseBodyReturnValueHandler} in the
 * chain (so @ResponseBody methods get JSON, not view resolution) and BEFORE
 * {@code StringReturnValueHandler} (the catch-all fallback).
 *
 * Simplifications:
 * <ul>
 *   <li>Only handles String (not void) return types</li>
 *   <li>No "redirect:" or "forward:" prefix handling</li>
 *   <li>Returns ModelAndView directly instead of setting on ModelAndViewContainer</li>
 * </ul>
 */
public class ViewNameReturnValueHandler implements HandlerMethodReturnValueHandler {

    /**
     * Supports methods that return a String (CharSequence).
     *
     * Maps to: {@code ViewNameMethodReturnValueHandler.supportsReturnType()} (line 56)
     * Real version: {@code void.class == paramType || CharSequence.class.isAssignableFrom(paramType)}
     * We only handle CharSequence (String) — void returns are handled by the
     * StringReturnValueHandler catch-all.
     */
    /**
     * ch27: Uses handlerMethod.getReturnType() instead of getMethod().getReturnType()
     * so that ConcurrentResultHandlerMethod (on async re-dispatch) can override the
     * apparent return type to the actual result type.
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        Class<?> returnType = handlerMethod.getReturnType();
        return CharSequence.class.isAssignableFrom(returnType);
    }

    /**
     * Interpret the String return value as a logical view name.
     *
     * Maps to: {@code ViewNameMethodReturnValueHandler.handleReturnValue()} (line 63)
     * Real version sets the view name on the ModelAndViewContainer. We create
     * a ModelAndView directly.
     *
     * For null returns (e.g., the handler method returned null), we return
     * null ModelAndView — no view rendering needed.
     *
     * @return a ModelAndView with the view name, or null if returnValue is null
     */
    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue instanceof CharSequence viewName) {
            return new ModelAndView(viewName.toString());
        }
        // null return value — no view to render
        return null;
    }
}
```

### ModelAndViewReturnValueHandler.java [MODIFIED]

`src/main/java/com/simplespringmvc/adapter/ModelAndViewReturnValueHandler.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Handles return values of type {@link ModelAndView} by returning them directly.
 * Controller methods can construct a ModelAndView with both the view name and
 * model data, and this handler passes it through to the DispatcherServlet.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ModelAndViewMethodReturnValueHandler}
 *
 * The real version:
 * <ul>
 *   <li>Handles both ModelAndView returns and null returns</li>
 *   <li>Copies view name and model to the ModelAndViewContainer</li>
 *   <li>Handles "redirect:" prefix on the view name</li>
 *   <li>Sets requestHandled if the ModelAndView is empty</li>
 * </ul>
 *
 * <h3>Usage example:</h3>
 * <pre>
 *   &#64;RequestMapping(path = "/greeting", method = "GET")
 *   public ModelAndView greeting(@RequestParam("name") String name) {
 *       return new ModelAndView("greeting")
 *               .addObject("name", name)
 *               .addObject("time", LocalTime.now().toString());
 *   }
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Returns ModelAndView directly instead of copying to ModelAndViewContainer</li>
 *   <li>No "redirect:" prefix handling</li>
 * </ul>
 */
public class ModelAndViewReturnValueHandler implements HandlerMethodReturnValueHandler {

    /**
     * Supports methods whose return type is {@link ModelAndView}.
     *
     * Maps to: {@code ModelAndViewMethodReturnValueHandler.supportsReturnType()} (line 56)
     * Real version: {@code ModelAndView.class.isAssignableFrom(returnType.getParameterType())}
     */
    /**
     * ch27: Uses handlerMethod.getReturnType() instead of getMethod().getReturnType()
     * so that ConcurrentResultHandlerMethod (on async re-dispatch) can override the
     * apparent return type to the actual result type.
     */
    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return ModelAndView.class.isAssignableFrom(handlerMethod.getReturnType());
    }

    /**
     * Return the ModelAndView as-is. If the return value is null, return null
     * (no view rendering needed).
     *
     * Maps to: {@code ModelAndViewMethodReturnValueHandler.handleReturnValue()} (line 67)
     */
    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue instanceof ModelAndView mv) {
            return mv;
        }
        return null;
    }
}
```

### DeferredResultTest.java [TEST]

`src/test/java/com/simplespringmvc/async/DeferredResultTest.java`

```java
package com.simplespringmvc.async;

import org.junit.jupiter.api.Test;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link DeferredResult} — the async result container.
 *
 * These tests verify the thread coordination (rendezvous) pattern where
 * two threads race: one sets the result, the other sets the result handler.
 * Whichever arrives second triggers the dispatch.
 */
class DeferredResultTest {

    // ─── Basic result setting ────────────────────────────────────────

    @Test
    void shouldAcceptResult_WhenNotYetSet() {
        DeferredResult<String> dr = new DeferredResult<>();

        boolean accepted = dr.setResult("hello");

        assertThat(accepted).isTrue();
        assertThat(dr.hasResult()).isTrue();
        assertThat(dr.getResult()).isEqualTo("hello");
    }

    @Test
    void shouldRejectSecondResult_WhenAlreadySet() {
        DeferredResult<String> dr = new DeferredResult<>();
        dr.setResult("first");

        boolean accepted = dr.setResult("second");

        assertThat(accepted).isFalse();
        assertThat(dr.getResult()).isEqualTo("first");
    }

    @Test
    void shouldAcceptNullResult_WhenNullIsValid() {
        DeferredResult<String> dr = new DeferredResult<>();

        boolean accepted = dr.setResult(null);

        assertThat(accepted).isTrue();
        assertThat(dr.hasResult()).isTrue();
        assertThat(dr.getResult()).isNull();
    }

    @Test
    void shouldAcceptErrorResult_WhenNotYetSet() {
        DeferredResult<String> dr = new DeferredResult<>();
        RuntimeException error = new RuntimeException("test error");

        boolean accepted = dr.setErrorResult(error);

        assertThat(accepted).isTrue();
        assertThat(dr.hasResult()).isTrue();
        assertThat(dr.getResult()).isEqualTo(error);
    }

    // ─── Rendezvous: handler arrives first ───────────────────────────

    @Test
    void shouldCallHandler_WhenResultArrrivesAfterHandler() {
        DeferredResult<String> dr = new DeferredResult<>();
        AtomicReference<Object> handledResult = new AtomicReference<>();

        // Handler arrives first
        dr.setResultHandler(handledResult::set);

        // Then result arrives — should trigger handler
        dr.setResult("async value");

        assertThat(handledResult.get()).isEqualTo("async value");
    }

    // ─── Rendezvous: result arrives first ────────────────────────────

    @Test
    void shouldCallHandler_WhenResultArrivesBeforeHandler() {
        DeferredResult<String> dr = new DeferredResult<>();
        AtomicReference<Object> handledResult = new AtomicReference<>();

        // Result arrives first
        dr.setResult("early result");

        // Then handler arrives — should trigger handler immediately
        dr.setResultHandler(handledResult::set);

        assertThat(handledResult.get()).isEqualTo("early result");
    }

    // ─── Cross-thread coordination ───────────────────────────────────

    @Test
    void shouldCoordinateAcrossThreads_WhenResultSetFromAnotherThread() throws Exception {
        DeferredResult<String> dr = new DeferredResult<>();
        CountDownLatch latch = new CountDownLatch(1);
        AtomicReference<Object> handledResult = new AtomicReference<>();

        // Handler registered on main thread
        dr.setResultHandler(result -> {
            handledResult.set(result);
            latch.countDown();
        });

        // Result set from another thread
        Thread worker = new Thread(() -> dr.setResult("from worker"));
        worker.start();

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(handledResult.get()).isEqualTo("from worker");
    }

    // ─── Timeout handling ────────────────────────────────────────────

    @Test
    void shouldUseTimeoutResult_WhenTimeoutOccurs() {
        DeferredResult<String> dr = new DeferredResult<>(1000L, () -> "timeout value");

        boolean handled = dr.handleTimeout();

        assertThat(handled).isTrue();
        assertThat(dr.getResult()).isEqualTo("timeout value");
    }

    @Test
    void shouldRunTimeoutCallback_WhenTimeoutOccurs() {
        DeferredResult<String> dr = new DeferredResult<>();
        AtomicBoolean timeoutCalled = new AtomicBoolean();

        dr.onTimeout(() -> timeoutCalled.set(true));
        dr.handleTimeout();

        assertThat(timeoutCalled.get()).isTrue();
    }

    @Test
    void shouldNotSetTimeoutResult_WhenNoTimeoutResultConfigured() {
        DeferredResult<String> dr = new DeferredResult<>();

        boolean handled = dr.handleTimeout();

        assertThat(handled).isFalse();
        assertThat(dr.hasResult()).isFalse();
    }

    // ─── Error and completion callbacks ──────────────────────────────

    @Test
    void shouldRunErrorCallback_WhenErrorOccurs() {
        DeferredResult<String> dr = new DeferredResult<>();
        AtomicReference<Throwable> errorRef = new AtomicReference<>();
        RuntimeException error = new RuntimeException("async error");

        dr.onError(errorRef::set);
        dr.handleError(error);

        assertThat(errorRef.get()).isEqualTo(error);
    }

    @Test
    void shouldMarkExpiredAndRunCompletionCallback_WhenCompleted() {
        DeferredResult<String> dr = new DeferredResult<>();
        AtomicBoolean completionCalled = new AtomicBoolean();

        dr.onCompletion(() -> completionCalled.set(true));
        dr.handleCompletion();

        assertThat(completionCalled.get()).isTrue();
        assertThat(dr.isSetOrExpired()).isTrue();
    }

    @Test
    void shouldRejectResult_WhenExpired() {
        DeferredResult<String> dr = new DeferredResult<>();
        dr.handleCompletion(); // marks as expired

        boolean accepted = dr.setResult("too late");

        assertThat(accepted).isFalse();
    }

    // ─── State queries ──────────────────────────────────────────────

    @Test
    void shouldReportNotSet_WhenNoResultSet() {
        DeferredResult<String> dr = new DeferredResult<>();

        assertThat(dr.hasResult()).isFalse();
        assertThat(dr.isSetOrExpired()).isFalse();
        assertThat(dr.getResult()).isNull();
    }

    @Test
    void shouldReturnTimeout_WhenConfigured() {
        DeferredResult<String> dr = new DeferredResult<>(5000L);

        assertThat(dr.getTimeoutValue()).isEqualTo(5000L);
    }

    @Test
    void shouldReturnNullTimeout_WhenNotConfigured() {
        DeferredResult<String> dr = new DeferredResult<>();

        assertThat(dr.getTimeoutValue()).isNull();
    }
}
```

### WebAsyncManagerTest.java [TEST]

`src/test/java/com/simplespringmvc/async/WebAsyncManagerTest.java`

```java
package com.simplespringmvc.async;

import jakarta.servlet.AsyncContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.util.concurrent.Callable;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link WebAsyncManager} — the async lifecycle coordinator.
 *
 * These tests verify the state machine (NOT_STARTED → ASYNC_PROCESSING → RESULT_SET),
 * per-request singleton semantics, and Callable/DeferredResult processing.
 *
 * Note: MockHttpServletRequest from Spring Test supports async mode when
 * setAsyncSupported(true) is called.
 */
class WebAsyncManagerTest {

    private MockHttpServletRequest request;
    private MockHttpServletResponse response;

    @BeforeEach
    void setUp() {
        request = new MockHttpServletRequest();
        request.setAsyncSupported(true);
        response = new MockHttpServletResponse();
    }

    // ─── Per-request singleton ───────────────────────────────────────

    @Test
    void shouldReturnSameInstance_WhenCalledTwiceForSameRequest() {
        WebAsyncManager first = WebAsyncManager.getAsyncManager(request);
        WebAsyncManager second = WebAsyncManager.getAsyncManager(request);

        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldReturnDifferentInstance_WhenCalledForDifferentRequests() {
        MockHttpServletRequest otherRequest = new MockHttpServletRequest();

        WebAsyncManager first = WebAsyncManager.getAsyncManager(request);
        WebAsyncManager second = WebAsyncManager.getAsyncManager(otherRequest);

        assertThat(first).isNotSameAs(second);
    }

    // ─── Initial state ──────────────────────────────────────────────

    @Test
    void shouldNotHaveConcurrentResult_WhenNewlyCreated() {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);

        assertThat(manager.isConcurrentHandlingStarted()).isFalse();
        assertThat(manager.hasConcurrentResult()).isFalse();
    }

    // ─── Callable processing ─────────────────────────────────────────

    @Test
    void shouldStartAsyncAndExecuteCallable_WhenCallableReturnsValue() throws Exception {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        CountDownLatch latch = new CountDownLatch(1);
        AtomicReference<Object> dispatchedResult = new AtomicReference<>();

        Callable<String> callable = () -> "async result";

        manager.startCallableProcessing(callable, request, response);

        // Wait for the callable to complete
        // The async dispatch happens on a separate thread
        assertThat(manager.isConcurrentHandlingStarted()).isTrue();

        // Give the background thread time to execute
        Thread.sleep(500);

        // The result should be saved
        assertThat(manager.hasConcurrentResult()).isTrue();
        assertThat(manager.getConcurrentResult()).isEqualTo("async result");
    }

    @Test
    void shouldSaveThrowable_WhenCallableThrows() throws Exception {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        RuntimeException error = new RuntimeException("callable failed");

        Callable<String> callable = () -> { throw error; };

        manager.startCallableProcessing(callable, request, response);

        // Wait for the callable to complete
        Thread.sleep(500);

        assertThat(manager.hasConcurrentResult()).isTrue();
        assertThat(manager.getConcurrentResult()).isEqualTo(error);
    }

    @Test
    void shouldRejectSecondStart_WhenAlreadyStarted() {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        manager.startCallableProcessing(() -> "first", request, response);

        assertThatThrownBy(() ->
                manager.startCallableProcessing(() -> "second", request, response))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("already started");
    }

    // ─── DeferredResult processing ───────────────────────────────────

    @Test
    void shouldStartAsyncAndWireHandler_WhenDeferredResultProcessed() {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        DeferredResult<String> dr = new DeferredResult<>();

        manager.startDeferredResultProcessing(dr, request, response);

        assertThat(manager.isConcurrentHandlingStarted()).isTrue();
        assertThat(manager.hasConcurrentResult()).isFalse(); // no result yet
    }

    @Test
    void shouldSaveResult_WhenDeferredResultIsSet() throws Exception {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        DeferredResult<String> dr = new DeferredResult<>();

        manager.startDeferredResultProcessing(dr, request, response);

        // Application sets the result
        dr.setResult("deferred value");

        // Result should be saved in the manager
        assertThat(manager.hasConcurrentResult()).isTrue();
        assertThat(manager.getConcurrentResult()).isEqualTo("deferred value");
    }

    @Test
    void shouldSaveErrorResult_WhenDeferredResultErrorSet() {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        DeferredResult<String> dr = new DeferredResult<>();
        RuntimeException error = new RuntimeException("deferred error");

        manager.startDeferredResultProcessing(dr, request, response);
        dr.setErrorResult(error);

        assertThat(manager.hasConcurrentResult()).isTrue();
        assertThat(manager.getConcurrentResult()).isEqualTo(error);
    }

    @Test
    void shouldApplyTimeout_WhenDeferredResultHasTimeout() {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        DeferredResult<String> dr = new DeferredResult<>(5000L);

        manager.startDeferredResultProcessing(dr, request, response);

        // The timeout is configured on the AsyncContext via the mock
        assertThat(manager.isConcurrentHandlingStarted()).isTrue();
    }

    // ─── State management ────────────────────────────────────────────

    @Test
    void shouldResetState_WhenConcurrentResultCleared() throws Exception {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);

        manager.startCallableProcessing(() -> "result", request, response);
        Thread.sleep(500);

        assertThat(manager.hasConcurrentResult()).isTrue();

        manager.clearConcurrentResult();

        assertThat(manager.hasConcurrentResult()).isFalse();
        assertThat(manager.isConcurrentHandlingStarted()).isFalse();
    }

    // ─── Cross-thread DeferredResult ─────────────────────────────────

    @Test
    void shouldHandleCrossThreadResult_WhenSetFromWorkerThread() throws Exception {
        WebAsyncManager manager = WebAsyncManager.getAsyncManager(request);
        DeferredResult<String> dr = new DeferredResult<>();
        CountDownLatch latch = new CountDownLatch(1);

        manager.startDeferredResultProcessing(dr, request, response);

        // Set result from another thread
        Thread worker = new Thread(() -> {
            dr.setResult("from worker");
            latch.countDown();
        });
        worker.start();

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(manager.getConcurrentResult()).isEqualTo("from worker");
    }
}
```

### CallableReturnValueHandlerTest.java [TEST]

`src/test/java/com/simplespringmvc/adapter/CallableReturnValueHandlerTest.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.async.DeferredResult;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.util.concurrent.Callable;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link CallableReturnValueHandler}.
 *
 * Verifies return type detection and that Callable doesn't match on re-dispatch
 * when wrapped in ConcurrentResultHandlerMethod.
 */
class CallableReturnValueHandlerTest {

    private final CallableReturnValueHandler handler = new CallableReturnValueHandler();

    // ─── supportsReturnType ──────────────────────────────────────────

    @Test
    void shouldSupportCallableReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("callableMethod"));

        assertThat(handler.supportsReturnType(hm)).isTrue();
    }

    @Test
    void shouldNotSupportStringReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("stringMethod"));

        assertThat(handler.supportsReturnType(hm)).isFalse();
    }

    @Test
    void shouldNotSupportDeferredResultReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("deferredResultMethod"));

        assertThat(handler.supportsReturnType(hm)).isFalse();
    }

    @Test
    void shouldNotMatchOnReDispatch_WhenWrappedInConcurrentResultHandlerMethod() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("callableMethod"));

        // Simulate re-dispatch: wrap with actual result type (String)
        HandlerMethod wrapped = hm.wrapConcurrentResult("async result");

        // Should NOT match — the wrapper changes getReturnType() to String
        assertThat(handler.supportsReturnType(wrapped)).isFalse();
    }

    // ─── handleReturnValue ───────────────────────────────────────────

    @Test
    void shouldReturnNull_WhenReturnValueIsNull() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("callableMethod"));
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();

        ModelAndView mv = handler.handleReturnValue(null, hm, request, response);

        assertThat(mv).isNull();
    }

    @Test
    void shouldStartAsyncProcessing_WhenCallableProvided() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("callableMethod"));
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setAsyncSupported(true);
        MockHttpServletResponse response = new MockHttpServletResponse();

        Callable<String> callable = () -> "result";

        ModelAndView mv = handler.handleReturnValue(callable, hm, request, response);

        assertThat(mv).isNull(); // response handled asynchronously
        assertThat(request.isAsyncStarted()).isTrue();
    }

    // ─── Test controller ─────────────────────────────────────────────

    static class TestController {
        public Callable<String> callableMethod() { return () -> "ok"; }
        public String stringMethod() { return "ok"; }
        public DeferredResult<String> deferredResultMethod() { return new DeferredResult<>(); }
    }
}
```

### DeferredResultReturnValueHandlerTest.java [TEST]

`src/test/java/com/simplespringmvc/adapter/DeferredResultReturnValueHandlerTest.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.async.DeferredResult;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.view.ModelAndView;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.util.concurrent.Callable;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link DeferredResultReturnValueHandler}.
 *
 * Verifies return type detection for DeferredResult and CompletionStage,
 * and that neither matches on re-dispatch.
 */
class DeferredResultReturnValueHandlerTest {

    private final DeferredResultReturnValueHandler handler = new DeferredResultReturnValueHandler();

    // ─── supportsReturnType ──────────────────────────────────────────

    @Test
    void shouldSupportDeferredResultReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("deferredResultMethod"));

        assertThat(handler.supportsReturnType(hm)).isTrue();
    }

    @Test
    void shouldSupportCompletionStageReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("completionStageMethod"));

        assertThat(handler.supportsReturnType(hm)).isTrue();
    }

    @Test
    void shouldSupportCompletableFutureReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("completableFutureMethod"));

        assertThat(handler.supportsReturnType(hm)).isTrue();
    }

    @Test
    void shouldNotSupportCallableReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("callableMethod"));

        assertThat(handler.supportsReturnType(hm)).isFalse();
    }

    @Test
    void shouldNotSupportStringReturnType() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("stringMethod"));

        assertThat(handler.supportsReturnType(hm)).isFalse();
    }

    @Test
    void shouldNotMatchOnReDispatch_WhenWrappedInConcurrentResultHandlerMethod() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("deferredResultMethod"));

        HandlerMethod wrapped = hm.wrapConcurrentResult("async result");

        assertThat(handler.supportsReturnType(wrapped)).isFalse();
    }

    // ─── handleReturnValue with DeferredResult ───────────────────────

    @Test
    void shouldReturnNull_WhenReturnValueIsNull() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("deferredResultMethod"));
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();

        ModelAndView mv = handler.handleReturnValue(null, hm, request, response);

        assertThat(mv).isNull();
    }

    @Test
    void shouldStartAsyncProcessing_WhenDeferredResultProvided() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("deferredResultMethod"));
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setAsyncSupported(true);
        MockHttpServletResponse response = new MockHttpServletResponse();

        DeferredResult<String> dr = new DeferredResult<>();

        ModelAndView mv = handler.handleReturnValue(dr, hm, request, response);

        assertThat(mv).isNull();
        assertThat(request.isAsyncStarted()).isTrue();
    }

    // ─── handleReturnValue with CompletionStage ──────────────────────

    @Test
    void shouldStartAsyncAndAdaptCompletionStage_WhenCompletableFutureProvided() throws Exception {
        HandlerMethod hm = new HandlerMethod(new TestController(),
                TestController.class.getMethod("completableFutureMethod"));
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setAsyncSupported(true);
        MockHttpServletResponse response = new MockHttpServletResponse();

        CompletableFuture<String> future = new CompletableFuture<>();

        ModelAndView mv = handler.handleReturnValue(future, hm, request, response);

        assertThat(mv).isNull();
        assertThat(request.isAsyncStarted()).isTrue();
    }

    // ─── Test controller ─────────────────────────────────────────────

    static class TestController {
        public DeferredResult<String> deferredResultMethod() { return new DeferredResult<>(); }
        public CompletionStage<String> completionStageMethod() { return CompletableFuture.completedFuture("ok"); }
        public CompletableFuture<String> completableFutureMethod() { return CompletableFuture.completedFuture("ok"); }
        public Callable<String> callableMethod() { return () -> "ok"; }
        public String stringMethod() { return "ok"; }
    }
}
```

### AsyncRequestHandlingIntegrationTest.java [TEST]

`src/test/java/com/simplespringmvc/integration/AsyncRequestHandlingIntegrationTest.java`

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.Controller;
import com.simplespringmvc.annotation.RequestMapping;
import com.simplespringmvc.annotation.ResponseBody;
import com.simplespringmvc.annotation.RestController;
import com.simplespringmvc.async.DeferredResult;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.Callable;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for async request handling (Feature 27).
 *
 * Tests the full lifecycle: HTTP request → DispatcherServlet → handler mapping
 * → handler invocation (returns Callable/DeferredResult) → async processing →
 * re-dispatch → result processing → HTTP response.
 *
 * These tests verify the end-to-end async flow through the real embedded Tomcat
 * server with actual HTTP requests.
 */
class AsyncRequestHandlingIntegrationTest {

    private EmbeddedTomcat tomcat;
    private HttpClient client;
    private String baseUrl;

    // Shared DeferredResult for cross-thread test — set by controller, completed by test
    static volatile DeferredResult<String> pendingResult;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean("asyncController", new AsyncController());
        container.registerBean("restAsyncController", new RestAsyncController());

        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.start();

        baseUrl = "http://localhost:" + tomcat.getPort();
        client = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        if (tomcat != null) {
            tomcat.stop();
        }
    }

    // ─── Callable tests ──────────────────────────────────────────────

    @Test
    void shouldReturnAsyncResult_WhenCallableCompletesSuccessfully() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/async/callable"))
                .header("Accept", "text/plain")
                .GET().build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("callable result");
    }

    @Test
    void shouldReturnJsonResult_WhenRestCallableCompletesSuccessfully() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/async/rest-callable"))
                .header("Accept", "application/json")
                .GET().build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("\"message\"");
        assertThat(response.body()).contains("async hello");
    }

    @Test
    void shouldReturnError_WhenCallableThrows() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/async/callable-error"))
                .GET().build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        // The exception should be handled by the exception resolver pipeline
        assertThat(response.statusCode()).isEqualTo(500);
    }

    // ─── DeferredResult tests ────────────────────────────────────────

    @Test
    void shouldReturnAsyncResult_WhenDeferredResultSetImmediately() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/async/deferred-immediate"))
                .header("Accept", "text/plain")
                .GET().build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("deferred immediate");
    }

    @Test
    void shouldReturnAsyncResult_WhenDeferredResultSetFromAnotherThread() throws Exception {
        // Start the request in a background thread (it will block until
        // the DeferredResult is completed)
        CompletableFuture<HttpResponse<String>> futureResponse = CompletableFuture.supplyAsync(() -> {
            try {
                HttpRequest request = HttpRequest.newBuilder()
                        .uri(URI.create(baseUrl + "/async/deferred-delayed"))
                        .header("Accept", "text/plain")
                        .GET().build();
                return client.send(request, HttpResponse.BodyHandlers.ofString());
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });

        // Wait for the controller to set the pendingResult reference
        Thread.sleep(500);

        // Complete the DeferredResult from the test thread
        if (pendingResult != null) {
            pendingResult.setResult("delayed result");
        }

        HttpResponse<String> response = futureResponse.get();

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).isEqualTo("delayed result");
    }

    @Test
    void shouldReturnJsonDeferredResult_WhenRestControllerReturnsDeferred() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/async/rest-deferred"))
                .header("Accept", "application/json")
                .GET().build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("\"message\"");
        assertThat(response.body()).contains("deferred hello");
    }

    // ─── CompletableFuture tests ─────────────────────────────────────

    @Test
    void shouldReturnResult_WhenCompletableFutureCompletes() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/async/completable-future"))
                .header("Accept", "application/json")
                .GET().build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("\"message\"");
        assertThat(response.body()).contains("future result");
    }

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    static class AsyncController {

        @RequestMapping(path = "/async/callable", method = "GET")
        @ResponseBody
        public Callable<String> callableMethod() {
            return () -> {
                Thread.sleep(100); // simulate slow work
                return "callable result";
            };
        }

        @RequestMapping(path = "/async/callable-error", method = "GET")
        @ResponseBody
        public Callable<String> callableErrorMethod() {
            return () -> {
                throw new RuntimeException("Callable failed!");
            };
        }

        @RequestMapping(path = "/async/deferred-immediate", method = "GET")
        @ResponseBody
        public DeferredResult<String> deferredImmediateMethod() {
            DeferredResult<String> dr = new DeferredResult<>();
            // Set result immediately — tests the "result arrives before handler" path
            dr.setResult("deferred immediate");
            return dr;
        }

        @RequestMapping(path = "/async/deferred-delayed", method = "GET")
        @ResponseBody
        public DeferredResult<String> deferredDelayedMethod() {
            DeferredResult<String> dr = new DeferredResult<>(30000L);
            // Store for the test to complete later
            pendingResult = dr;
            return dr;
        }
    }

    @RestController
    static class RestAsyncController {

        @RequestMapping(path = "/async/rest-callable", method = "GET")
        public Callable<AsyncMessage> restCallableMethod() {
            return () -> new AsyncMessage("async hello");
        }

        @RequestMapping(path = "/async/rest-deferred", method = "GET")
        public DeferredResult<AsyncMessage> restDeferredMethod() {
            DeferredResult<AsyncMessage> dr = new DeferredResult<>();
            // Complete asynchronously from another thread
            new Thread(() -> {
                try {
                    Thread.sleep(50);
                } catch (InterruptedException ignored) {}
                dr.setResult(new AsyncMessage("deferred hello"));
            }).start();
            return dr;
        }

        @RequestMapping(path = "/async/completable-future", method = "GET")
        public CompletionStage<AsyncMessage> completableFutureMethod() {
            return CompletableFuture.supplyAsync(() -> new AsyncMessage("future result"));
        }
    }

    /**
     * Simple DTO for JSON serialization testing.
     */
    public static class AsyncMessage {
        private String message;

        public AsyncMessage() {}

        public AsyncMessage(String message) {
            this.message = message;
        }

        public String getMessage() { return message; }
        public void setMessage(String message) { this.message = message; }
    }
}
```

---

## Summary

| Component | Role |
|---|---|
| `DeferredResult<T>` | Thread-safe async result container with rendezvous pattern |
| `WebAsyncManager` | Per-request lifecycle coordinator with state machine |
| `AsyncHandlerInterceptor` | Extension for async-aware interceptor cleanup |
| `CallableReturnValueHandler` | Starts async for `Callable` returns |
| `DeferredResultReturnValueHandler` | Starts async for `DeferredResult`/`CompletionStage` returns |
| `ConcurrentResultHandlerMethod` | Changes apparent return type to prevent infinite async loops |
| `HandlerExecutionChain` (modified) | Notifies async interceptors via `applyAfterConcurrentHandlingStarted` |
| `SimpleHandlerAdapter` (modified) | Detects re-dispatch, processes saved concurrent result |
| `SimpleDispatcherServlet` (modified) | WebAsyncManager-aware async lifecycle check |

**Next chapter:** Chapter 28 -- AOP / Proxy-Based Features.
