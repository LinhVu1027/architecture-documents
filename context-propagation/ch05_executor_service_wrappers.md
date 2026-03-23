# Chapter 5: Executor Service Wrappers

> Line references based on commit `be901c1` of the Context Propagation repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have `ContextSnapshotFactory` to capture ThreadLocal values and `ContextSnapshot` to restore them via try-with-resources — but callers must manually capture a snapshot, wrap the task, and restore it for every thread pool submission | Every `executor.submit(task)` call requires boilerplate: `factory.captureAll().wrap(task)` — easy to forget, leading to silent context loss across threads | Build `ContextExecutorService` and `ContextScheduledExecutorService` wrappers that automatically capture and restore context for every submitted task |

---

## 5.1 The Integration Point

This feature wraps `ExecutorService` — the standard JDK interface for submitting tasks to thread pools. The integration point is the **`submit()`/`execute()` call boundary** — the exact moment when a task crosses from the submitting thread to the worker thread. That's where context is lost, and that's where we intercept.

The wrapper delegates every method to the underlying executor, but intercepts each task submission to apply `capture().wrap(task)`:

**New file:** `src/main/java/dev/linhvu/context/ContextExecutorService.java`

```java
public class ContextExecutorService<EXECUTOR extends ExecutorService> implements ExecutorService {

    private final EXECUTOR executorService;
    private final Supplier<ContextSnapshot> contextSnapshot;

    protected ContextExecutorService(EXECUTOR executorService, Supplier<ContextSnapshot> contextSnapshot) {
        this.executorService = executorService;
        this.contextSnapshot = contextSnapshot;
    }

    protected ContextSnapshot capture() {
        return this.contextSnapshot.get();
    }

    @Override
    public void execute(Runnable command) {
        this.executorService.execute(capture().wrap(command));  // <-- The integration point
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return this.executorService.submit(capture().wrap(task));
    }

    // ... all other ExecutorService methods follow the same pattern

    public static ExecutorService wrap(ExecutorService service, ContextSnapshotFactory factory) {
        return new ContextExecutorService<>(service, factory::captureAll);
    }
}
```

Two key decisions here:

1. **`Supplier<ContextSnapshot>` (lazy capture) instead of storing a single snapshot.** The supplier is called on every `submit()`/`execute()` — each task gets a fresh snapshot of the submitting thread's current ThreadLocal values. If we captured once at wrap-time, every task would carry stale context.
2. **Generic `<EXECUTOR extends ExecutorService>` type parameter.** This allows `ContextScheduledExecutorService` to extend this class with the more specific `ScheduledExecutorService` type, reusing all the base delegation logic while adding scheduling methods.

This connects **ContextSnapshotFactory** (the capture API from ch04) to **Java's ExecutorService** (the thread boundary). To make it work, we need to build:
- `ContextExecutorService` — wraps all `ExecutorService` methods with capture-and-wrap
- `ContextScheduledExecutorService` — extends the above for `ScheduledExecutorService`'s four scheduling methods

## 5.2 ContextExecutorService

The full implementation delegates every `ExecutorService` method. The pattern is always the same: `capture().wrap(task)`.

**New file:** `src/main/java/dev/linhvu/context/ContextExecutorService.java`

```java
package dev.linhvu.context;

import java.util.Collection;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.function.Supplier;
import java.util.stream.Collectors;

public class ContextExecutorService<EXECUTOR extends ExecutorService> implements ExecutorService {

    private final EXECUTOR executorService;

    private final Supplier<ContextSnapshot> contextSnapshot;

    protected ContextExecutorService(EXECUTOR executorService, Supplier<ContextSnapshot> contextSnapshot) {
        this.executorService = executorService;
        this.contextSnapshot = contextSnapshot;
    }

    protected EXECUTOR getExecutorService() {
        return this.executorService;
    }

    protected ContextSnapshot capture() {
        return this.contextSnapshot.get();
    }

    @Override
    public void execute(Runnable command) {
        this.executorService.execute(capture().wrap(command));
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return this.executorService.submit(capture().wrap(task));
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
        return this.executorService.submit(capture().wrap(task), result);
    }

    @Override
    public Future<?> submit(Runnable task) {
        return this.executorService.submit(capture().wrap(task));
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAll(wrapped);
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
            throws InterruptedException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAll(wrapped, timeout, unit);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
            throws InterruptedException, ExecutionException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAny(wrapped);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAny(wrapped, timeout, unit);
    }

    @Override
    public void shutdown() {
        this.executorService.shutdown();
    }

    @Override
    public List<Runnable> shutdownNow() {
        return this.executorService.shutdownNow();
    }

    @Override
    public boolean isShutdown() {
        return this.executorService.isShutdown();
    }

    @Override
    public boolean isTerminated() {
        return this.executorService.isTerminated();
    }

    @Override
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return this.executorService.awaitTermination(timeout, unit);
    }

    public static ExecutorService wrap(ExecutorService service, ContextSnapshotFactory factory) {
        return new ContextExecutorService<>(service, factory::captureAll);
    }

}
```

Notice the `invokeAll()` and `invokeAny()` pattern: a single `capture()` call produces one snapshot, then `capture()::wrap` is used as a method reference to wrap each `Callable` in the collection. All tasks in a batch share the same snapshot — they all carry the same context from the submitting thread.

## 5.3 ContextScheduledExecutorService

The scheduled variant extends `ContextExecutorService<ScheduledExecutorService>` and adds the four scheduling methods from `ScheduledExecutorService`. Each one follows the same `capture().wrap(task)` pattern.

**New file:** `src/main/java/dev/linhvu/context/ContextScheduledExecutorService.java`

```java
package dev.linhvu.context;

import java.util.concurrent.Callable;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

public final class ContextScheduledExecutorService extends ContextExecutorService<ScheduledExecutorService>
        implements ScheduledExecutorService {

    private ContextScheduledExecutorService(ScheduledExecutorService service,
            ContextSnapshotFactory factory) {
        super(service, factory::captureAll);
    }

    @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        return getExecutorService().schedule(capture().wrap(command), delay, unit);
    }

    @Override
    public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
        return getExecutorService().schedule(capture().wrap(callable), delay, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
        return getExecutorService().scheduleAtFixedRate(capture().wrap(command), initialDelay, period, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
        return getExecutorService().scheduleWithFixedDelay(capture().wrap(command), initialDelay, delay, unit);
    }

    public static ScheduledExecutorService wrap(ScheduledExecutorService service, ContextSnapshotFactory factory) {
        return new ContextScheduledExecutorService(service, factory);
    }

}
```

Key observations:
- **`final` class** — there's no reason to further extend the scheduled variant
- **Private constructor** — the only way to create instances is via the static `wrap()` factory
- **`getExecutorService()`** — uses the parent's accessor to get the typed `ScheduledExecutorService` (this is why the generic type parameter exists)
- For recurring tasks (`scheduleAtFixedRate`, `scheduleWithFixedDelay`), `capture()` is called once at scheduling time — every recurring execution replays the same snapshot

## 5.4 Try It Yourself

<details>
<summary>Challenge 1: Why does each submit() call capture() instead of storing a single snapshot?</summary>

Consider this scenario:

```java
ExecutorService executor = ContextExecutorService.wrap(pool, factory);

USER.set("alice");
executor.submit(() -> System.out.println(USER.get()));  // Should print "alice"

USER.set("bob");
executor.submit(() -> System.out.println(USER.get()));  // Should print "bob"
```

If we captured once at wrap-time, both tasks would print "alice" (or whatever was current when `wrap()` was called). By calling `capture()` inside each `submit()`, each task gets the context that was active at its submission time.

This is why the constructor takes `Supplier<ContextSnapshot>` (lazy) instead of `ContextSnapshot` (eager).

</details>

<details>
<summary>Challenge 2: Implement a simplified ContextExecutorService that only handles submit(Runnable)</summary>

Try implementing a minimal version that only wraps `submit(Runnable)`:

```java
public class MinimalContextExecutorService implements ExecutorService {

    private final ExecutorService delegate;
    private final ContextSnapshotFactory factory;

    public MinimalContextExecutorService(ExecutorService delegate, ContextSnapshotFactory factory) {
        this.delegate = delegate;
        this.factory = factory;
    }

    @Override
    public Future<?> submit(Runnable task) {
        ContextSnapshot snapshot = factory.captureAll();
        return delegate.submit(snapshot.wrap(task));
    }

    // ... other methods just delegate without wrapping
}
```

Notice how `factory.captureAll()` is called inside `submit()`, not in the constructor. That's the capture-at-submission-time pattern in its simplest form.

</details>

<details>
<summary>Challenge 3: What happens to the worker thread's own ThreadLocals?</summary>

The worker thread might have its own ThreadLocal values set before our task runs. The `snapshot.wrap(task)` call uses `setThreadLocals()` which:

1. Saves the worker thread's current ThreadLocal values (the "previous values")
2. Sets the snapshot's values (from the submitting thread)
3. Runs the task
4. Restores the worker thread's previous values via `Scope.close()`

This means the worker thread's own state is preserved — our context propagation is non-destructive. The test `shouldRestoreWorkerThreadLocals_AfterTaskCompletes` verifies this explicitly.

</details>

## 5.5 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/context/ContextExecutorServiceTest.java`

```java
class ContextExecutorServiceTest {

    @Test
    void shouldPropagateThreadLocal_WhenExecuteRunnable()    // execute() propagates
    void shouldPropagateThreadLocal_WhenSubmitRunnable()      // submit(Runnable) propagates
    void shouldPropagateThreadLocal_WhenSubmitCallable()      // submit(Callable) propagates
    void shouldPropagateThreadLocal_WhenSubmitRunnableWithResult()  // submit(Runnable, T) propagates
    void shouldPropagateThreadLocal_WhenInvokeAll()           // invokeAll() propagates to all tasks
    void shouldPropagateThreadLocal_WhenInvokeAny()           // invokeAny() propagates
    void shouldCaptureAtSubmissionTime_NotAtWrapTime()        // proves lazy capture
    void shouldRestoreWorkerThreadLocals_AfterTaskCompletes() // proves non-destructive
    void shouldDelegateShutdownMethods()                      // lifecycle delegation works
}
```

**New file:** `src/test/java/dev/linhvu/context/ContextScheduledExecutorServiceTest.java`

```java
class ContextScheduledExecutorServiceTest {

    @Test
    void shouldPropagateThreadLocal_WhenScheduleRunnable()         // schedule(Runnable) propagates
    void shouldPropagateThreadLocal_WhenScheduleCallable()         // schedule(Callable) propagates
    void shouldPropagateThreadLocal_WhenScheduleAtFixedRate()      // recurring tasks propagate
    void shouldPropagateThreadLocal_WhenScheduleWithFixedDelay()   // recurring tasks propagate
    void shouldPropagateThreadLocal_WhenUsingSubmitFromParentClass() // inherited methods work
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/context/ExecutorServiceIntegrationTest.java`

```java
class ExecutorServiceIntegrationTest {

    @Test
    void shouldPropagateMultipleThreadLocals_WhenSubmittingTask()    // full stack with 2 ThreadLocals
    void shouldPropagatePartialContext_WhenOnlySomeThreadLocalsSet() // only set values propagate
    void shouldPropagateWithClearMissing_WhenEnabled()               // clearMissing clears stale values
    void shouldPropagateWithKeyFilter_WhenPredicateConfigured()      // captureKeyPredicate filters keys
    void shouldWorkWithScheduledExecutor_AcrossFullStack()           // scheduled variant + full stack
}
```

**Run:** `./gradlew test` — expected: all 9 test classes pass (6 prior + 3 new)

---

## 5.6 Why This Works

The Decorator pattern is the natural fit here: `ContextExecutorService` IS-an `ExecutorService` (it implements the interface) and HAS-an `ExecutorService` (it delegates to the wrapped instance). Callers don't know — and don't care — that context propagation is happening. They just use `executor.submit(task)` as usual.

> ★ **Insight** -------------------------------------------
> - **Capture-at-submission-time is the critical design choice.** The `Supplier<ContextSnapshot>` field (called via `capture()` in each method) ensures each task carries the context that was active when it was submitted. If you captured once at wrap-time, all tasks would see stale context — a subtle, hard-to-debug concurrency bug. The real framework uses the same `Supplier` approach for exactly this reason (`ContextExecutorService.java:125`).
> - **The `wrap()` method from ch03 does the heavy lifting.** `ContextSnapshot.wrap(Runnable)` sets ThreadLocals before the task runs and restores them after — the executor wrappers just invoke `capture().wrap(task)` at the right moment. This layering means the snapshot logic (ch03) and the executor wrapping (ch05) evolve independently.
> - **The generic type parameter isn't just for show.** `<EXECUTOR extends ExecutorService>` lets `ContextScheduledExecutorService` extend the base class with `ScheduledExecutorService` as the type parameter, getting a type-safe `getExecutorService()` that returns the scheduled variant — no casting needed. Without generics, the subclass would need to cast on every scheduling method call.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Non-destructive context propagation.** The worker thread's own ThreadLocal values are saved before the snapshot is applied and restored after the task completes (via `Scope.close()`). This means context propagation is transparent to the worker thread — it doesn't permanently overwrite any existing state. This is essential for thread pool reuse where the same thread serves many tasks.
> - **When NOT to use this pattern:** If your executor runs very short-lived tasks at extremely high throughput, the per-task snapshot capture adds overhead (one `HashMap` allocation + iteration over registered accessors per submission). In such cases, consider batching tasks or using the lower-level `ContextSnapshot.wrap()` only where needed, rather than wrapping the entire executor.
> -----------------------------------------------------------

## 5.7 What We Enhanced

| Aspect | Before (ch04) | Current (ch05) | Real Framework |
|--------|---------------|-----------------|----------------|
| **Thread pool usage** | Callers must manually call `factory.captureAll().wrap(task)` for every submission | `ContextExecutorService.wrap(executor, factory)` — automatic propagation for all tasks | Same approach — `ContextExecutorService.java:141` |
| **Scheduled tasks** | No support for scheduled/recurring context propagation | `ContextScheduledExecutorService` wraps all 4 scheduling methods | Same approach — `ContextScheduledExecutorService.java:84` |
| **API surface** | `ContextSnapshot.wrap(Runnable/Callable)` from ch03 required manual use | `wrap()` methods are called automatically by the executor wrapper — callers just use `submit()`/`execute()` as normal | Same layering — executor calls `capture().wrap()` |

## 5.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `ContextExecutorService` | `ContextExecutorService` | `ContextExecutorService.java:38` | Real version also has `wrap(service, Supplier<ContextSnapshot>)` for non-ThreadLocal contexts and a deprecated `wrap(service)` overload |
| `ContextScheduledExecutorService` | `ContextScheduledExecutorService` | `ContextScheduledExecutorService.java:35` | Real version also has `wrap(service, Supplier)` overload and deprecated `wrap(service)` |
| `capture()` | `capture()` | `ContextExecutorService.java:125` | Identical implementation — `return this.contextSnapshot.get()` |
| `wrap(service, factory)` | `wrap(service, factory)` | `ContextExecutorService.java:141` | Identical — `new ContextExecutorService<>(service, factory::captureAll)` |
| N/A | `wrap(service, Supplier)` | `ContextExecutorService.java:167` | Accepts any `Supplier<ContextSnapshot>` — supports reactive context (Reactor `Context`) via `ContextAccessor`, which we omit |
| N/A | `wrap(service)` (deprecated) | `ContextExecutorService.java:183` | Uses global `ContextRegistry.getInstance()` — deprecated in favor of explicit factory |

## 5.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/context/ContextExecutorService.java` [NEW]

```java
package dev.linhvu.context;

import java.util.Collection;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.function.Supplier;
import java.util.stream.Collectors;

/**
 * Wraps an {@link ExecutorService} to automatically capture and restore
 * {@link ThreadLocal} context for every submitted task.
 *
 * <p>A fresh {@link ContextSnapshot} is captured at <b>task submission time</b>
 * (not at wrapper creation time). This ensures each task carries the context
 * that was active when it was submitted, not when the executor was wrapped.
 *
 * <p>Usage:
 * <pre>{@code
 * ContextSnapshotFactory factory = ContextSnapshotFactory.builder().build();
 * ExecutorService executor = ContextExecutorService.wrap(
 *     Executors.newFixedThreadPool(4), factory);
 *
 * // ThreadLocal values are automatically propagated to the worker thread
 * executor.submit(() -> {
 *     // ThreadLocals from the submitting thread are available here
 * });
 * }</pre>
 *
 * @param <EXECUTOR> the type of the wrapped ExecutorService
 * @see ContextSnapshot
 * @see ContextSnapshotFactory
 */
public class ContextExecutorService<EXECUTOR extends ExecutorService> implements ExecutorService {

    private final EXECUTOR executorService;

    private final Supplier<ContextSnapshot> contextSnapshot;

    protected ContextExecutorService(EXECUTOR executorService, Supplier<ContextSnapshot> contextSnapshot) {
        this.executorService = executorService;
        this.contextSnapshot = contextSnapshot;
    }

    protected EXECUTOR getExecutorService() {
        return this.executorService;
    }

    /**
     * Capture a fresh snapshot from the current thread's ThreadLocal values.
     * Called at the point of task submission, not at wrapper creation.
     */
    protected ContextSnapshot capture() {
        return this.contextSnapshot.get();
    }

    @Override
    public void execute(Runnable command) {
        this.executorService.execute(capture().wrap(command));
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return this.executorService.submit(capture().wrap(task));
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
        return this.executorService.submit(capture().wrap(task), result);
    }

    @Override
    public Future<?> submit(Runnable task) {
        return this.executorService.submit(capture().wrap(task));
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAll(wrapped);
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
            throws InterruptedException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAll(wrapped, timeout, unit);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
            throws InterruptedException, ExecutionException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAny(wrapped);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        List<Callable<T>> wrapped = tasks.stream().map(capture()::wrap).collect(Collectors.toList());
        return this.executorService.invokeAny(wrapped, timeout, unit);
    }

    @Override
    public void shutdown() {
        this.executorService.shutdown();
    }

    @Override
    public List<Runnable> shutdownNow() {
        return this.executorService.shutdownNow();
    }

    @Override
    public boolean isShutdown() {
        return this.executorService.isShutdown();
    }

    @Override
    public boolean isTerminated() {
        return this.executorService.isTerminated();
    }

    @Override
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return this.executorService.awaitTermination(timeout, unit);
    }

    /**
     * Wrap the given {@link ExecutorService} to propagate context to every
     * executed task through the given {@link ContextSnapshotFactory}.
     *
     * @param service the executor service to wrap
     * @param factory the factory for capturing snapshots at task submission time
     * @return a context-propagating executor service wrapper
     */
    public static ExecutorService wrap(ExecutorService service, ContextSnapshotFactory factory) {
        return new ContextExecutorService<>(service, factory::captureAll);
    }

}
```

#### File: `src/main/java/dev/linhvu/context/ContextScheduledExecutorService.java` [NEW]

```java
package dev.linhvu.context;

import java.util.concurrent.Callable;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

/**
 * Wraps a {@link ScheduledExecutorService} to automatically capture and restore
 * {@link ThreadLocal} context for every submitted and scheduled task.
 *
 * <p>Extends {@link ContextExecutorService} to inherit context propagation for
 * all standard {@code ExecutorService} methods ({@code submit}, {@code execute},
 * {@code invokeAll}, {@code invokeAny}), and adds propagation for the four
 * scheduling methods: {@code schedule(Runnable)}, {@code schedule(Callable)},
 * {@code scheduleAtFixedRate}, and {@code scheduleWithFixedDelay}.
 *
 * <p>Like the parent class, a fresh snapshot is captured at task submission time.
 * For recurring tasks ({@code scheduleAtFixedRate}, {@code scheduleWithFixedDelay}),
 * the snapshot is captured once when the task is scheduled — each execution
 * restores the same snapshot from the scheduling thread.
 *
 * @see ContextExecutorService
 * @see ContextSnapshot
 * @see ContextSnapshotFactory
 */
public final class ContextScheduledExecutorService extends ContextExecutorService<ScheduledExecutorService>
        implements ScheduledExecutorService {

    private ContextScheduledExecutorService(ScheduledExecutorService service,
            ContextSnapshotFactory factory) {
        super(service, factory::captureAll);
    }

    @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        return getExecutorService().schedule(capture().wrap(command), delay, unit);
    }

    @Override
    public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
        return getExecutorService().schedule(capture().wrap(callable), delay, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
        return getExecutorService().scheduleAtFixedRate(capture().wrap(command), initialDelay, period, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
        return getExecutorService().scheduleWithFixedDelay(capture().wrap(command), initialDelay, delay, unit);
    }

    /**
     * Wrap the given {@link ScheduledExecutorService} to propagate context to
     * every executed and scheduled task through the given {@link ContextSnapshotFactory}.
     *
     * @param service the scheduled executor service to wrap
     * @param factory the factory for capturing snapshots at task submission time
     * @return a context-propagating scheduled executor service wrapper
     */
    public static ScheduledExecutorService wrap(ScheduledExecutorService service, ContextSnapshotFactory factory) {
        return new ContextScheduledExecutorService(service, factory);
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/context/ContextExecutorServiceTest.java` [NEW]

```java
package dev.linhvu.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class ContextExecutorServiceTest {

    private static final ThreadLocal<String> USER = new ThreadLocal<>();

    private ContextRegistry registry;

    private ContextSnapshotFactory factory;

    private ExecutorService rawExecutor;

    private ExecutorService contextExecutor;

    @BeforeEach
    void setUp() {
        registry = new ContextRegistry();
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", USER));
        factory = ContextSnapshotFactory.builder().contextRegistry(registry).build();
        rawExecutor = Executors.newSingleThreadExecutor();
        contextExecutor = ContextExecutorService.wrap(rawExecutor, factory);
    }

    @AfterEach
    void tearDown() throws InterruptedException {
        contextExecutor.shutdown();
        contextExecutor.awaitTermination(5, TimeUnit.SECONDS);
        USER.remove();
    }

    @Test
    void shouldPropagateThreadLocal_WhenExecuteRunnable() throws Exception {
        USER.set("alice");
        AtomicReference<String> captured = new AtomicReference<>();

        contextExecutor.execute(() -> captured.set(USER.get()));

        // Wait for task to complete
        contextExecutor.submit(() -> {}).get();
        assertThat(captured.get()).isEqualTo("alice");
    }

    @Test
    void shouldPropagateThreadLocal_WhenSubmitRunnable() throws Exception {
        USER.set("bob");
        AtomicReference<String> captured = new AtomicReference<>();

        contextExecutor.submit(() -> captured.set(USER.get())).get();

        assertThat(captured.get()).isEqualTo("bob");
    }

    @Test
    void shouldPropagateThreadLocal_WhenSubmitCallable() throws Exception {
        USER.set("charlie");

        String result = contextExecutor.submit(() -> USER.get()).get();

        assertThat(result).isEqualTo("charlie");
    }

    @Test
    void shouldPropagateThreadLocal_WhenSubmitRunnableWithResult() throws Exception {
        USER.set("dave");
        AtomicReference<String> captured = new AtomicReference<>();

        String result = contextExecutor.submit(() -> captured.set(USER.get()), "done").get();

        assertThat(result).isEqualTo("done");
        assertThat(captured.get()).isEqualTo("dave");
    }

    @Test
    void shouldPropagateThreadLocal_WhenInvokeAll() throws Exception {
        USER.set("eve");

        List<Callable<String>> tasks = Arrays.asList(
                () -> USER.get(),
                () -> USER.get() + "-copy"
        );

        List<Future<String>> futures = contextExecutor.invokeAll(tasks);

        assertThat(futures.get(0).get()).isEqualTo("eve");
        assertThat(futures.get(1).get()).isEqualTo("eve-copy");
    }

    @Test
    void shouldPropagateThreadLocal_WhenInvokeAny() throws Exception {
        USER.set("frank");

        List<Callable<String>> tasks = Arrays.asList(
                () -> USER.get()
        );

        String result = contextExecutor.invokeAny(tasks);

        assertThat(result).isEqualTo("frank");
    }

    @Test
    void shouldCaptureAtSubmissionTime_NotAtWrapTime() throws Exception {
        // The snapshot is captured when submit() is called, not when wrap() was called
        USER.set("before");
        AtomicReference<String> captured1 = new AtomicReference<>();
        AtomicReference<String> captured2 = new AtomicReference<>();

        contextExecutor.submit(() -> captured1.set(USER.get())).get();

        USER.set("after");
        contextExecutor.submit(() -> captured2.set(USER.get())).get();

        assertThat(captured1.get()).isEqualTo("before");
        assertThat(captured2.get()).isEqualTo("after");
    }

    @Test
    void shouldRestoreWorkerThreadLocals_AfterTaskCompletes() throws Exception {
        USER.set("main-thread");
        AtomicReference<String> workerBefore = new AtomicReference<>();
        AtomicReference<String> workerAfter = new AtomicReference<>();

        // First: set a value on the worker thread directly
        rawExecutor.submit(() -> USER.set("worker-original")).get();

        // Capture what the worker thread has before and after our context-propagating task
        // We use the raw executor to check state, then context executor, then raw again
        rawExecutor.submit(() -> workerBefore.set(USER.get())).get();
        contextExecutor.submit(() -> USER.get()).get();
        rawExecutor.submit(() -> workerAfter.set(USER.get())).get();

        assertThat(workerBefore.get()).isEqualTo("worker-original");
        assertThat(workerAfter.get()).isEqualTo("worker-original");
    }

    @Test
    void shouldDelegateShutdownMethods() throws Exception {
        assertThat(contextExecutor.isShutdown()).isFalse();
        assertThat(contextExecutor.isTerminated()).isFalse();

        contextExecutor.shutdown();
        contextExecutor.awaitTermination(5, TimeUnit.SECONDS);

        assertThat(contextExecutor.isShutdown()).isTrue();
        assertThat(contextExecutor.isTerminated()).isTrue();
    }

}
```

#### File: `src/test/java/dev/linhvu/context/ContextScheduledExecutorServiceTest.java` [NEW]

```java
package dev.linhvu.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class ContextScheduledExecutorServiceTest {

    private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();

    private ContextRegistry registry;

    private ContextSnapshotFactory factory;

    private ScheduledExecutorService rawScheduler;

    private ScheduledExecutorService contextScheduler;

    @BeforeEach
    void setUp() {
        registry = new ContextRegistry();
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("traceId", TRACE_ID));
        factory = ContextSnapshotFactory.builder().contextRegistry(registry).build();
        rawScheduler = Executors.newScheduledThreadPool(1);
        contextScheduler = ContextScheduledExecutorService.wrap(rawScheduler, factory);
    }

    @AfterEach
    void tearDown() throws InterruptedException {
        contextScheduler.shutdown();
        contextScheduler.awaitTermination(5, TimeUnit.SECONDS);
        TRACE_ID.remove();
    }

    @Test
    void shouldPropagateThreadLocal_WhenScheduleRunnable() throws Exception {
        TRACE_ID.set("trace-123");
        AtomicReference<String> captured = new AtomicReference<>();

        contextScheduler.schedule(
                () -> captured.set(TRACE_ID.get()),
                10, TimeUnit.MILLISECONDS
        ).get();

        assertThat(captured.get()).isEqualTo("trace-123");
    }

    @Test
    void shouldPropagateThreadLocal_WhenScheduleCallable() throws Exception {
        TRACE_ID.set("trace-456");

        String result = contextScheduler.schedule(
                () -> TRACE_ID.get(),
                10, TimeUnit.MILLISECONDS
        ).get();

        assertThat(result).isEqualTo("trace-456");
    }

    @Test
    void shouldPropagateThreadLocal_WhenScheduleAtFixedRate() throws Exception {
        TRACE_ID.set("trace-789");
        AtomicReference<String> captured = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        ScheduledFuture<?> future = contextScheduler.scheduleAtFixedRate(() -> {
            captured.set(TRACE_ID.get());
            latch.countDown();
        }, 0, 100, TimeUnit.MILLISECONDS);

        latch.await(5, TimeUnit.SECONDS);
        future.cancel(false);

        assertThat(captured.get()).isEqualTo("trace-789");
    }

    @Test
    void shouldPropagateThreadLocal_WhenScheduleWithFixedDelay() throws Exception {
        TRACE_ID.set("trace-abc");
        AtomicReference<String> captured = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        ScheduledFuture<?> future = contextScheduler.scheduleWithFixedDelay(() -> {
            captured.set(TRACE_ID.get());
            latch.countDown();
        }, 0, 100, TimeUnit.MILLISECONDS);

        latch.await(5, TimeUnit.SECONDS);
        future.cancel(false);

        assertThat(captured.get()).isEqualTo("trace-abc");
    }

    @Test
    void shouldPropagateThreadLocal_WhenUsingSubmitFromParentClass() throws Exception {
        TRACE_ID.set("trace-submit");

        String result = contextScheduler.submit(() -> TRACE_ID.get()).get();

        assertThat(result).isEqualTo("trace-submit");
    }

}
```

#### File: `src/test/java/dev/linhvu/context/ExecutorServiceIntegrationTest.java` [NEW]

```java
package dev.linhvu.context;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests verifying that executor wrappers work correctly with
 * the full context propagation stack: ThreadLocalAccessor + ContextRegistry
 * + ContextSnapshotFactory + ContextExecutorService.
 */
class ExecutorServiceIntegrationTest {

    private static final ThreadLocal<String> USER = new ThreadLocal<>();

    private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();

    private ContextRegistry registry;

    private ContextSnapshotFactory factory;

    private ExecutorService executor;

    @BeforeEach
    void setUp() {
        registry = new ContextRegistry();
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("user", USER));
        registry.registerThreadLocalAccessor(new TestThreadLocalAccessor("traceId", TRACE_ID));
        factory = ContextSnapshotFactory.builder().contextRegistry(registry).build();
        executor = ContextExecutorService.wrap(Executors.newFixedThreadPool(2), factory);
    }

    @AfterEach
    void tearDown() throws InterruptedException {
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        USER.remove();
        TRACE_ID.remove();
    }

    @Test
    void shouldPropagateMultipleThreadLocals_WhenSubmittingTask() throws Exception {
        USER.set("alice");
        TRACE_ID.set("trace-001");

        AtomicReference<String> capturedUser = new AtomicReference<>();
        AtomicReference<String> capturedTrace = new AtomicReference<>();

        executor.submit(() -> {
            capturedUser.set(USER.get());
            capturedTrace.set(TRACE_ID.get());
        }).get();

        assertThat(capturedUser.get()).isEqualTo("alice");
        assertThat(capturedTrace.get()).isEqualTo("trace-001");
    }

    @Test
    void shouldPropagatePartialContext_WhenOnlySomeThreadLocalsSet() throws Exception {
        USER.set("bob");
        // TRACE_ID not set

        AtomicReference<String> capturedUser = new AtomicReference<>();
        AtomicReference<String> capturedTrace = new AtomicReference<>();

        executor.submit(() -> {
            capturedUser.set(USER.get());
            capturedTrace.set(TRACE_ID.get());
        }).get();

        assertThat(capturedUser.get()).isEqualTo("bob");
        assertThat(capturedTrace.get()).isNull();
    }

    @Test
    void shouldPropagateWithClearMissing_WhenEnabled() throws Exception {
        ContextSnapshotFactory clearMissingFactory = ContextSnapshotFactory.builder()
                .contextRegistry(registry)
                .clearMissing(true)
                .build();
        ExecutorService clearMissingExecutor = ContextExecutorService.wrap(
                Executors.newSingleThreadExecutor(), clearMissingFactory);

        try {
            USER.set("carol");
            // TRACE_ID not set on the submitting thread

            // Set TRACE_ID on the worker thread beforehand
            clearMissingExecutor.submit(() -> TRACE_ID.set("stale-trace")).get();

            // Now submit with clearMissing=true — the missing TRACE_ID should be cleared
            AtomicReference<String> capturedUser = new AtomicReference<>();
            AtomicReference<String> capturedTrace = new AtomicReference<>();

            clearMissingExecutor.submit(() -> {
                capturedUser.set(USER.get());
                capturedTrace.set(TRACE_ID.get());
            }).get();

            assertThat(capturedUser.get()).isEqualTo("carol");
            assertThat(capturedTrace.get()).isNull(); // cleared because missing from snapshot
        }
        finally {
            clearMissingExecutor.shutdown();
            clearMissingExecutor.awaitTermination(5, TimeUnit.SECONDS);
        }
    }

    @Test
    void shouldPropagateWithKeyFilter_WhenPredicateConfigured() throws Exception {
        // Only capture the "user" key, ignore "traceId"
        ContextSnapshotFactory filteredFactory = ContextSnapshotFactory.builder()
                .contextRegistry(registry)
                .captureKeyPredicate(key -> "user".equals(key))
                .build();
        ExecutorService filteredExecutor = ContextExecutorService.wrap(
                Executors.newSingleThreadExecutor(), filteredFactory);

        try {
            USER.set("dave");
            TRACE_ID.set("trace-filtered");

            AtomicReference<String> capturedUser = new AtomicReference<>();
            AtomicReference<String> capturedTrace = new AtomicReference<>();

            filteredExecutor.submit(() -> {
                capturedUser.set(USER.get());
                capturedTrace.set(TRACE_ID.get());
            }).get();

            assertThat(capturedUser.get()).isEqualTo("dave");
            assertThat(capturedTrace.get()).isNull(); // not captured due to predicate
        }
        finally {
            filteredExecutor.shutdown();
            filteredExecutor.awaitTermination(5, TimeUnit.SECONDS);
        }
    }

    @Test
    void shouldWorkWithScheduledExecutor_AcrossFullStack() throws Exception {
        ScheduledExecutorService scheduledExecutor = ContextScheduledExecutorService.wrap(
                Executors.newScheduledThreadPool(1), factory);

        try {
            USER.set("eve");
            TRACE_ID.set("trace-scheduled");

            AtomicReference<String> capturedUser = new AtomicReference<>();
            AtomicReference<String> capturedTrace = new AtomicReference<>();

            scheduledExecutor.schedule(() -> {
                capturedUser.set(USER.get());
                capturedTrace.set(TRACE_ID.get());
            }, 10, TimeUnit.MILLISECONDS).get();

            assertThat(capturedUser.get()).isEqualTo("eve");
            assertThat(capturedTrace.get()).isEqualTo("trace-scheduled");
        }
        finally {
            scheduledExecutor.shutdown();
            scheduledExecutor.awaitTermination(5, TimeUnit.SECONDS);
        }
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **ContextExecutorService** | Decorator around `ExecutorService` that captures a fresh context snapshot on every task submission and wraps the task to restore that context on the worker thread |
| **ContextScheduledExecutorService** | Extension for `ScheduledExecutorService` that adds context propagation for `schedule()`, `scheduleAtFixedRate()`, and `scheduleWithFixedDelay()` |
| **Capture-at-submission-time** | The snapshot is taken when `submit()`/`execute()` is called, not when the wrapper is created — ensures each task carries the context active at its submission moment |
| **Decorator pattern** | The wrapper IS-an `ExecutorService` and HAS-an `ExecutorService` — callers use it transparently without knowing context propagation is happening |

**This is the final chapter** of Simple Context Propagation. The library is now complete: `ThreadLocalAccessor` (ch01) defines the contract, `ContextRegistry` (ch02) collects them, `ContextSnapshot` (ch03) captures and restores values, `ContextSnapshotFactory` (ch04) provides the capture API, and `ContextExecutorService` / `ContextScheduledExecutorService` (ch05) make it automatic for thread pools. The next step is to use this library in `simple-micrometer` and `simple-tracing` — they'll register their own `ThreadLocalAccessor` implementations and use `ContextExecutorService.wrap()` to propagate observations and spans across threads.
