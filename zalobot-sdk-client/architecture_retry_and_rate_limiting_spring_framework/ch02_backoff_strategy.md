# Chapter 2: BackOff Strategy — The Foundation for Retry Timing

## What You'll Learn
- Why backoff needs to be a **strategy interface** rather than a concrete class
- The `BackOff` / `BackOffExecution` split: configuration vs. stateful execution
- How `FixedBackOff` and `ExponentialBackOff` implement the same interface
- How jitter prevents the thundering herd problem
- How `STOP` signals tell the retry loop to give up

---

## 2.1 The Problem with the Current ExponentialBackOff

The existing `zalobot-listener/ExponentialBackOff` mixes configuration and execution state:

```java
// Current: mutable state baked into the config object
public class ExponentialBackOff {
    private final long initialIntervalMillis;  // config
    private final long maxIntervalMillis;       // config
    private final double multiplier;            // config
    private long currentIntervalMillis;         // ← execution state!

    public long nextBackOffMillis() { ... }     // mutates state
    public void reset() { ... }                 // manual reset required
}
```

This means:
- You **can't share one config** across multiple retry sessions (each would mutate the same state)
- You **must remember to call `reset()`** or the next retry session starts mid-sequence
- There's **no way to say "give up"** — it always returns an interval

---

## 2.2 Spring's Solution: Separate Config from Execution

Spring splits backoff into two interfaces:

```
BackOff (immutable config)          BackOffExecution (mutable state)
┌─────────────────────────┐         ┌──────────────────────────────┐
│ start() → BackOffExec.  │────────→│ nextBackOff() → long or STOP│
│                         │         │ (tracks attempts internally) │
│ (thread-safe, shareable)│         │ (single-use, not shared)     │
└─────────────────────────┘         └──────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - This is the **Factory Method** pattern: `BackOff.start()` creates a fresh execution each time. It solves the "shared mutable state" problem elegantly — one config object, many independent executions.
> - The alternative (passing attempt count into a stateless `getDelay(int attempt)` method) seems simpler but forces callers to track state. Spring chose to encapsulate state in the execution, following the principle of "make the right thing easy."
> - This pattern appears throughout Spring: `ConnectionFactory.createConnection()`, `SessionFactory.openSession()`, etc. When you see "factory creates stateful instance from config," it's this pattern.
> ─────────────────────────────────────────────────────

---

## 2.3 The Interfaces

### BackOff.java

```java
package dev.linhvu.zalobot.client.retry;

/**
 * Strategy interface for providing a {@link BackOffExecution} that determines
 * the rate at which an operation should be retried.
 *
 * <p>Implementations are expected to be thread-safe and reusable. Each call
 * to {@link #start()} creates a new, independent execution instance.
 */
@FunctionalInterface
public interface BackOff {

    /**
     * Start a new back-off execution.
     * @return a fresh {@link BackOffExecution} ready to be used
     */
    BackOffExecution start();
}
```

### BackOffExecution.java

```java
package dev.linhvu.zalobot.client.retry;

/**
 * Represents a single back-off execution session. Each call to
 * {@link #nextBackOff()} returns the number of milliseconds to wait
 * before the next retry attempt, or {@link #STOP} to indicate that
 * no further attempts should be made.
 *
 * <p>Implementations are stateful and not thread-safe. Each retry
 * session should use its own execution instance obtained via
 * {@link BackOff#start()}.
 */
@FunctionalInterface
public interface BackOffExecution {

    /**
     * Return value indicating that the operation should not be retried.
     */
    long STOP = -1;

    /**
     * Return the number of milliseconds to wait before the next retry,
     * or {@link #STOP} to indicate that retrying should stop.
     */
    long nextBackOff();
}
```

---

## 2.4 FixedBackOff — The Simplest Implementation

```java
package dev.linhvu.zalobot.client.retry;

/**
 * A {@link BackOff} implementation that provides a fixed interval between
 * retry attempts and a maximum number of attempts.
 *
 * <p>Example: {@code new FixedBackOff(1000, 3)} will wait 1 second between
 * retries, up to 3 retry attempts (4 total invocations including the initial).
 */
public class FixedBackOff implements BackOff {

    public static final long DEFAULT_INTERVAL = 5000;
    public static final long UNLIMITED_ATTEMPTS = Long.MAX_VALUE;

    private long interval;
    private long maxAttempts;

    public FixedBackOff() {
        this(DEFAULT_INTERVAL, UNLIMITED_ATTEMPTS);
    }

    public FixedBackOff(long interval, long maxAttempts) {
        this.interval = interval;
        this.maxAttempts = maxAttempts;
    }

    // getters and setters ...

    public long getInterval() { return this.interval; }
    public void setInterval(long interval) { this.interval = interval; }
    public long getMaxAttempts() { return this.maxAttempts; }
    public void setMaxAttempts(long maxAttempts) { this.maxAttempts = maxAttempts; }

    @Override
    public BackOffExecution start() {
        return new FixedBackOffExecution();
    }

    private class FixedBackOffExecution implements BackOffExecution {
        private long currentAttempts = 0;

        @Override
        public long nextBackOff() {
            this.currentAttempts++;
            if (this.currentAttempts <= getMaxAttempts()) {
                return getInterval();
            }
            return STOP;  // ← "give up" signal
        }
    }
}
```

**Trace through 3 calls with `new FixedBackOff(1000, 3)`:**

| Call # | `currentAttempts` | `<= maxAttempts`? | Returns |
|--------|-------------------|-------------------|---------|
| 1 | 1 | 1 <= 3 ✓ | 1000 |
| 2 | 2 | 2 <= 3 ✓ | 1000 |
| 3 | 3 | 3 <= 3 ✓ | 1000 |
| 4 | 4 | 4 <= 3 ✗ | STOP (-1) |

> ★ **Insight** ─────────────────────────────────────
> - The `STOP` sentinel value (`-1`) is a classic "special return value" pattern — simpler than throwing an exception or returning an `Optional<Long>`. Spring chose `-1` because valid delays are always non-negative.
> - An alternative is `OptionalLong` but it creates garbage on every call. For a hot retry loop, the primitive `long` with sentinel is more efficient.
> ─────────────────────────────────────────────────────

---

## 2.5 ExponentialBackOff — With Jitter and Stop Support

```java
package dev.linhvu.zalobot.client.retry;

/**
 * A {@link BackOff} implementation that increases the delay exponentially
 * for each attempt, with optional jitter to prevent thundering herd.
 *
 * <p>Example with defaults: initial=2000ms, multiplier=1.5, maxInterval=30s
 * <pre>
 * attempt#   delay
 *    1        2000
 *    2        3000
 *    3        4500
 *    4        6750
 *    5       10125
 *    ...     (capped at 30000)
 * </pre>
 */
public class ExponentialBackOff implements BackOff {

    public static final long DEFAULT_INITIAL_INTERVAL = 2000L;
    public static final long DEFAULT_JITTER = 0;
    public static final double DEFAULT_MULTIPLIER = 1.5;
    public static final long DEFAULT_MAX_INTERVAL = 30_000L;
    public static final long DEFAULT_MAX_ATTEMPTS = Long.MAX_VALUE;

    private long initialInterval = DEFAULT_INITIAL_INTERVAL;
    private long jitter = DEFAULT_JITTER;
    private double multiplier = DEFAULT_MULTIPLIER;
    private long maxInterval = DEFAULT_MAX_INTERVAL;
    private long maxAttempts = DEFAULT_MAX_ATTEMPTS;

    public ExponentialBackOff() {}

    public ExponentialBackOff(long initialInterval, double multiplier) {
        this.initialInterval = initialInterval;
        this.multiplier = multiplier;
    }

    // getters and setters for all fields ...
    public long getInitialInterval() { return this.initialInterval; }
    public void setInitialInterval(long v) { this.initialInterval = v; }
    public long getJitter() { return this.jitter; }
    public void setJitter(long v) { this.jitter = v; }
    public double getMultiplier() { return this.multiplier; }
    public void setMultiplier(double v) { this.multiplier = v; }
    public long getMaxInterval() { return this.maxInterval; }
    public void setMaxInterval(long v) { this.maxInterval = v; }
    public long getMaxAttempts() { return this.maxAttempts; }
    public void setMaxAttempts(long v) { this.maxAttempts = v; }

    @Override
    public BackOffExecution start() {
        return new ExponentialBackOffExecution();
    }

    private class ExponentialBackOffExecution implements BackOffExecution {
        private long currentInterval = -1;
        private int attempts = 0;

        @Override
        public long nextBackOff() {
            if (this.attempts >= getMaxAttempts()) {
                return STOP;
            }
            long nextInterval = computeNextInterval();
            this.attempts++;
            return nextInterval;
        }

        private long computeNextInterval() {
            long maxInterval = getMaxInterval();
            long nextInterval;

            if (this.currentInterval < 0) {
                // First call: use initial interval
                nextInterval = getInitialInterval();
            }
            else if (this.currentInterval >= maxInterval) {
                // Already at max: stay there
                nextInterval = maxInterval;
            }
            else {
                // Multiply and cap
                nextInterval = Math.min(
                    (long) (this.currentInterval * getMultiplier()),
                    maxInterval
                );
            }

            this.currentInterval = nextInterval;
            return Math.min(applyJitter(nextInterval), maxInterval);
        }

        private long applyJitter(long interval) {
            long jitter = getJitter();
            if (jitter > 0) {
                long initialInterval = getInitialInterval();
                // Scale jitter proportionally to interval growth
                long applicableJitter = jitter * (interval / initialInterval);
                long min = Math.max(interval - applicableJitter, initialInterval);
                long max = Math.min(interval + applicableJitter, getMaxInterval());
                return min + (long) (Math.random() * (max - min));
            }
            return interval;
        }
    }
}
```

**Trace through 5 calls with `initialInterval=1000, multiplier=2.0, maxInterval=10000, maxAttempts=5`:**

| Call # | `currentInterval` before | Computed | Capped | `attempts` after | Returns |
|--------|--------------------------|----------|--------|------------------|---------|
| 1 | -1 (initial) | 1000 | 1000 | 1 | 1000 |
| 2 | 1000 | 2000 | 2000 | 2 | 2000 |
| 3 | 2000 | 4000 | 4000 | 3 | 4000 |
| 4 | 4000 | 8000 | 8000 | 4 | 8000 |
| 5 | 8000 | 16000→10000 | 10000 | 5 | 10000 |
| 6 | — | — | — | 5 >= 5 | STOP |

> ★ **Insight** ─────────────────────────────────────
> - **Jitter prevents thundering herd**: Without jitter, 100 clients that fail at time T=0 will ALL retry at T=1000, T=2000, T=4000 — creating synchronized bursts. With jitter, retries spread across a time window, smoothing the load.
> - Spring scales jitter proportionally: `applicableJitter = jitter * (interval / initialInterval)`. This means early retries (small interval) have small jitter, while later retries (large interval) have large jitter. This is better than fixed jitter because the spread grows with the delay.
> - **When NOT to use jitter**: If you're the only client (e.g., a single polling loop), jitter adds unnecessary randomness. It's most valuable when many independent clients retry against the same server.
> ─────────────────────────────────────────────────────

---

## 2.6 Connection to Real Spring Framework

| Simplified | Real Spring Source | Key Differences |
|-----------|-------------------|-----------------|
| `BackOff` | `o.s.util.backoff.BackOff` (`BackOff.java:48`) | Identical interface |
| `BackOffExecution` | `o.s.util.backoff.BackOffExecution` (`BackOffExecution.java:30`) | Identical interface |
| `FixedBackOff` | `o.s.util.backoff.FixedBackOff` (`FixedBackOff.java:29`) | Spring adds `toString()` |
| `ExponentialBackOff` | `o.s.util.backoff.ExponentialBackOff` (`ExponentialBackOff.java:63`) | Spring adds `maxElapsedTime` tracking; we omit it (handled by RetryPolicy timeout) |
| Jitter logic | `ExponentialBackOffExecution.applyJitter()` (`ExponentialBackOff.java:310`) | Identical formula |
| `STOP = -1` | `BackOffExecution.STOP` (`BackOffExecution.java:36`) | Identical sentinel |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **BackOff** | Immutable configuration for delay strategy; creates fresh executions via `start()` |
| **BackOffExecution** | Mutable, single-use state tracker; returns delay or `STOP` |
| **STOP sentinel** | Special return value (-1) meaning "no more retries" |
| **FixedBackOff** | Same delay every time, up to N attempts |
| **ExponentialBackOff** | Delay grows by multiplier each attempt, capped at maxInterval |
| **Jitter** | Random offset added to delay to prevent synchronized retries |
| **Factory Method** | `start()` creates independent stateful executions from shared config |

**Next: Chapter 3** — We'll build `RetryPolicy`, which decides *whether* to retry based on exception type and attempt count, composing with `BackOff` for the *timing* decision.
