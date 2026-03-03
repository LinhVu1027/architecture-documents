# Chapter 6: Concurrency Throttle — Rate Limiting API Calls

## What You'll Learn
- Why concurrency limiting is more practical than token-bucket rate limiting for an SDK
- How `ConcurrencyThrottle` uses `ReentrantLock` + `Condition` for thread coordination
- BLOCK vs. REJECT policies and when to use each
- How to integrate throttling with the retry layer
- Where the throttle sits in the request pipeline

---

## 6.1 The Problem: Too Many Concurrent Requests

Consider a bot that processes incoming messages with 10 concurrent listener threads:

```
Thread-1: client.sendMessage() ──→ Zalo API
Thread-2: client.sendMessage() ──→ Zalo API
Thread-3: client.sendMessage() ──→ Zalo API      ← 10 simultaneous requests
...                                                 = instant rate limiting
Thread-10: client.sendMessage() ──→ Zalo API
```

The Zalo API will rate-limit most of these, causing retries, which create MORE requests, leading to a cascading failure known as **retry amplification**.

**What we want**: limit to N concurrent API calls. If a thread tries to make a call when N are already in flight, it either:
- **BLOCK**: waits until a slot opens up, then proceeds
- **REJECT**: immediately throws an exception

---

## 6.2 Why Not Token-Bucket Rate Limiting?

| Approach | How It Works | Why NOT for SDK |
|----------|-------------|-----------------|
| **Token Bucket** | Allow N requests per second | Requires knowing the API's rate limit precisely; limits change per endpoint; needs a timer thread |
| **Sliding Window** | Track requests in a time window | Same problems as token bucket; high memory for tracking |
| **Concurrency Limit** ✓ | Allow N simultaneous requests | Simple; self-adjusting; no timer needed; naturally limits throughput |

Concurrency limiting is self-adjusting: if requests take 100ms, N=5 allows ~50 req/sec. If requests take 1s (server is slow), N=5 allows ~5 req/sec. The throughput adapts to the server's capacity without configuration.

> ★ **Insight** ─────────────────────────────────────
> - Spring Framework made the same design choice: `ConcurrencyThrottleSupport` instead of `RateLimiter`. The reasoning is practical — concurrency control works **without knowing the API's rate limits**, which can change without notice.
> - Token-bucket is the right choice when you're the API **provider** (you know your own limits). Concurrency control is the right choice when you're the API **consumer** (you don't know theirs).
> - For cases where you DO know the rate limit, you can approximate it: if the API allows 100 req/s and each request takes ~100ms, set `concurrencyLimit = 10` (100 * 0.1 = 10 concurrent).
> ─────────────────────────────────────────────────────

---

## 6.3 The ConcurrencyThrottle Implementation

```java
package dev.linhvu.zalobot.client.throttle;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Controls concurrent access to a resource by limiting the number of
 * simultaneous operations. Thread-safe.
 *
 * <p>Usage:
 * <pre>
 * throttle.beforeAccess();
 * try {
 *     // do work
 * } finally {
 *     throttle.afterAccess();
 * }
 * </pre>
 */
public class ConcurrencyThrottle {

    /** No concurrency limit — all calls pass through immediately. */
    public static final int UNBOUNDED = -1;

    /** No concurrency allowed — all calls are blocked/rejected. */
    public static final int NO_CONCURRENCY = 0;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition slotAvailable = this.lock.newCondition();

    private int concurrencyLimit;
    private int concurrencyCount = 0;
    private ThrottlePolicy policy = ThrottlePolicy.BLOCK;

    public ConcurrencyThrottle() {
        this(UNBOUNDED);
    }

    public ConcurrencyThrottle(int concurrencyLimit) {
        this.concurrencyLimit = concurrencyLimit;
    }

    public void setConcurrencyLimit(int limit) {
        this.concurrencyLimit = limit;
    }

    public int getConcurrencyLimit() {
        return this.concurrencyLimit;
    }

    public void setPolicy(ThrottlePolicy policy) {
        this.policy = policy;
    }

    public boolean isActive() {
        return this.concurrencyLimit >= 0;
    }

    /**
     * Acquire a concurrency slot. Blocks or rejects depending on policy.
     *
     * @throws ConcurrencyRejectedException if policy is REJECT and limit reached
     */
    public void beforeAccess() {
        if (this.concurrencyLimit == UNBOUNDED) {
            return;  // No throttling
        }

        if (this.concurrencyLimit == NO_CONCURRENCY) {
            handleLimitReached();
            return;
        }

        this.lock.lock();
        try {
            while (this.concurrencyCount >= this.concurrencyLimit) {
                handleLimitReached();
            }
            this.concurrencyCount++;
        }
        finally {
            this.lock.unlock();
        }
    }

    /**
     * Release a concurrency slot, waking up one blocked thread.
     */
    public void afterAccess() {
        if (this.concurrencyLimit < 0) {
            return;  // No throttling
        }

        this.lock.lock();
        try {
            this.concurrencyCount--;
            this.slotAvailable.signal();  // Wake one waiting thread
        }
        finally {
            this.lock.unlock();
        }
    }

    private void handleLimitReached() {
        if (this.policy == ThrottlePolicy.REJECT) {
            throw new ConcurrencyRejectedException(
                "Concurrency limit " + this.concurrencyLimit + " reached");
        }
        // BLOCK: wait for a slot
        try {
            this.slotAvailable.await();
        }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ConcurrencyRejectedException(
                "Interrupted while waiting for concurrency slot");
        }
    }
}
```

### ThrottlePolicy Enum

```java
package dev.linhvu.zalobot.client.throttle;

/**
 * Policy for handling requests when the concurrency limit is reached.
 */
public enum ThrottlePolicy {
    /** Wait until a slot becomes available. */
    BLOCK,
    /** Immediately reject with {@link ConcurrencyRejectedException}. */
    REJECT
}
```

### ConcurrencyRejectedException

```java
package dev.linhvu.zalobot.client.throttle;

/**
 * Thrown when a request is rejected due to concurrency limits.
 */
public class ConcurrencyRejectedException extends RuntimeException {
    public ConcurrencyRejectedException(String message) {
        super(message);
    }
}
```

---

## 6.4 Thread Coordination Walkthrough

**Scenario**: `concurrencyLimit = 2`, BLOCK policy, 3 threads.

```
Time    Thread-A                Thread-B                Thread-C
────    ────────────────        ────────────────        ────────────────
0ms     beforeAccess()          beforeAccess()
        lock.lock()             (waiting for lock)
        count=0 < 2 ✓
        count → 1
        lock.unlock()
                                lock.lock()
                                count=1 < 2 ✓
                                count → 2
                                lock.unlock()

10ms    (doing API call)        (doing API call)        beforeAccess()
                                                        lock.lock()
                                                        count=2 >= 2
                                                        await() → BLOCKED

50ms    afterAccess()
        lock.lock()
        count → 1
        signal() → wakes C
        lock.unlock()
                                                        (wakes up)
                                                        count=1 < 2 ✓
                                                        count → 2
                                                        lock.unlock()
                                                        (doing API call)
```

> ★ **Insight** ─────────────────────────────────────
> - We use `signal()` (not `signalAll()`) to wake ONE waiting thread. This is more efficient because only one slot was freed — waking all threads would cause all but one to immediately re-block.
> - Spring's `ConcurrencyThrottleSupport` uses the same `ReentrantLock` + `Condition` pattern rather than `java.util.concurrent.Semaphore`. The advantage: we can implement both BLOCK and REJECT policies with custom logic in `onLimitReached()`. A `Semaphore` only supports blocking (via `acquire()`) or try-acquire (which doesn't wait at all).
> - **When to use REJECT over BLOCK**: Use REJECT when you'd rather fail fast than queue up (e.g., user-facing APIs with strict response time SLAs). Use BLOCK when throughput matters more than latency (e.g., background message processing).
> ─────────────────────────────────────────────────────

---

## 6.5 Integration with the Client

The throttle wraps around the retry+exchange pipeline:

```
exchangeInternal()
  │
  ├─ throttle.beforeAccess()      ← acquire slot
  │
  ├─ try {
  │    retryTemplate.execute(() -> {
  │      doExchange()              ← actual HTTP call (may retry)
  │    })
  │  }
  │
  └─ finally {
       throttle.afterAccess()     ← release slot (always, even on error)
     }
```

**Why throttle OUTSIDE retry?** Because the concurrency limit should count actual in-flight requests, not retry attempts. If we throttled inside the retry loop, a single operation retrying 3 times would consume 3 slots.

Modified `exchangeInternal()`:

```java
private <N> ZaloApiResponse<N> exchangeInternal(
        HttpMethod method, String methodPath,
        Map<String, String> headers, Object body, Class<N> clazz) {

    this.concurrencyThrottle.beforeAccess();
    try {
        return this.retryTemplate.execute(() -> {
            return doExchange(method, methodPath, headers, body, clazz);
        });
    }
    catch (RetryException e) {
        Throwable cause = e.getCause();
        if (cause instanceof ZaloBotException zbe) {
            throw zbe;
        }
        throw new ZaloBotClientException(
            "Request failed after " + e.getRetryCount() + " retries", cause);
    }
    finally {
        this.concurrencyThrottle.afterAccess();  // ← always release
    }
}
```

---

## 6.6 Connection to Real Spring Framework

| Simplified | Real Spring Source | Key Differences |
|-----------|-------------------|-----------------|
| `ConcurrencyThrottle` | `o.s.util.ConcurrencyThrottleSupport` (`ConcurrencyThrottleSupport.java:50`) | Spring uses abstract base class pattern; we use a concrete class. Spring also supports serialization. |
| `ThrottlePolicy.BLOCK` | Default `onLimitReached()` behavior (`ConcurrencyThrottleSupport.java:145`) | Identical: `await()` in a while loop |
| `ThrottlePolicy.REJECT` | Overridden `onAccessRejected()` in subclasses | Spring throws `IllegalStateException`; we throw `ConcurrencyRejectedException` |
| `ConcurrencyRejectedException` | `o.s.resilience.InvocationRejectedException` | Spring extends `RejectedExecutionException`; we extend `RuntimeException` |
| `beforeAccess()`/`afterAccess()` | `ConcurrencyThrottleSupport.beforeAccess()/afterAccess()` | Identical lock+condition pattern |
| `UNBOUNDED = -1` | `ConcurrencyThrottleSupport.UNBOUNDED_CONCURRENCY = -1` | Same sentinel value |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Concurrency Throttle** | Limits how many API calls can execute simultaneously |
| **BLOCK policy** | Wait for a slot when limit is reached (default) |
| **REJECT policy** | Fail immediately when limit is reached |
| **beforeAccess/afterAccess** | Acquire/release pattern; afterAccess MUST be in finally block |
| **signal() vs signalAll()** | Wake one thread (efficient) vs wake all (wasteful) |
| **Throttle outside retry** | One slot per logical operation, not per retry attempt |

**Next: Chapter 7** — We'll expose retry and throttle configuration through `ZaloBotClient.Builder`, making it easy for users to customize resilience settings.
