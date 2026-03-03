# Chapter 3: Retry Policy — Deciding When to Retry

## What You'll Learn
- Why retry decisions need a separate abstraction from backoff timing
- How to filter exceptions with include/exclude sets
- The `RetryPolicy.Builder` fluent API for concise configuration
- How `RetryPolicy` composes `BackOff` to answer both "should we?" and "how long?"
- Why authentication errors must never be retried

---

## 3.1 The Problem: Not All Errors Are Equal

From Chapter 1, we know `DefaultZaloBotClient` throws three kinds of exceptions:

```
ZaloBotException
├── ZaloBotClientException       ← network errors (IOException)    → SHOULD RETRY
├── ZaloBotApiException          ← API errors (rate limit, etc.)   → MAYBE RETRY
│   └── ZaloBotAuthenticationException ← auth errors               → NEVER RETRY
└── ZaloBotSerializationException ← JSON parsing                   → NEVER RETRY
```

A naive "retry everything" strategy would:
- Waste time retrying bad credentials (they won't magically become valid)
- Waste time retrying malformed JSON (the response won't change)
- Correctly retry network timeouts and rate limits

We need a way to say: "Retry `ZaloBotClientException` and `ZaloBotApiException`, but **never** retry `ZaloBotAuthenticationException` or `ZaloBotSerializationException`."

---

## 3.2 Spring's RetryPolicy: Three Responsibilities

Spring's `RetryPolicy` answers three questions:

| Question | Method | Returns |
|----------|--------|---------|
| Should we retry this exception? | `shouldRetry(Throwable)` | `boolean` |
| How long should we wait? | `getBackOff()` | `BackOff` |
| When should we time out overall? | `getTimeout()` | `Duration` |

```java
package dev.linhvu.zalobot.client.retry;

import java.time.Duration;

/**
 * Strategy interface that defines a retry policy: which exceptions to retry,
 * how long to back off, and when to give up entirely.
 *
 * <p>Use {@link #withDefaults()} for a sensible default (retry all exceptions,
 * 3 attempts, 1-second fixed delay), or {@link #builder()} for full control.
 */
public interface RetryPolicy {

    /**
     * Determine whether the operation should be retried for the given exception.
     */
    boolean shouldRetry(Throwable throwable);

    /**
     * Get the overall timeout. Zero means no timeout.
     */
    default Duration getTimeout() {
        return Duration.ZERO;
    }

    /**
     * Get the backoff strategy. Default: fixed 1-second delay, max 3 retries.
     */
    default BackOff getBackOff() {
        return new FixedBackOff(Builder.DEFAULT_DELAY, Builder.DEFAULT_MAX_RETRIES);
    }

    // Factory methods — see section 3.3
    static RetryPolicy withDefaults() {
        return throwable -> true;
    }

    static RetryPolicy withMaxRetries(long maxRetries) {
        return builder().maxRetries(maxRetries).build();
    }

    static Builder builder() {
        return new Builder();
    }

    // Builder — see section 3.4
    // ...
}
```

> ★ **Insight** ─────────────────────────────────────
> - `RetryPolicy` is itself a **functional interface** (only `shouldRetry` is abstract). This means you can write `RetryPolicy policy = ex -> ex instanceof IOException` — a one-liner lambda for simple cases.
> - But it also has default methods (`getTimeout()`, `getBackOff()`) and a `Builder` for complex cases. This dual nature — lambda for simple, Builder for complex — is a pattern Spring uses extensively (`Predicate`, `Comparator`, etc.).
> - **When NOT to use the lambda form**: When you need custom timeout or backoff. The lambda form only controls exception filtering; timeout and backoff remain at defaults.
> ─────────────────────────────────────────────────────

---

## 3.3 Factory Methods: Convenience on Top of Builder

```java
// Inside RetryPolicy interface:

/**
 * Default: retry all exceptions, 3 attempts, 1-second fixed delay.
 */
static RetryPolicy withDefaults() {
    return throwable -> true;  // lambda implements shouldRetry()
}

/**
 * Retry all exceptions with custom attempt count.
 */
static RetryPolicy withMaxRetries(long maxRetries) {
    return builder()
            .maxRetries(maxRetries)
            .build();
}
```

Usage:
```java
// Simple: retry everything 3 times
RetryPolicy simple = RetryPolicy.withDefaults();

// Simple: retry everything 5 times
RetryPolicy fiveTimes = RetryPolicy.withMaxRetries(5);

// Custom: retry only IOExceptions, exponential backoff
RetryPolicy custom = RetryPolicy.builder()
    .includes(ZaloBotClientException.class)
    .excludes(ZaloBotAuthenticationException.class)
    .delay(Duration.ofSeconds(1))
    .multiplier(2.0)
    .maxDelay(Duration.ofSeconds(30))
    .maxRetries(5)
    .timeout(Duration.ofMinutes(2))
    .build();
```

---

## 3.4 The Builder: Full Configuration Power

```java
// Inside RetryPolicy interface:

final class Builder {

    public static final long DEFAULT_MAX_RETRIES = 3;
    public static final long DEFAULT_DELAY = 1000;
    public static final long DEFAULT_MAX_DELAY = Long.MAX_VALUE;
    public static final double DEFAULT_MULTIPLIER = 1.0;  // fixed by default

    private BackOff backOff;
    private Long maxRetries;
    private Duration timeout = Duration.ZERO;
    private Duration delay;
    private Duration jitter;
    private Double multiplier;
    private Duration maxDelay;

    private final Set<Class<? extends Throwable>> includes = new LinkedHashSet<>();
    private final Set<Class<? extends Throwable>> excludes = new LinkedHashSet<>();
    private Predicate<Throwable> predicate;

    Builder() {}

    // --- Exception filtering ---

    @SafeVarargs
    public final Builder includes(Class<? extends Throwable>... types) {
        Collections.addAll(this.includes, types);
        return this;
    }

    @SafeVarargs
    public final Builder excludes(Class<? extends Throwable>... types) {
        Collections.addAll(this.excludes, types);
        return this;
    }

    public Builder predicate(Predicate<Throwable> predicate) {
        this.predicate = (this.predicate != null)
                ? this.predicate.and(predicate) : predicate;
        return this;
    }

    // --- Timing ---

    public Builder maxRetries(long maxRetries) {
        this.maxRetries = maxRetries;
        return this;
    }

    public Builder timeout(Duration timeout) {
        this.timeout = timeout;
        return this;
    }

    public Builder delay(Duration delay) {
        this.delay = delay;
        return this;
    }

    public Builder jitter(Duration jitter) {
        this.jitter = jitter;
        return this;
    }

    public Builder multiplier(double multiplier) {
        this.multiplier = multiplier;
        return this;
    }

    public Builder maxDelay(Duration maxDelay) {
        this.maxDelay = maxDelay;
        return this;
    }

    public Builder backOff(BackOff backOff) {
        this.backOff = backOff;
        return this;
    }

    // --- Build ---

    public RetryPolicy build() {
        BackOff backOff = this.backOff;
        if (backOff != null) {
            // Custom BackOff: can't also specify delay/multiplier/etc.
            boolean misconfigured = (this.maxRetries != null || this.delay != null
                    || this.jitter != null || this.multiplier != null
                    || this.maxDelay != null);
            if (misconfigured) {
                throw new IllegalStateException(
                    "Cannot combine custom BackOff with delay/multiplier/maxRetries/jitter/maxDelay");
            }
        }
        else {
            // Build ExponentialBackOff from individual settings
            ExponentialBackOff exponential = new ExponentialBackOff();
            exponential.setMaxAttempts(
                this.maxRetries != null ? this.maxRetries : DEFAULT_MAX_RETRIES);
            exponential.setInitialInterval(
                this.delay != null ? this.delay.toMillis() : DEFAULT_DELAY);
            exponential.setMaxInterval(
                this.maxDelay != null ? this.maxDelay.toMillis() : DEFAULT_MAX_DELAY);
            exponential.setMultiplier(
                this.multiplier != null ? this.multiplier : DEFAULT_MULTIPLIER);
            if (this.jitter != null) {
                exponential.setJitter(this.jitter.toMillis());
            }
            backOff = exponential;
        }

        return new DefaultRetryPolicy(
            this.includes, this.excludes, this.predicate, this.timeout, backOff);
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - The Builder has a **mutual exclusion constraint**: you can either provide a custom `BackOff` OR use `delay`/`multiplier`/`jitter`/`maxDelay`/`maxRetries`, but not both. This prevents contradictory configurations (e.g., `backOff(new FixedBackOff(500, 2)).delay(Duration.ofSeconds(3))` — which delay wins?).
> - Spring validates this at `build()` time, not at each setter. This is deliberate: builders are often populated in multiple steps (conditional logic, configuration injection), and validating incrementally would be annoying and fragile.
> - Notice that `DEFAULT_MULTIPLIER = 1.0` — meaning the builder creates a **fixed** backoff by default, not exponential. You must explicitly set `multiplier(2.0)` to get exponential behavior. This is conservative: surprises in retry timing can cause subtle production issues.
> ─────────────────────────────────────────────────────

---

## 3.5 DefaultRetryPolicy: Exception Filtering

```java
package dev.linhvu.zalobot.client.retry;

import java.time.Duration;
import java.util.Set;
import java.util.function.Predicate;

/**
 * Default implementation of {@link RetryPolicy} created by the Builder.
 * Uses include/exclude sets for exception type matching.
 */
class DefaultRetryPolicy implements RetryPolicy {

    private final Set<Class<? extends Throwable>> includes;
    private final Set<Class<? extends Throwable>> excludes;
    private final Predicate<Throwable> predicate;
    private final Duration timeout;
    private final BackOff backOff;

    DefaultRetryPolicy(
            Set<Class<? extends Throwable>> includes,
            Set<Class<? extends Throwable>> excludes,
            Predicate<Throwable> predicate,
            Duration timeout,
            BackOff backOff) {
        this.includes = includes;
        this.excludes = excludes;
        this.predicate = predicate;
        this.timeout = timeout;
        this.backOff = backOff;
    }

    @Override
    public boolean shouldRetry(Throwable throwable) {
        // 1. Check excludes first (takes priority)
        for (Class<? extends Throwable> excluded : this.excludes) {
            if (excluded.isInstance(throwable)) {
                return false;
            }
        }

        // 2. Check includes (empty = include all)
        boolean included = this.includes.isEmpty();
        if (!included) {
            for (Class<? extends Throwable> inc : this.includes) {
                if (inc.isInstance(throwable)) {
                    included = true;
                    break;
                }
            }
        }

        // 3. Apply custom predicate
        if (included && this.predicate != null) {
            return this.predicate.test(throwable);
        }
        return included;
    }

    @Override
    public Duration getTimeout() {
        return this.timeout;
    }

    @Override
    public BackOff getBackOff() {
        return this.backOff;
    }
}
```

**Exception filtering walkthrough:**

Given this policy:
```java
RetryPolicy policy = RetryPolicy.builder()
    .includes(ZaloBotClientException.class, ZaloBotApiException.class)
    .excludes(ZaloBotAuthenticationException.class)
    .build();
```

| Exception Type | Excluded? | Included? | Result |
|---------------|-----------|-----------|--------|
| `ZaloBotClientException` | No | Yes (directly) | **RETRY** |
| `ZaloBotApiException` | No | Yes (directly) | **RETRY** |
| `ZaloBotAuthenticationException` | **Yes** | (skipped) | **NO RETRY** |
| `ZaloBotSerializationException` | No | No (not in includes) | **NO RETRY** |
| `RuntimeException` | No | No (not in includes) | **NO RETRY** |

Note: `ZaloBotAuthenticationException extends ZaloBotApiException`, so without the `excludes`, it would match `includes(ZaloBotApiException.class)`. The exclude takes priority, which is exactly what we want — don't retry bad credentials even though they're technically API errors.

---

## 3.6 Recommended Policy for Zalobot SDK

```java
// The "sensible default" for Zalo Bot API calls:
RetryPolicy zaloBotDefault = RetryPolicy.builder()
    .includes(ZaloBotClientException.class, ZaloBotApiException.class)
    .excludes(ZaloBotAuthenticationException.class, ZaloBotSerializationException.class)
    .delay(Duration.ofSeconds(1))
    .multiplier(2.0)
    .maxDelay(Duration.ofSeconds(30))
    .maxRetries(3)
    .jitter(Duration.ofMillis(200))
    .timeout(Duration.ofMinutes(2))
    .build();
```

This means:
- Retry network errors (`ZaloBotClientException`) — transient by nature
- Retry API errors (`ZaloBotApiException`) — includes rate limiting
- **Never** retry auth errors — bad token won't fix itself
- **Never** retry serialization errors — response won't change
- Exponential backoff: 1s → 2s → 4s → 8s → ... capped at 30s
- Max 3 retries (4 total attempts)
- ±200ms jitter to avoid thundering herd
- Give up entirely after 2 minutes

---

## 3.7 Connection to Real Spring Framework

| Simplified | Real Spring Source | Key Differences |
|-----------|-------------------|-----------------|
| `RetryPolicy` interface | `o.s.core.retry.RetryPolicy` (`RetryPolicy.java:48`) | Identical API |
| `RetryPolicy.Builder` | `o.s.core.retry.RetryPolicy.Builder` (`RetryPolicy.java:148`) | Spring also supports `includes(Collection)`, `excludes(Collection)` overloads |
| `DefaultRetryPolicy` | `o.s.core.retry.DefaultRetryPolicy` (`DefaultRetryPolicy.java:36`) | Spring uses `ExceptionTypeFilter` which walks the cause chain; we use `instanceof` for simplicity |
| `shouldRetry()` logic | `DefaultRetryPolicy.shouldRetry()` (`DefaultRetryPolicy.java:64-67`) | Spring's `ExceptionTypeFilter.match(throwable, true)` also checks nested causes |
| `Builder.build()` | `RetryPolicy.Builder.build()` (`RetryPolicy.java:462-483`) | Identical structure: detect misconfiguration, then build ExponentialBackOff |
| Mutual exclusion check | `RetryPolicy.java:465-469` | Same constraint: custom BackOff XOR delay/multiplier/etc. |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **RetryPolicy** | Strategy that answers "should we retry?" + "how long?" + "when to give up entirely?" |
| **includes/excludes** | Exception type filtering: includes = allowlist, excludes = denylist (priority) |
| **predicate** | Custom function for exception filtering beyond type matching |
| **timeout** | Maximum total elapsed time for all attempts combined |
| **Builder mutual exclusion** | Can't mix custom `BackOff` with `delay`/`multiplier` — prevents contradictions |
| **DefaultRetryPolicy** | Concrete policy combining exception filter + BackOff + timeout |

**Next: Chapter 4** — We'll build `RetryTemplate`, the execution engine that uses `RetryPolicy` to run the actual retry loop with state tracking and listener notifications.
