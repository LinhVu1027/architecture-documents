# Chapter 4: Retry Template — The Execution Engine

## What You'll Learn
- How `RetryTemplate` orchestrates the retry loop with state tracking
- The `Retryable<R>` functional interface for wrapping operations
- How `RetryListener` provides observability into retry lifecycle
- The `RetryException` terminal failure with suppressed exception chain
- Timeout checking: preemptive vs. post-sleep

---

## 4.1 The Problem: Who Runs the Loop?

We now have `BackOff` (how long to wait) and `RetryPolicy` (whether to retry). But we still need something to:

1. Run the initial attempt
2. Catch the exception
3. Ask `RetryPolicy.shouldRetry()`
4. Get `BackOff.start()` and call `nextBackOff()`
5. Sleep for the delay
6. Retry the operation
7. Handle success, exhaustion, timeout, and interruption
8. Notify listeners at each phase

This is `RetryTemplate` — the **Template Method** pattern applied to retry logic.

---

## 4.2 Supporting Types

### Retryable<R> — The Operation to Retry

```java
package dev.linhvu.zalobot.client.retry;

/**
 * A retryable operation that produces a result of type {@code R}.
 * Used as input to {@link RetryTemplate#execute(Retryable)}.
 */
@FunctionalInterface
public interface Retryable<R> {

    /**
     * Execute the operation.
     * @return the result
     * @throws Throwable if the operation fails
     */
    R execute() throws Throwable;

    /**
     * A logical name for this operation, used in log messages.
     */
    default String getName() {
        return getClass().getName();
    }
}
```

### RetryListener — Observability

```java
package dev.linhvu.zalobot.client.retry;

/**
 * Listener for retry lifecycle events. All methods have empty default
 * implementations, so you only override what you need.
 */
public interface RetryListener {

    /** Called after every attempt (initial + retries), success or failure. */
    default void onRetryableExecution(RetryPolicy policy, Retryable<?> retryable, RetryState state) {}

    /** Called before each retry attempt (not the initial attempt). */
    default void beforeRetry(RetryPolicy policy, Retryable<?> retryable, RetryState state) {}

    /** Called after the first successful retry. */
    default void onRetrySuccess(RetryPolicy policy, Retryable<?> retryable, Object result) {}

    /** Called after each failed retry attempt. */
    default void onRetryFailure(RetryPolicy policy, Retryable<?> retryable, Throwable throwable) {}

    /** Called when all retries are exhausted. */
    default void onRetryPolicyExhaustion(RetryPolicy policy, Retryable<?> retryable, RetryException exception) {}

    /** Called when interrupted during backoff sleep. */
    default void onRetryPolicyInterruption(RetryPolicy policy, Retryable<?> retryable, RetryException exception) {}

    /** Called when the overall timeout is exceeded. */
    default void onRetryPolicyTimeout(RetryPolicy policy, Retryable<?> retryable, RetryException exception) {}
}
```

### RetryState — Current State of Retry Processing

```java
package dev.linhvu.zalobot.client.retry;

import java.util.List;

/**
 * Read-only view of the current retry state.
 */
public interface RetryState {

    /** Number of retry attempts so far (0 = still on initial attempt). */
    int getRetryCount();

    /** All exceptions encountered so far, in order. */
    List<Throwable> getExceptions();

    /** The most recent exception. */
    default Throwable getLastException() {
        List<Throwable> exceptions = getExceptions();
        return exceptions.isEmpty() ? null : exceptions.get(exceptions.size() - 1);
    }

    /** Whether the last attempt was successful. */
    default boolean isSuccessful() {
        return getExceptions().isEmpty() || getLastException() == null;
    }
}
```

### RetryException — Terminal Failure

```java
package dev.linhvu.zalobot.client.retry;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Thrown when a retry policy is exhausted, interrupted, or times out.
 * Contains the last exception as the cause and all previous exceptions
 * as suppressed exceptions.
 */
public class RetryException extends Exception implements RetryState {

    RetryException(String message, RetryState retryState) {
        super(message, retryState.getLastException());
        List<Throwable> exceptions = retryState.getExceptions();
        // Add all but the last as suppressed (last is the cause)
        for (int i = 0; i < exceptions.size() - 1; i++) {
            addSuppressed(exceptions.get(i));
        }
    }

    @Override
    public int getRetryCount() {
        return getSuppressed().length;
    }

    @Override
    public List<Throwable> getExceptions() {
        Throwable[] suppressed = getSuppressed();
        List<Throwable> exceptions = new ArrayList<>(suppressed.length + 1);
        Collections.addAll(exceptions, suppressed);
        exceptions.add(getCause());
        return Collections.unmodifiableList(exceptions);
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - `RetryException` uses Java's **suppressed exception** mechanism (introduced in Java 7 for try-with-resources). This is clever: the `cause` is the last failure (most relevant for debugging), while previous failures are preserved as suppressed. Stack traces show all of them.
> - An alternative is a custom `List<Throwable>` field, but suppressed exceptions integrate with standard logging frameworks and `printStackTrace()` output.
> - `RetryException` also implements `RetryState`, so the same object serves dual purpose: it's both the exception thrown to callers AND the state object passed to `RetryListener.onRetryPolicyExhaustion()`.
> ─────────────────────────────────────────────────────

---

## 4.3 RetryTemplate: The Core Loop

```java
package dev.linhvu.zalobot.client.retry;

import java.lang.reflect.UndeclaredThrowableException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.function.Supplier;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Executes a {@link Retryable} operation with retry logic driven by a
 * {@link RetryPolicy}. The default policy retries up to 3 times with
 * a 1-second fixed backoff.
 *
 * <p>Thread-safe: multiple threads can call {@code execute()} concurrently
 * as long as the {@link Retryable} itself is thread-safe.
 */
public class RetryTemplate {

    private static final Logger logger = LoggerFactory.getLogger(RetryTemplate.class);

    private RetryPolicy retryPolicy = RetryPolicy.withDefaults();
    private RetryListener retryListener = new RetryListener() {};

    public RetryTemplate() {}

    public RetryTemplate(RetryPolicy retryPolicy) {
        this.retryPolicy = retryPolicy;
    }

    public void setRetryPolicy(RetryPolicy retryPolicy) {
        this.retryPolicy = retryPolicy;
    }

    public RetryPolicy getRetryPolicy() {
        return this.retryPolicy;
    }

    public void setRetryListener(RetryListener retryListener) {
        this.retryListener = retryListener;
    }

    /**
     * Execute the retryable operation. Returns the result on success,
     * or throws {@link RetryException} when all retries are exhausted.
     */
    public <R> R execute(Retryable<R> retryable) throws RetryException {
        long startTime = System.currentTimeMillis();
        String name = retryable.getName();
        MutableRetryState state = new MutableRetryState();

        // ═══════════════════════════════════════════
        // PHASE 1: Initial attempt
        // ═══════════════════════════════════════════
        logger.debug("Executing retryable operation '{}'", name);
        R result;
        try {
            result = retryable.execute();
        }
        catch (Throwable initialException) {
            logger.debug("Operation '{}' failed; initiating retry", name, initialException);
            state.addException(initialException);
            this.retryListener.onRetryableExecution(this.retryPolicy, retryable, state);

            // ═══════════════════════════════════════
            // PHASE 2: Retry loop
            // ═══════════════════════════════════════
            BackOffExecution backOffExecution = this.retryPolicy.getBackOff().start();
            Throwable lastException = initialException;
            long timeout = this.retryPolicy.getTimeout().toMillis();

            while (this.retryPolicy.shouldRetry(lastException)
                    && state.getRetryCount() < Integer.MAX_VALUE) {

                // Check timeout BEFORE sleeping
                checkTimeout(timeout, startTime, 0, name, state);

                // Get sleep time from backoff
                long sleepTime;
                try {
                    sleepTime = backOffExecution.nextBackOff();
                    if (sleepTime == BackOffExecution.STOP) {
                        break;  // BackOff says "no more retries"
                    }
                    // Check if sleeping would exceed timeout
                    checkTimeout(timeout, startTime, sleepTime, name, state);
                    logger.debug("Backing off {}ms for '{}'", sleepTime, name);
                    Thread.sleep(sleepTime);
                }
                catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    RetryException ex = new RetryException(
                        "Interrupted during back-off for '" + name + "'", state);
                    this.retryListener.onRetryPolicyInterruption(
                        this.retryPolicy, retryable, ex);
                    throw ex;
                }

                // Execute retry attempt
                state.increaseRetryCount();
                this.retryListener.beforeRetry(this.retryPolicy, retryable, state);

                try {
                    result = retryable.execute();
                }
                catch (Throwable retryException) {
                    logger.debug("Retry #{} for '{}' failed", state.getRetryCount(), name, retryException);
                    state.addException(retryException);
                    this.retryListener.onRetryFailure(this.retryPolicy, retryable, retryException);
                    this.retryListener.onRetryableExecution(this.retryPolicy, retryable, state);
                    lastException = retryException;
                    continue;
                }

                // Retry succeeded!
                logger.debug("Operation '{}' succeeded after {} retries", name, state.getRetryCount());
                this.retryListener.onRetrySuccess(this.retryPolicy, retryable, result);
                this.retryListener.onRetryableExecution(this.retryPolicy, retryable, state);
                return result;
            }

            // ═══════════════════════════════════════
            // PHASE 3: Policy exhausted
            // ═══════════════════════════════════════
            RetryException retryEx = new RetryException(
                "Retry policy for '" + name + "' exhausted after "
                    + state.getRetryCount() + " retries", state);
            this.retryListener.onRetryPolicyExhaustion(this.retryPolicy, retryable, retryEx);
            throw retryEx;
        }

        // Initial attempt succeeded — no retry needed
        logger.debug("Operation '{}' completed successfully", name);
        this.retryListener.onRetryableExecution(this.retryPolicy, retryable, state);
        return result;
    }

    /**
     * Convenience: invoke a Supplier with retry, rethrowing RuntimeExceptions.
     */
    public <R> R invoke(Supplier<R> supplier) {
        try {
            return execute(new Retryable<R>() {
                @Override public R execute() { return supplier.get(); }
                @Override public String getName() { return supplier.getClass().getName(); }
            });
        }
        catch (RetryException e) {
            Throwable cause = e.getCause();
            if (cause instanceof RuntimeException re) throw re;
            if (cause instanceof Error err) throw err;
            throw new UndeclaredThrowableException(cause);
        }
    }

    // ─── Timeout checking ────────────────────────────

    private void checkTimeout(long timeout, long startTime, long pendingSleep,
                              String name, RetryState state) throws RetryException {
        if (timeout > 0) {
            long elapsed = System.currentTimeMillis() + pendingSleep - startTime;
            if (elapsed >= timeout) {
                String msg = pendingSleep > 0
                    ? "Retry for '" + name + "' would exceed timeout due to pending sleep"
                    : "Retry for '" + name + "' exceeded timeout";
                RetryException ex = new RetryException(msg, state);
                this.retryListener.onRetryPolicyTimeout(this.retryPolicy, null, ex);
                throw ex;
            }
        }
    }

    // ─── Mutable state tracker ───────────────────────

    private static class MutableRetryState implements RetryState {
        private int retryCount;
        private final List<Throwable> exceptions = new ArrayList<>(4);

        void increaseRetryCount() { this.retryCount++; }
        void addException(Throwable e) { this.exceptions.add(e); }

        @Override public int getRetryCount() { return this.retryCount; }
        @Override public List<Throwable> getExceptions() {
            return Collections.unmodifiableList(this.exceptions);
        }
    }
}
```

---

## 4.4 Execution Flow Walkthrough

**Scenario:** `sendMessage()` fails twice with `IOException`, then succeeds on the third try.

```
Policy: maxRetries=3, delay=1000ms, multiplier=2.0

Time    Action                          State
─────   ─────────────────────────────   ──────────────────
0ms     execute() → Initial attempt     retryCount=0
        → throws IOException
        state.addException(IOException)
        listener.onRetryableExecution()

        backOff.start() → execution
        shouldRetry(IOException) → true

        nextBackOff() → 1000ms
        checkTimeout(0+1000) → OK
        Thread.sleep(1000)

1000ms  state.retryCount++ → 1
        listener.beforeRetry()
        retryable.execute() → retry #1
        → throws IOException
        state.addException(IOException)
        listener.onRetryFailure()
        listener.onRetryableExecution()

        shouldRetry(IOException) → true
        nextBackOff() → 2000ms
        checkTimeout(0+2000) → OK
        Thread.sleep(2000)

3000ms  state.retryCount++ → 2
        listener.beforeRetry()
        retryable.execute() → retry #2
        → SUCCESS! returns SendMessageResult

        listener.onRetrySuccess()
        listener.onRetryableExecution()
        return SendMessageResult
```

> ★ **Insight** ─────────────────────────────────────
> - The retry loop has **two exit paths that stop retrying**: `shouldRetry()` returns `false` (policy says "don't retry this exception type"), or `nextBackOff()` returns `STOP` (backoff says "max attempts reached"). This dual control means `RetryPolicy` handles the *what* while `BackOff` handles the *when* — and either can stop the loop.
> - Spring's `RetryTemplate` checks timeout **twice per iteration**: once before sleeping (current elapsed >= timeout?) and once after computing sleep time (elapsed + sleep >= timeout?). The second check is **preemptive**: it prevents sleeping for 30 seconds when the timeout would expire in 5 seconds. This avoids wasting time on a sleep that will just result in a timeout.
> - The `MutableRetryState` is a private inner class — it's never exposed outside `RetryTemplate`. The `RetryState` interface (read-only) is what listeners see. This is the **Interface Segregation Principle**: mutation is internal, observation is external.
> ─────────────────────────────────────────────────────

---

## 4.5 Connection to Real Spring Framework

| Simplified | Real Spring Source | Key Differences |
|-----------|-------------------|-----------------|
| `RetryTemplate` | `o.s.core.retry.RetryTemplate` (`RetryTemplate.java:56`) | Spring uses `LogAccessor` instead of SLF4J directly |
| `execute()` method | `RetryTemplate.execute()` (`RetryTemplate.java:126-207`) | Identical control flow |
| `checkTimeout()` | `RetryTemplate.checkIfTimeoutExceeded()` (`RetryTemplate.java:209-228`) | Same preemptive + actual timeout checks |
| `MutableRetryState` | `RetryTemplate.MutableRetryState` (`RetryTemplate.java:284-312`) | Identical structure |
| `invoke(Supplier)` | `RetryTemplate.invoke(Supplier)` (`RetryTemplate.java:231-254`) | Same unwrapping of RetryException to RuntimeException |
| `Retryable<R>` | `o.s.core.retry.Retryable` (`Retryable.java:34`) | Identical |
| `RetryListener` | `o.s.core.retry.RetryListener` (`RetryListener.java:34`) | Spring has overloaded `beforeRetry()` with 2-arg and 3-arg versions |
| `RetryException` | `o.s.core.retry.RetryException` (`RetryException.java:42`) | Identical suppressed-exception pattern |
| `RetryState` | `o.s.core.retry.RetryState` (`RetryState.java`) | Identical interface |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **RetryTemplate** | The execution engine: runs initial attempt → retry loop → success or exhaustion |
| **Retryable<R>** | Functional interface wrapping the operation to retry |
| **RetryListener** | Observer with 7 callbacks for retry lifecycle events |
| **RetryState** | Read-only view of retry progress (count, exceptions) |
| **RetryException** | Terminal exception with cause + suppressed exceptions |
| **MutableRetryState** | Internal state tracker; exposed as read-only `RetryState` to listeners |
| **Preemptive timeout** | Check if sleeping would exceed timeout, before actually sleeping |

**Next: Chapter 5** — We'll wire `RetryTemplate` into `DefaultZaloBotClient.exchangeInternal()` so that every API call benefits from retry automatically.
