# Chapter 1: The Problem — Why the Zalobot SDK Needs Retry & Rate Limiting

## What You'll Learn
- Three failure modes in the current zalobot SDK that have no recovery strategy
- Why the existing `ExponentialBackOff` in `zalobot-listener` is insufficient
- What Spring Framework's resilience infrastructure provides that we're missing
- The gap analysis that drives the rest of this guide

---

## 1.1 The Current State: No Retry in the Client

Open `DefaultZaloBotClient.java:113-154` — the `exchangeInternal()` method. This is the **single gateway** for every API call in the SDK:

```java
// DefaultZaloBotClient.java:113-154 (current implementation)
private <N> ZaloApiResponse<N> exchangeInternal(
        HttpMethod method, String methodPath,
        Map<String, String> headers, Object body, Class<N> clazz) {

    URI uri = buildUri(methodPath);
    ClientHttpRequest request = this.clientHttpRequestFactory.createRequest(uri, method);
    // ... serialize body ...

    try (ClientHttpResponse response = request.execute()) {  // ← ONE attempt. That's it.
        // ... deserialize response ...
        if (apiResponse != null && !apiResponse.ok()) {
            ZaloErrorCode code = ZaloErrorCode.fromCode(apiResponse.errorCode());
            if (code.isAuthenticationError()) {
                throw new ZaloBotAuthenticationException(...);
            }
            throw new ZaloBotApiException(...);
        }
        return apiResponse;
    }
    catch (ZaloBotException e) {
        throw e;                    // ← API errors: propagated immediately
    }
    catch (IOException e) {
        throw new ZaloBotClientException("HTTP request failed: " + e.getMessage(), e);
        // ↑ Network errors: ALSO propagated immediately. No retry. No backoff.
    }
}
```

**Problem #1: Transient network failures are fatal.**
If the network hiccups for 200ms, the `sendMessage()` call fails permanently. The caller gets a `ZaloBotClientException` and must implement their own retry — something most users won't do correctly.

**Problem #2: Rate-limited responses are treated as permanent errors.**
Zalo's API returns error codes for rate limiting (`ZaloErrorCode` includes rate-limit codes). The current code throws `ZaloBotApiException` for ALL non-OK responses. A rate-limited call should back off and retry, not fail.

**Problem #3: No concurrency control.**
If a user creates 100 threads sending messages, all 100 hit the API simultaneously. This virtually guarantees rate limiting, which (per Problem #2) causes all 100 to fail.

---

## 1.2 The Existing BackOff: Listener-Only

The SDK does have an `ExponentialBackOff` — but it lives in `zalobot-listener`:

```java
// zalobot-listener: ExponentialBackOff.java (current)
public class ExponentialBackOff {
    private final long initialIntervalMillis;
    private final long maxIntervalMillis;
    private final double multiplier;
    private long currentIntervalMillis;  // ← mutable state, not thread-safe

    public long nextBackOffMillis() { ... }
    public void reset() { ... }
}
```

This class has three limitations:

| Limitation | Impact |
|-----------|--------|
| **Lives in `zalobot-listener`** | Cannot be used by `zalobot-client` (would create a circular dependency) |
| **Mutable + not thread-safe** | Cannot be shared across concurrent retry sessions |
| **No "stop" signal** | Always returns a value; never signals "give up" |
| **No jitter** | Multiple retrying clients will synchronize their retries (thundering herd) |

---

## 1.3 What Spring Framework Provides

Spring Framework 7.0 introduced a complete retry infrastructure in `spring-core`:

```
Spring's Resilience Stack
─────────────────────────
spring-core/util/backoff/
  BackOff            ← strategy interface (HOW LONG to wait)
  BackOffExecution   ← stateful execution (tracks attempts, returns STOP)
  FixedBackOff       ← constant delay
  ExponentialBackOff ← growing delay with jitter, max interval, max attempts

spring-core/retry/
  RetryPolicy        ← strategy interface (SHOULD we retry?)
  RetryTemplate      ← execution engine (runs the retry loop)
  RetryListener      ← observability (events during retry)
  RetryException     ← terminal failure (carries all exceptions)
  Retryable<R>       ← the operation to retry

spring-core/util/
  ConcurrencyThrottleSupport ← semaphore-like rate limiting
```

**The key insight**: Spring separates three concerns that our SDK currently conflates:
1. **Should we retry?** → `RetryPolicy` (exception filtering, attempt counting)
2. **How long to wait?** → `BackOff` (fixed, exponential, jittered)
3. **Execute with retry** → `RetryTemplate` (the loop, the state, the events)

---

## 1.4 Gap Analysis

| Capability | Spring Framework | Zalobot SDK (Current) | Gap |
|-----------|-----------------|----------------------|-----|
| Backoff abstraction | `BackOff` interface + `BackOffExecution` | Concrete `ExponentialBackOff` in listener module | Need interface + factory in client module |
| Fixed backoff | `FixedBackOff` | None | Missing |
| Exponential with jitter | `ExponentialBackOff` with jitter field | `ExponentialBackOff` without jitter | Need jitter support |
| Stop signal | `BackOffExecution.STOP = -1` | None (always returns interval) | Need max attempts / stop |
| Retry policy | `RetryPolicy` with includes/excludes/predicate | None | Missing entirely |
| Exception filtering | Don't retry `ZaloBotAuthenticationException` | Retries everything or nothing | Need exception-type filtering |
| Retry execution | `RetryTemplate` | None (client has single attempt) | Missing entirely |
| Retry events | `RetryListener` with 7 callbacks | `ErrorHandler` in listener (different purpose) | Missing for client |
| Concurrency control | `ConcurrencyThrottleSupport` | None | Missing entirely |
| Builder integration | `RetryPolicy.Builder` | `ZaloBotClient.Builder` (no retry config) | Need retry config in builder |

---

## 1.5 The Plan

Over the next 7 chapters, we'll build the missing pieces bottom-up:

```
Chapter 2:  BackOff strategy layer
Chapter 3:  RetryPolicy (exception filtering + attempt limits)
Chapter 4:  RetryTemplate (execution engine)
Chapter 5:  Wire into DefaultZaloBotClient
Chapter 6:  ConcurrencyThrottle
Chapter 7:  Expose via ZaloBotClient.Builder
Chapter 8:  Migrate zalobot-listener to use shared BackOff
```

Each chapter produces compilable code and maps directly to Spring Framework source.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Transient failure** | A temporary error (network timeout, rate limit) that would succeed if retried |
| **Permanent failure** | An error (bad token, invalid request) that will never succeed regardless of retries |
| **BackOff** | The delay strategy between retry attempts |
| **Retry policy** | The rules for deciding whether to retry (which exceptions, how many attempts) |
| **Concurrency throttle** | Limiting how many API calls can execute simultaneously |
| **Thundering herd** | When many clients retry at the same time, overwhelming the server |

**Next: Chapter 2** — We'll build the `BackOff` / `BackOffExecution` abstraction, porting Spring's pattern into the zalobot-client module.
