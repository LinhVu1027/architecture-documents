# Chapter 22: SSE (Server-Sent Events)

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The framework handles synchronous request/response — every handler method writes its result immediately, and the connection closes after `doDispatch()` completes | No way to keep a connection open and push events from server to client over time (stock tickers, notifications, log streams) | Build an `SseEmitter` that manages a long-lived HTTP connection with SSE wire format, thread-safe sends, early send buffering, and lifecycle callbacks — plus the return value handler and async dispatch integration to make it work end-to-end |

---

## 22.1 The Integration Point

SSE requires connecting at **three** separate seams in the existing system:

### Seam 1: EmbeddedTomcat — Enable async support

**Modifying:** `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java`
**Change:** Call `wrapper.setAsyncSupported(true)` after registering the servlet

```java
var wrapper = Tomcat.addServlet(context, "dispatcher", dispatcherServlet);
context.addServletMappingDecoded("/", "dispatcher");

// ch22: Enable async support so SSE can call request.startAsync()
wrapper.setAsyncSupported(true);
```

Without this single line, `request.startAsync()` throws `IllegalStateException`. In Spring Boot, `ServletRegistrationBean.setAsyncSupported(true)` is the default.

### Seam 2: SimpleDispatcherServlet — Skip post-processing for async

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** After `adapter.handle()`, check if async has started and return immediately

```java
mv = adapter.handle(request, response, mappedHandler.getHandler());

// ch22: If async processing has started (e.g., SSE emitter returned),
// skip postHandle, afterCompletion, and view rendering.
if (request.isAsyncStarted()) {
    return;
}

mappedHandler.applyPostHandle(request, response);
```

Two key decisions here:

1. **Why `request.isAsyncStarted()` instead of a custom flag?** The Servlet API already tracks async state — using it directly is simpler and more correct than the real framework's `WebAsyncManager.isConcurrentHandlingStarted()`, which wraps the same call with additional state management.
2. **Why skip everything, not just interceptors?** Once async starts, the connection belongs to the emitter. View rendering, exception handling, and interceptor cleanup would all operate on a response that's still being written to. The async lifecycle (AsyncContext listeners) handles completion instead.

This connects the **dispatch pipeline** to the **async Servlet infrastructure**. To make it work, we need to build:
- `SseEmitter` — the emitter that manages connection state, event formatting, and thread safety
- `SseEmitterReturnValueHandler` — detects SseEmitter returns, starts async, creates the I/O bridge
- Registration in `SimpleHandlerAdapter` — so the handler is found before `ResponseBodyReturnValueHandler`

### Seam 3: SimpleHandlerAdapter — Register the SSE handler first

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Add `SseEmitterReturnValueHandler` before `ResponseBodyReturnValueHandler`

```java
// ch22: SseEmitterReturnValueHandler BEFORE ResponseBodyReturnValueHandler
returnValueHandlers.addHandler(new SseEmitterReturnValueHandler());
returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
        messageConverters, contentNegotiationManager));
```

This ordering is critical: `@RestController` has class-level `@ResponseBody`, so `ResponseBodyReturnValueHandler.supportsReturnType()` returns `true` for ALL methods on such classes. By registering the SSE handler first, methods returning `SseEmitter` are correctly intercepted before the JSON handler sees them.

---

## 22.2 The SseEmitter

The core class combines Spring's `ResponseBodyEmitter` and `SseEmitter` into one. It has three responsibilities: thread-safe connection management, SSE wire format construction, and lifecycle callback dispatch.

**New file:** `src/main/java/com/simplespringmvc/sse/SseEmitter.java`

```java
package com.simplespringmvc.sse;

import com.simplespringmvc.http.MediaType;
import java.io.IOException;
import java.util.*;
import java.util.concurrent.locks.ReentrantLock;
import java.util.function.Consumer;

public class SseEmitter {

    private final Long timeout;
    private Handler handler;
    private final List<DataWithMediaType> earlySendAttempts = new ArrayList<>(8);
    private boolean complete;
    private Throwable failure;
    private final List<Runnable> timeoutCallbacks = new ArrayList<>(1);
    private final List<Consumer<Throwable>> errorCallbacks = new ArrayList<>(1);
    private final List<Runnable> completionCallbacks = new ArrayList<>(1);
    protected final ReentrantLock writeLock = new ReentrantLock();

    public SseEmitter() { this(null); }
    public SseEmitter(Long timeout) { this.timeout = timeout; }
    public Long getTimeout() { return timeout; }

    // ─── Send methods ──────────────────────────────────────────
    public void send(Object data) throws IOException {
        send(event().data(data));
    }

    public void send(SseEventBuilder builder) throws IOException {
        Collection<DataWithMediaType> dataToSend = builder.build();
        writeLock.lock();
        try {
            if (complete) throw new IllegalStateException("SseEmitter has already completed");
            if (handler != null) {
                handler.send(dataToSend);
            } else {
                earlySendAttempts.addAll(dataToSend);
            }
        } catch (IOException | IllegalStateException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new IllegalStateException("Failed to send SSE event", ex);
        } finally {
            writeLock.unlock();
        }
    }

    // ─── Lifecycle ─────────────────────────────────────────────
    public void complete() { /* sets complete=true, delegates to handler */ }
    public void completeWithError(Throwable ex) { /* sets complete+failure, delegates */ }
    public void onTimeout(Runnable callback) { /* registers or forwards */ }
    public void onError(Consumer<Throwable> callback) { /* registers or forwards */ }
    public void onCompletion(Runnable callback) { /* registers or forwards */ }

    // ─── Initialization ────────────────────────────────────────
    public void initialize(Handler handler) throws IOException {
        writeLock.lock();
        try {
            this.handler = handler;
            if (!earlySendAttempts.isEmpty()) {
                handler.send(List.copyOf(earlySendAttempts));
                earlySendAttempts.clear();
            }
            if (complete) { /* notify handler of completion */ return; }
            // Forward all registered callbacks to the handler
        } finally { writeLock.unlock(); }
    }

    // ─── Event builder ─────────────────────────────────────────
    public static SseEventBuilder event() { return new SseEventBuilderImpl(); }

    // ─── Inner types ───────────────────────────────────────────
    public interface Handler {
        void send(Collection<DataWithMediaType> items) throws IOException;
        void complete();
        void completeWithError(Throwable failure);
        void onTimeout(Runnable callback);
        void onError(Consumer<Throwable> callback);
        void onCompletion(Runnable callback);
    }

    public interface SseEventBuilder { /* id, name, reconnectTime, comment, data, build */ }
    public record DataWithMediaType(Object data, MediaType mediaType) {}
}
```

The **two-phase lifecycle** is the central design:
- **Phase 1 (before `initialize`):** The controller method has returned the emitter but the return value handler hasn't wired it to the response yet. Any `send()` calls are buffered.
- **Phase 2 (after `initialize`):** The handler is wired. Sends go directly to the response. Buffered sends are flushed.

### The SseEventBuilder — Interleaving Protocol and Data

The SSE wire protocol has fields like `id:`, `event:`, `data:`, `retry:`. The builder accumulates these as plain text, but the data payload might be a Java object that needs JSON serialization. The solution: build a list of `DataWithMediaType` entries where protocol text is String/TEXT_PLAIN and data payloads carry their own media type.

For `event().id("1").name("update").data(user)`, the builder produces:

```
┌──────────────────────────────────────────┬────────────┐
│ DataWithMediaType                        │ MediaType  │
├──────────────────────────────────────────┼────────────┤
│ "id:1\nevent:update\ndata:"             │ TEXT_PLAIN │  ← protocol framing
│ User{name="John"}                        │ null       │  ← data payload (serialize with Jackson)
│ "\n\n"                                   │ TEXT_PLAIN │  ← event terminator
└──────────────────────────────────────────┴────────────┘
```

On the wire, this becomes:
```
id:1
event:update
data:{"name":"John"}

```

---

## 22.3 The SseEmitterReturnValueHandler

**New file:** `src/main/java/com/simplespringmvc/adapter/SseEmitterReturnValueHandler.java`

```java
public class SseEmitterReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return SseEmitter.class.isAssignableFrom(handlerMethod.getMethod().getReturnType());
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
            HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (returnValue == null) return null;
        SseEmitter emitter = (SseEmitter) returnValue;

        // 1. Set SSE headers
        response.setContentType("text/event-stream");
        response.setCharacterEncoding("UTF-8");
        response.setHeader("Cache-Control", "no-cache");

        // 2. Start async mode
        AsyncContext asyncContext = request.startAsync(request, response);
        asyncContext.setTimeout(emitter.getTimeout() != null ? emitter.getTimeout() : 0);

        // 3. Create the I/O bridge handler
        SseHandler sseHandler = new SseHandler(response, asyncContext);

        // 4. Register async lifecycle listeners (timeout/error/completion)
        asyncContext.addListener(new AsyncListener() { /* ... */ });

        // 5. Initialize the emitter
        emitter.initialize(sseHandler);

        return null; // response handled asynchronously
    }
}
```

The `SseHandler` inner class bridges emitter sends to the response:
- **Strings** → write directly to `response.getWriter()`
- **Objects** → serialize to JSON with Jackson `ObjectMapper`
- **Complete** → flush and call `asyncContext.complete()`

This simplification bypasses the real framework's `StreamingServletServerHttpResponse` wrapper (which prevents message converters from modifying response headers on an already-committed response) by using Jackson directly.

---

## 22.4 Try It Yourself

<details>
<summary>Challenge 1: Implement the SseEventBuilder that produces correct SSE wire format</summary>

The builder needs to interleave protocol text with data objects. The key insight is `saveAppendedText()` — when you hit a `data()` call, flush the accumulated StringBuilder as a TEXT_PLAIN entry, then add the data object separately.

```java
private static class SseEventBuilderImpl implements SseEventBuilder {
    private final List<DataWithMediaType> dataToSend = new ArrayList<>(4);
    private StringBuilder sb;

    private void append(String text) {
        if (sb == null) sb = new StringBuilder();
        sb.append(text);
    }

    private void saveAppendedText() {
        if (sb != null) {
            dataToSend.add(new DataWithMediaType(sb.toString(), MediaType.TEXT_PLAIN));
            sb = null;
        }
    }

    @Override
    public SseEventBuilder data(Object data, MediaType mediaType) {
        append("data:");
        saveAppendedText();  // flush protocol text accumulated so far
        if (data instanceof String str) {
            data = str.replace("\n", "\ndata:");  // multi-line SSE compliance
        }
        dataToSend.add(new DataWithMediaType(data, mediaType));
        append("\n");
        return this;
    }

    @Override
    public Collection<DataWithMediaType> build() {
        append("\n");  // blank line = event terminator
        saveAppendedText();
        return dataToSend;
    }
}
```

</details>

<details>
<summary>Challenge 2: Handle the async dispatch skip in doDispatch()</summary>

After `adapter.handle()` returns, the response may already be in async mode. You need to check and bail out before post-processing runs.

```java
mv = adapter.handle(request, response, mappedHandler.getHandler());

// ch22: If async processing has started, skip post-processing.
// The async lifecycle manages its own completion.
if (request.isAsyncStarted()) {
    return;
}

mappedHandler.applyPostHandle(request, response);
```

This single check replaces what the real framework does with `WebAsyncManager.isConcurrentHandlingStarted()`.

</details>

---

## 22.5 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/sse/SseEmitterTest.java`

Key test methods:

| Test | What It Verifies |
|------|-----------------|
| `shouldBuildEventWithAllFields` | SSE wire format: id, event, retry, comment, data interleaved correctly |
| `shouldHandleMultilineStringData` | Multi-line strings get `\ndata:` inserted per SSE spec |
| `shouldBufferEarlySends_WhenNotInitialized` | Sends before `initialize()` are buffered and flushed |
| `shouldRejectSend_WhenComplete` | `IllegalStateException` after `complete()` |
| `shouldNotifyHandler_WhenCompleteBeforeInit` | Complete-before-init notifies handler during initialization |
| `shouldForwardTimeoutCallback_WhenRegisteredBeforeInit` | Callbacks registered before init are forwarded |
| `shouldHandleConcurrentSends_WithoutCorruption` | 10 threads × 100 sends = 1000 sends without corruption |

**New file:** `src/test/java/com/simplespringmvc/adapter/SseEmitterReturnValueHandlerTest.java`

| Test | What It Verifies |
|------|-----------------|
| `shouldSupportSseEmitterReturnType` | Handler matches `SseEmitter` return type |
| `shouldNotSupportStringReturnType` | Handler doesn't match non-SSE types |
| `shouldSetSseContentType` | Response has `text/event-stream` content type |
| `shouldStartAsyncProcessing` | `request.isAsyncStarted()` is true after handling |
| `shouldWriteJsonData_WhenObjectSent` | Objects are serialized to JSON in SSE data field |

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/SseIntegrationTest.java`

| Test | What It Verifies |
|------|-----------------|
| `shouldReceiveSseEvents_WhenControllerSendsData` | End-to-end: Tomcat → controller → SseEmitter → client receives SSE events |
| `shouldReceiveNamedEvents_WithIdAndEventType` | Full SSE protocol: id, event type, data |
| `shouldReceiveJsonEvents_WhenObjectsSent` | Objects serialized to JSON within SSE events |
| `shouldWorkWithRestController` | SSE works on `@RestController` (SSE handler takes priority over `@ResponseBody`) |
| `shouldReceiveMultipleEvents_InCorrectOrder` | Events arrive in send order |
| `shouldReceiveReconnectTime_WhenRetrySet` | `retry:` field is included |

**Run:** `./gradlew test` — expected: 759 tests pass (including all prior features)

---

## 22.5 Why This Works

> `★ Insight ─────────────────────────────────────`
> **The two-phase lifecycle solves a race condition.** When a controller returns an `SseEmitter`, there's a gap between the return (which triggers the return value handler) and when the emitter is wired to the response. If a background thread starts sending immediately, those sends need somewhere to go. The `earlySendAttempts` buffer catches them. During `initialize()`, under the same `writeLock`, buffered sends are flushed and the handler is set — atomically. This is a classic **initialize-then-delegate** pattern for bridging async producers with lazy consumers.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **SSE separates protocol framing from data serialization.** The `DataWithMediaType` list interleaves plain-text protocol fields (written as Strings) with data payloads (serialized by Jackson). This is why the real framework's `DefaultSseEmitterHandler` iterates message converters per item — protocol text goes through `StringHttpMessageConverter`, while data objects go through `MappingJackson2HttpMessageConverter`. Our simplified version achieves the same result by checking `instanceof String`. The interleaving pattern means adding XML support would only require adding an XML converter — no SSE code changes needed.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **Return value handler ordering is a silent correctness requirement.** `SseEmitterReturnValueHandler` must be registered before `ResponseBodyReturnValueHandler`. The real framework enforces this in `RequestMappingHandlerAdapter.getDefaultReturnValueHandlers()` (line 681) where `ResponseBodyEmitterReturnValueHandler` appears before `RequestResponseBodyMethodProcessor`. Getting this wrong doesn't cause an error — it causes the `SseEmitter` to be serialized as JSON (`{"timeout":null,"complete":false,...}`), which is a much harder bug to diagnose.
> `─────────────────────────────────────────────────`

---

## 22.6 What We Enhanced

| Aspect | Before (ch21) | Current (ch22) | Real Framework |
|--------|---------------|----------------|----------------|
| HTTP connection model | Synchronous only — response written and closed in `doDispatch()` | Supports long-lived async connections via Servlet `AsyncContext` | Uses `WebAsyncManager` + `DeferredResult` abstraction layer over `AsyncContext` (`WebAsyncManager.java:398`) |
| Async dispatch handling | No async checks — `doDispatch()` always runs full pipeline | Checks `request.isAsyncStarted()` and skips postHandle/afterCompletion | Checks `asyncManager.isConcurrentHandlingStarted()` at line 965, also calls `applyAfterConcurrentHandlingStarted()` on `AsyncHandlerInterceptor`s |
| Servlet async support | Not enabled | `wrapper.setAsyncSupported(true)` in `EmbeddedTomcat` | Enabled by default via `ServletRegistrationBean` |
| Data serialization for SSE | N/A | Jackson `ObjectMapper` directly | Iterates `HttpMessageConverter` list with `StreamingServletServerHttpResponse` wrapper to absorb header changes |
| Return value handler chain | ResponseBody → ModelAndView → ViewName → String | **SseEmitter** → ResponseBody → ModelAndView → ViewName → String | `ResponseBodyEmitterReturnValueHandler` before `RequestResponseBodyMethodProcessor` (`RequestMappingHandlerAdapter.java:681`) |

---

## 22.7 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `SseEmitter` | `ResponseBodyEmitter` + `SseEmitter` | `ResponseBodyEmitter.java:71`, `SseEmitter.java:44` | Two-class hierarchy; base class supports generic streaming, subclass adds SSE format |
| `SseEmitter.Handler` | `ResponseBodyEmitter.Handler` | `ResponseBodyEmitter.java:375` | Real version has both `send(Object, MediaType)` and `send(Set<DataWithMediaType>)` |
| `SseEmitter.SseEventBuilder` | `SseEmitter.SseEventBuilder` + `SseEventBuilderImpl` | `SseEmitter.java:151` | Real version supports `ModelAndView` data for SSE fragment rendering (6.2+) |
| `SseEmitterReturnValueHandler` | `ResponseBodyEmitterReturnValueHandler` | `ResponseBodyEmitterReturnValueHandler.java:94` | Real version handles `ResponseEntity<SseEmitter>`, reactive types, view fragments |
| `SseHandler.send()` (Jackson direct) | `DefaultSseEmitterHandler.sendInternal()` | `ResponseBodyEmitterReturnValueHandler.java:303` | Real version iterates message converters with `StreamingServletServerHttpResponse` |
| `request.isAsyncStarted()` | `asyncManager.isConcurrentHandlingStarted()` | `DispatcherServlet.java:965` | Real version uses `WebAsyncManager` abstraction with state machine and interceptor hooks |
| `asyncContext.complete()` | `deferredResult.setResult(null)` → async dispatch | `WebAsyncManager.java:491` | Real version re-dispatches through `DispatcherServlet` for final cleanup |
| `wrapper.setAsyncSupported(true)` | `ServletRegistrationBean` default | `ServletRegistrationBean.java` | Always enabled by default in Spring Boot |

---

## 22.8 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/sse/SseEmitter.java` [NEW]

```java
package com.simplespringmvc.sse;

import com.simplespringmvc.http.MediaType;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Collection;
import java.util.concurrent.locks.ReentrantLock;
import java.util.function.Consumer;

public class SseEmitter {

    private final Long timeout;
    private Handler handler;
    private final List<DataWithMediaType> earlySendAttempts = new ArrayList<>(8);
    private boolean complete;
    private Throwable failure;
    private final List<Runnable> timeoutCallbacks = new ArrayList<>(1);
    private final List<Consumer<Throwable>> errorCallbacks = new ArrayList<>(1);
    private final List<Runnable> completionCallbacks = new ArrayList<>(1);

    protected final ReentrantLock writeLock = new ReentrantLock();

    public SseEmitter() {
        this(null);
    }

    public SseEmitter(Long timeout) {
        this.timeout = timeout;
    }

    public Long getTimeout() {
        return timeout;
    }

    public void send(Object data) throws IOException {
        send(event().data(data));
    }

    public void send(Object data, MediaType mediaType) throws IOException {
        send(event().data(data, mediaType));
    }

    public void send(SseEventBuilder builder) throws IOException {
        Collection<DataWithMediaType> dataToSend = builder.build();
        writeLock.lock();
        try {
            if (complete) {
                throw new IllegalStateException("SseEmitter has already completed");
            }
            if (handler != null) {
                handler.send(dataToSend);
            } else {
                earlySendAttempts.addAll(dataToSend);
            }
        } catch (IOException | IllegalStateException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new IllegalStateException("Failed to send SSE event", ex);
        } finally {
            writeLock.unlock();
        }
    }

    public void complete() {
        writeLock.lock();
        try {
            complete = true;
            if (handler != null) {
                handler.complete();
            }
        } finally {
            writeLock.unlock();
        }
    }

    public void completeWithError(Throwable ex) {
        writeLock.lock();
        try {
            complete = true;
            failure = ex;
            if (handler != null) {
                handler.completeWithError(ex);
            }
        } finally {
            writeLock.unlock();
        }
    }

    public void onTimeout(Runnable callback) {
        writeLock.lock();
        try {
            timeoutCallbacks.add(callback);
            if (handler != null) {
                handler.onTimeout(callback);
            }
        } finally {
            writeLock.unlock();
        }
    }

    public void onError(Consumer<Throwable> callback) {
        writeLock.lock();
        try {
            errorCallbacks.add(callback);
            if (handler != null) {
                handler.onError(callback);
            }
        } finally {
            writeLock.unlock();
        }
    }

    public void onCompletion(Runnable callback) {
        writeLock.lock();
        try {
            completionCallbacks.add(callback);
            if (handler != null) {
                handler.onCompletion(callback);
            }
        } finally {
            writeLock.unlock();
        }
    }

    public void initialize(Handler handler) throws IOException {
        writeLock.lock();
        try {
            this.handler = handler;

            if (!earlySendAttempts.isEmpty()) {
                handler.send(List.copyOf(earlySendAttempts));
                earlySendAttempts.clear();
            }

            if (complete) {
                if (failure != null) {
                    handler.completeWithError(failure);
                } else {
                    handler.complete();
                }
                return;
            }

            for (Runnable cb : timeoutCallbacks) {
                handler.onTimeout(cb);
            }
            for (Consumer<Throwable> cb : errorCallbacks) {
                handler.onError(cb);
            }
            for (Runnable cb : completionCallbacks) {
                handler.onCompletion(cb);
            }
        } finally {
            writeLock.unlock();
        }
    }

    public void initializeWithError(Throwable ex) {
        writeLock.lock();
        try {
            complete = true;
            failure = ex;
            earlySendAttempts.clear();
            for (Consumer<Throwable> cb : errorCallbacks) {
                cb.accept(ex);
            }
        } finally {
            writeLock.unlock();
        }
    }

    public static SseEventBuilder event() {
        return new SseEventBuilderImpl();
    }

    public interface Handler {
        void send(Collection<DataWithMediaType> items) throws IOException;
        void complete();
        void completeWithError(Throwable failure);
        void onTimeout(Runnable callback);
        void onError(Consumer<Throwable> callback);
        void onCompletion(Runnable callback);
    }

    public interface SseEventBuilder {
        SseEventBuilder id(String id);
        SseEventBuilder name(String eventName);
        SseEventBuilder reconnectTime(long millis);
        SseEventBuilder comment(String comment);
        SseEventBuilder data(Object data);
        SseEventBuilder data(Object data, MediaType mediaType);
        Collection<DataWithMediaType> build();
    }

    public record DataWithMediaType(Object data, MediaType mediaType) {}

    private static class SseEventBuilderImpl implements SseEventBuilder {

        private final List<DataWithMediaType> dataToSend = new ArrayList<>(4);
        private StringBuilder sb;

        private void append(String text) {
            if (sb == null) {
                sb = new StringBuilder();
            }
            sb.append(text);
        }

        private void saveAppendedText() {
            if (sb != null) {
                dataToSend.add(new DataWithMediaType(sb.toString(), MediaType.TEXT_PLAIN));
                sb = null;
            }
        }

        @Override
        public SseEventBuilder id(String id) {
            append("id:");
            append(id);
            append("\n");
            return this;
        }

        @Override
        public SseEventBuilder name(String eventName) {
            append("event:");
            append(eventName);
            append("\n");
            return this;
        }

        @Override
        public SseEventBuilder reconnectTime(long millis) {
            append("retry:");
            append(String.valueOf(millis));
            append("\n");
            return this;
        }

        @Override
        public SseEventBuilder comment(String comment) {
            append(":");
            append(comment);
            append("\n");
            return this;
        }

        @Override
        public SseEventBuilder data(Object data) {
            return data(data, null);
        }

        @Override
        public SseEventBuilder data(Object data, MediaType mediaType) {
            append("data:");
            saveAppendedText();

            if (data instanceof String str) {
                data = str.replace("\n", "\ndata:");
            }

            dataToSend.add(new DataWithMediaType(data, mediaType));
            append("\n");
            return this;
        }

        @Override
        public Collection<DataWithMediaType> build() {
            append("\n");
            saveAppendedText();
            return dataToSend;
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SseEmitterReturnValueHandler.java` [NEW]

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

public class SseEmitterReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(HandlerMethod handlerMethod) {
        return SseEmitter.class.isAssignableFrom(handlerMethod.getMethod().getReturnType());
    }

    @Override
    public ModelAndView handleReturnValue(Object returnValue, HandlerMethod handlerMethod,
                                          HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        if (returnValue == null) {
            return null;
        }

        SseEmitter emitter = (SseEmitter) returnValue;

        response.setContentType("text/event-stream");
        response.setCharacterEncoding("UTF-8");
        response.setHeader("Cache-Control", "no-cache");
        response.setHeader("Connection", "keep-alive");

        AsyncContext asyncContext;
        try {
            asyncContext = request.startAsync(request, response);
        } catch (IllegalStateException ex) {
            emitter.initializeWithError(ex);
            throw ex;
        }

        if (emitter.getTimeout() != null) {
            asyncContext.setTimeout(emitter.getTimeout());
        } else {
            asyncContext.setTimeout(0);
        }

        SseHandler sseHandler = new SseHandler(response, asyncContext);

        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onTimeout(AsyncEvent event) {
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
            }
        });

        try {
            emitter.initialize(sseHandler);
        } catch (Throwable ex) {
            emitter.initializeWithError(ex);
            throw ex;
        }

        return null;
    }

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

        @Override
        public void send(Collection<SseEmitter.DataWithMediaType> items) throws IOException {
            PrintWriter writer = response.getWriter();
            for (SseEmitter.DataWithMediaType item : items) {
                if (item.data() instanceof String str) {
                    writer.write(str);
                } else {
                    writer.write(objectMapper.writeValueAsString(item.data()));
                }
            }
            writer.flush();
            if (writer.checkError()) {
                throw new IOException("Client disconnected");
            }
        }

        @Override
        public void complete() {
            if (asyncComplete) return;
            asyncComplete = true;
            try {
                response.getWriter().flush();
            } catch (IOException ignored) {
            }
            asyncContext.complete();
        }

        @Override
        public void completeWithError(Throwable failure) {
            if (asyncComplete) return;
            asyncComplete = true;
            asyncContext.complete();
        }

        @Override
        public void onTimeout(Runnable callback) { timeoutCallbacks.add(callback); }

        @Override
        public void onError(Consumer<Throwable> callback) { errorCallbacks.add(callback); }

        @Override
        public void onCompletion(Runnable callback) { completionCallbacks.add(callback); }

        void fireTimeoutCallbacks() {
            for (Runnable cb : timeoutCallbacks) cb.run();
        }

        void fireErrorCallbacks(Throwable cause) {
            for (Consumer<Throwable> cb : errorCallbacks) cb.accept(cause);
        }

        void fireCompletionCallbacks() {
            for (Runnable cb : completionCallbacks) cb.run();
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java` [MODIFIED]

Change: Added `wrapper.setAsyncSupported(true)` after servlet registration.

See the full file in section 22.1 (Seam 1). The only change is replacing `Tomcat.addServlet(...)` with `var wrapper = Tomcat.addServlet(...)` and adding the `setAsyncSupported(true)` call.

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

Change: Added `request.isAsyncStarted()` check after handler invocation in `doDispatch()`.

See section 22.1 (Seam 2) for the diff. The rest of the file is unchanged.

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

Change: Added `SseEmitterReturnValueHandler` as the first return value handler in both constructors.

See section 22.1 (Seam 3) for the diff. The rest of the file is unchanged.

### Test Code

#### File: `src/test/java/com/simplespringmvc/sse/SseEmitterTest.java` [NEW]

See the full test file containing 20 tests covering: event builder format, early send buffering, completion lifecycle, callback forwarding, timeout configuration, error initialization, and concurrent thread safety.

#### File: `src/test/java/com/simplespringmvc/adapter/SseEmitterReturnValueHandlerTest.java` [NEW]

See the full test file containing 9 tests covering: return type detection, SSE content type, async processing, timeout configuration, string/JSON/structured event writing, and early send flushing.

#### File: `src/test/java/com/simplespringmvc/integration/SseIntegrationTest.java` [NEW]

See the full test file containing 7 integration tests with real Tomcat: simple events, named events with ID, JSON serialization, comments, @RestController support, event ordering, and retry field.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Server-Sent Events (SSE)** | HTTP-based protocol for server-to-client streaming over a long-lived connection (`text/event-stream`) |
| **Two-phase lifecycle** | Emitter buffers sends before initialization, flushes them when the handler is wired — solves the race between controller return and async setup |
| **DataWithMediaType interleaving** | SSE protocol text (String/TEXT_PLAIN) and data payloads (objects/JSON) are separate entries, allowing different serialization strategies per chunk |
| **AsyncContext** | Servlet API for keeping a connection open beyond the normal request/response cycle — the foundation for SSE, long polling, and streaming |
| **ReentrantLock** | Explicit lock (not `synchronized`) for thread-safe sends — allows the same thread to re-enter the lock, which is important when `send()` calls `handler.send()` under the lock |
| **Return value handler ordering** | `SseEmitterReturnValueHandler` must come before `ResponseBodyReturnValueHandler` to prevent `@RestController` from intercepting `SseEmitter` returns |

**Next: Chapter 23 — @HttpExchange / HTTP Interface Clients** — Define HTTP clients as annotated interfaces and have the framework generate proxy implementations that make real HTTP calls — the declarative client-side counterpart to @RequestMapping.
