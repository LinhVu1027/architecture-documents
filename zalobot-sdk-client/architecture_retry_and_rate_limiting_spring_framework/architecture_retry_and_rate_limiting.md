# Architecture: Retry Logic & Rate Limiting

**Source:** Spring Framework 7.0 (`spring-core`, `spring-context`, `spring-aop`)
**Target:** zalobot-sdk-java

---

## Overview

Spring Framework's retry and rate limiting infrastructure works like a **postal service with automatic redelivery**: when a letter (API call) fails to reach its destination, the system automatically re-sends it after a configurable waiting period (backoff), but gives up after a certain number of attempts (retry policy). Meanwhile, the post office limits how many letters can be processed simultaneously (concurrency throttle) to prevent overwhelming the destination.

The zalobot SDK currently has a **simple ExponentialBackOff** in its listener layer, but lacks:
- Retry logic for HTTP client calls (`DefaultZaloBotClient`)
- Exception-based retry filtering (don't retry auth errors)
- Configurable retry policies
- Rate limiting / concurrency control for API calls
- Retry event observability (listeners)

This guide progressively builds these capabilities, mirroring Spring's architecture.

---

## Interface Hierarchy

```
BackOff (strategy: how long to wait)                     [spring-core/util/backoff]
├── start() → BackOffExecution
│
├── FixedBackOff
│   ├── interval: long = 5000
│   ├── maxAttempts: long = MAX_VALUE
│   └── start() → FixedBackOffExecution
│       └── nextBackOff() → interval or STOP
│
└── ExponentialBackOff
    ├── initialInterval: long = 2000
    ├── jitter: long = 0
    ├── multiplier: double = 1.5
    ├── maxInterval: long = 30000
    ├── maxElapsedTime: long = MAX_VALUE
    ├── maxAttempts: long = MAX_VALUE
    └── start() → ExponentialBackOffExecution
        └── nextBackOff() → computed interval or STOP


RetryPolicy (strategy: should we retry?)                 [spring-core/retry]
├── shouldRetry(Throwable) → boolean
├── getTimeout() → Duration
├── getBackOff() → BackOff
│
├── withDefaults() → lambda: always retry
├── withMaxRetries(long) → DefaultRetryPolicy
├── builder() → Builder
│   ├── maxRetries(long)
│   ├── timeout(Duration)
│   ├── delay(Duration)
│   ├── jitter(Duration)
│   ├── multiplier(double)
│   ├── maxDelay(Duration)
│   ├── includes(Class<? extends Throwable>...)
│   ├── excludes(Class<? extends Throwable>...)
│   ├── predicate(Predicate<Throwable>)
│   └── build() → DefaultRetryPolicy
│
└── DefaultRetryPolicy
    ├── exceptionFilter: ExceptionTypeFilter
    ├── predicate: Predicate<Throwable>
    ├── timeout: Duration
    └── backOff: BackOff


RetryOperations (execution: run with retry)              [spring-core/retry]
├── execute(Retryable<R>) → R throws RetryException
├── invoke(Supplier<R>) → R
├── invoke(Runnable)
│
└── RetryTemplate
    ├── retryPolicy: RetryPolicy
    ├── retryListener: RetryListener
    └── execute() → initial attempt → retry loop → success or exhaustion


RetryListener (observability: events)                    [spring-core/retry]
├── onRetryableExecution(policy, retryable, state)
├── beforeRetry(policy, retryable, state)
├── onRetrySuccess(policy, retryable, result)
├── onRetryFailure(policy, retryable, throwable)
├── onRetryPolicyExhaustion(policy, retryable, exception)
├── onRetryPolicyInterruption(policy, retryable, exception)
├── onRetryPolicyTimeout(policy, retryable, exception)
│
└── CompositeRetryListener
    └── delegates: List<RetryListener>


ConcurrencyThrottleSupport (rate limiting)               [spring-core/util]
├── concurrencyLimit: int = -1 (unbounded)
├── concurrencyCount: int
├── concurrencyLock: ReentrantLock
├── beforeAccess() → block or reject
├── afterAccess() → release slot
└── onLimitReached() → wait or reject
```

---

## ASCII Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        BACKOFF LAYER (spring-core/util/backoff)         │
│                                                                         │
│  ┌─────────────┐        ┌──────────────────┐                            │
│  │  «interface» │        │  «interface»      │                           │
│  │   BackOff    │───────→│  BackOffExecution │                           │
│  │  start()     │        │  nextBackOff()    │                           │
│  └──────┬───────┘        │  STOP = -1        │                           │
│         │                └──────────┬────────┘                           │
│    ┌────┴─────────┐           ┌────┴───────────────┐                    │
│    │              │           │                     │                    │
│ ┌──┴────────┐ ┌───┴──────────┐ ┌──────────────┐ ┌───┴────────────────┐ │
│ │FixedBackOff│ │ExponentialBO │ │FixedBOExec   │ │ExponentialBOExec   │ │
│ │-interval   │ │-initialIntvl │ │-currentAtmpts│ │-currentInterval    │ │
│ │-maxAttempts│ │-jitter       │ └──────────────┘ │-currentElapsedTime │ │
│ └───────────┘ │-multiplier   │                   │-attempts           │ │
│               │-maxInterval  │                   └────────────────────┘ │
│               │-maxElapsedT  │                                          │
│               │-maxAttempts  │                                          │
│               └──────────────┘                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                        RETRY LAYER (spring-core/retry)                  │
│                                                                         │
│ ┌──────────────┐    ┌───────────────┐    ┌─────────────────┐            │
│ │ «interface»   │    │ «interface»    │    │ «interface»      │           │
│ │RetryOperations│    │ RetryPolicy    │    │ Retryable<R>     │           │
│ │execute()      │    │ shouldRetry()  │    │ execute() → R    │           │
│ │invoke()       │    │ getTimeout()   │    │ getName()        │           │
│ └──────┬────────┘    │ getBackOff()   │    └─────────────────┘           │
│        │             │ builder()      │                                  │
│ ┌──────┴────────┐    └──────┬────────┘    ┌─────────────────┐           │
│ │ RetryTemplate  │           │             │ RetryException   │           │
│ │-retryPolicy    │──────────→│             │ extends Exception│           │
│ │-retryListener  │    ┌──────┴────────┐    │ impl RetryState  │           │
│ └───────────────┘    │DefaultRetryPol │    │-retryCount       │           │
│                      │-exceptionFilter│    │-exceptions       │           │
│                      │-predicate      │    └─────────────────┘           │
│                      │-timeout        │                                  │
│                      │-backOff        │    ┌─────────────────┐           │
│                      └────────────────┘    │ «interface»      │           │
│                                            │ RetryListener    │           │
│                                            │ 7 callback methods│          │
│                                            └─────────────────┘           │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                    CONCURRENCY LAYER (spring-core/util)                  │
│                                                                         │
│ ┌────────────────────────┐                                              │
│ │ConcurrencyThrottleSupp │                                              │
│ │-concurrencyLimit: int  │    ┌──────────────────────────────────┐      │
│ │-concurrencyCount: int  │    │ ConcurrencyThrottleInterceptor   │      │
│ │-concurrencyLock: Lock  │◄───│ extends ConcurrencyThrottleSupp  │      │
│ │+beforeAccess()         │    │ implements MethodInterceptor      │      │
│ │+afterAccess()          │    │ invoke(MethodInvocation)          │      │
│ │#onLimitReached()       │    └──────────────────────────────────┘      │
│ │#onAccessRejected()     │                                              │
│ └────────────────────────┘                                              │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## State Diagram

### RetryTemplate Execution States

```
                    ┌──────────────┐
                    │   CREATED    │
                    │ (configured) │
                    └──────┬───────┘
                           │ execute(retryable)
                    ┌──────▼───────┐
                    │   INITIAL    │
                    │   ATTEMPT    │
                    └──────┬───────┘
                      ┌────┴────┐
                 success     failure
                   │            │
            ┌──────▼──────┐  ┌──▼──────────────┐
            │  COMPLETED  │  │  SHOULD_RETRY?   │
            │ (return R)  │  │ policy.shouldRetry│
            └─────────────┘  └──────┬───────────┘
                               ┌────┴────┐
                           yes │         │ no
                    ┌──────────▼┐   ┌────▼─────────┐
                    │  BACKING  │   │  EXHAUSTED   │
                    │   OFF     │   │ throw RetryEx │
                    │ (sleep N) │   └──────────────┘
                    └──────┬────┘
                           │ timeout exceeded?
                      ┌────┴────┐
                   no │         │ yes
               ┌──────▼──────┐  ┌──────▼──────────┐
               │   RETRY     │  │   TIMED_OUT     │
               │   ATTEMPT   │  │ throw RetryEx   │
               └──────┬──────┘  └─────────────────┘
                 ┌────┴────┐
            success     failure
               │            │
        ┌──────▼──────┐     │
        │  COMPLETED  │     └──→ back to SHOULD_RETRY?
        │ (return R)  │
        └─────────────┘
```

### BackOffExecution States

```
  ┌──────────┐  nextBackOff()  ┌──────────────┐  nextBackOff()  ┌──────────┐
  │ INITIAL  │ ──────────────→ │ INCREMENTING │ ──────────────→ │ CAPPED   │
  │ interval │                 │ interval*mul │                 │ maxIntvl │
  └──────────┘                 └──────────────┘                 └────┬─────┘
                                                                     │
                                                        maxAttempts or maxElapsed
                                                                     │
                                                                ┌────▼─────┐
                                                                │  STOP    │
                                                                │ return -1│
                                                                └──────────┘
```

### ConcurrencyThrottle States

```
  ┌───────────────┐   beforeAccess()   ┌───────────────┐
  │    IDLE       │ ──────────────────→│   ACQUIRED    │
  │ count < limit │                    │  count++      │
  └───────────────┘                    └───────┬───────┘
                                               │ afterAccess()
        ┌──────────────┐                       │
        │   BLOCKED    │                ┌──────▼───────┐
        │ count >= limit│◄──────────── │  RELEASED    │
        │ wait on cond │                │  count--     │
        └──────┬───────┘                │  signal()    │
               │ slot freed             └──────────────┘
        ┌──────▼───────┐
        │  ACQUIRED    │
        └──────────────┘
```

---

## Flows Diagram

### Flow 1: RetryTemplate.execute() — Successful Retry

```
 Caller            RetryTemplate         RetryPolicy        BackOff          Retryable
   │                    │                    │                  │                 │
   │  execute(task)     │                    │                  │                 │
   │───────────────────→│                    │                  │                 │
   │                    │ task.execute()      │                  │                 │
   │                    │───────────────────────────────────────────────────────→│
   │                    │                    │                  │    IOException  │
   │                    │◄──────────────────────────────────────────────────────│
   │                    │                    │                  │                 │
   │                    │ shouldRetry(IOEx)   │                  │                 │
   │                    │───────────────────→│                  │                 │
   │                    │          true      │                  │                 │
   │                    │◄───────────────────│                  │                 │
   │                    │                    │                  │                 │
   │                    │                    │  getBackOff()     │                 │
   │                    │───────────────────→│──────────────────│                 │
   │                    │                    │  start()          │                 │
   │                    │                    │                  │                 │
   │                    │               nextBackOff() = 1000   │                 │
   │                    │──────────────────────────────────────→│                 │
   │                    │                    │                  │                 │
   │                    │  Thread.sleep(1000) │                  │                 │
   │                    │─────────┐          │                  │                 │
   │                    │         │ zzz...   │                  │                 │
   │                    │◄────────┘          │                  │                 │
   │                    │                    │                  │                 │
   │                    │ task.execute()      │                  │                 │
   │                    │───────────────────────────────────────────────────────→│
   │                    │                    │                  │    result: R    │
   │                    │◄──────────────────────────────────────────────────────│
   │     result: R      │                    │                  │                 │
   │◄───────────────────│                    │                  │                 │
```

### Flow 2: ConcurrencyThrottle — Block Policy

```
 Thread-A         ConcurrencyThrottle         Thread-B
   │                      │                      │
   │ beforeAccess()       │                      │
   │─────────────────────→│   count=0, limit=1   │
   │   count → 1          │                      │
   │◄─────────────────────│                      │
   │                      │                      │
   │  (doing work...)     │  beforeAccess()      │
   │                      │◄─────────────────────│
   │                      │  count=1 >= limit=1  │
   │                      │  onLimitReached()    │
   │                      │  await() → BLOCKED   │
   │                      │──────────┐           │
   │                      │          │ waiting   │
   │ afterAccess()        │          │           │
   │─────────────────────→│          │           │
   │   count → 0          │          │           │
   │   signal()           │          │           │
   │                      │◄─────────┘           │
   │                      │   count → 1          │
   │                      │─────────────────────→│
   │                      │                      │ (doing work...)
```

---

## Design Patterns

| # | Pattern | Where Used | Why Chosen |
|---|---------|-----------|------------|
| 1 | **Strategy** | `BackOff` / `RetryPolicy` | Allows swapping retry/backoff algorithms without changing execution logic. Users can plug in Fixed, Exponential, or custom strategies. |
| 2 | **Template Method** | `RetryTemplate.execute()` | Centralizes the retry loop algorithm while delegating decision points (shouldRetry, nextBackOff) to strategy objects. |
| 3 | **Builder** | `RetryPolicy.Builder` | Complex object construction with many optional parameters. Builder validates constraints (e.g., can't mix custom BackOff with delay/multiplier). |
| 4 | **Observer** | `RetryListener` | Decouples retry execution from monitoring/logging. Multiple listeners can observe retry events without modifying the core logic. |
| 5 | **Composite** | `CompositeRetryListener` | Treats a group of listeners as a single listener, simplifying RetryTemplate's API. |
| 6 | **Factory Method** | `BackOff.start()` | Each `start()` call creates a fresh stateful execution, allowing the same BackOff config to be used concurrently. |
| 7 | **Semaphore** | `ConcurrencyThrottleSupport` | Controls concurrent access using a counter + lock, more flexible than `java.util.concurrent.Semaphore` because it supports both BLOCK and REJECT policies. |

---

## Architectural Decisions

| Decision | Chosen Approach | Alternatives Considered | Why Rejected |
|----------|----------------|------------------------|--------------|
| BackOff separate from RetryPolicy | BackOff is a standalone interface; RetryPolicy composes it | Embed backoff logic directly in RetryPolicy | Violates SRP; prevents reusing BackOff in non-retry contexts (e.g., message listener recovery) |
| Stateful BackOffExecution | `BackOff.start()` creates a new stateful execution each time | Stateless BackOff with attempt counter passed in | Stateful execution is simpler for callers; avoids threading issues when BackOff config is shared |
| Exception filtering via includes/excludes | `ExceptionTypeFilter` with include/exclude sets + Predicate | Single predicate only | Include/exclude sets are more declarative and cover 90% of use cases; Predicate handles the rest |
| Concurrency via lock+condition | `ReentrantLock` + `Condition` | `java.util.concurrent.Semaphore` | Lock+Condition allows BLOCK vs REJECT policies; Semaphore only supports blocking |
| No circuit breaker in core | Omitted from spring-core | Include CircuitBreaker pattern | Too complex for core; left to libraries like Resilience4j. Rate limiting via concurrency control is simpler and sufficient for most SDK use cases |
| RetryListener as single interface | One interface with 7 default methods | Separate interfaces per event | Single interface with defaults is simpler; CompositeRetryListener handles multiple listeners |
| RetryPolicy.Builder builds ExponentialBackOff internally | Builder auto-creates ExponentialBackOff from delay/multiplier/jitter | Require users to construct BackOff manually | Convenience: most users want "delay 1s, multiply 2x" without knowing about ExponentialBackOff internals |

---

## Mapping Table: Simplified → Real Spring Source

| Simplified Concept | Real Spring Framework Source | File:Line |
|--------------------|-----------------------------|-----------|
| `BackOff` interface | `o.s.util.backoff.BackOff` | `BackOff.java:48` |
| `BackOffExecution` interface | `o.s.util.backoff.BackOffExecution` | `BackOffExecution.java:30` |
| `FixedBackOff` | `o.s.util.backoff.FixedBackOff` | `FixedBackOff.java:29` |
| `ExponentialBackOff` | `o.s.util.backoff.ExponentialBackOff` | `ExponentialBackOff.java:63` |
| Jitter computation | `ExponentialBackOffExecution.applyJitter()` | `ExponentialBackOff.java:310-320` |
| `RetryPolicy` interface | `o.s.core.retry.RetryPolicy` | `RetryPolicy.java:48` |
| `RetryPolicy.Builder` | `o.s.core.retry.RetryPolicy.Builder` | `RetryPolicy.java:148` |
| `DefaultRetryPolicy` | `o.s.core.retry.DefaultRetryPolicy` | `DefaultRetryPolicy.java:36` |
| Exception filtering | `o.s.util.ExceptionTypeFilter` | `DefaultRetryPolicy.java:56` |
| `RetryTemplate` | `o.s.core.retry.RetryTemplate` | `RetryTemplate.java:56` |
| Retry loop | `RetryTemplate.execute()` | `RetryTemplate.java:126-207` |
| Timeout check | `RetryTemplate.checkIfTimeoutExceeded()` | `RetryTemplate.java:209-228` |
| `Retryable` interface | `o.s.core.retry.Retryable` | `Retryable.java:34` |
| `RetryListener` | `o.s.core.retry.RetryListener` | `RetryListener.java:34` |
| `RetryException` | `o.s.core.retry.RetryException` | `RetryException.java:42` |
| `RetryState` | `o.s.core.retry.RetryState` | `RetryState.java` |
| `ConcurrencyThrottleSupport` | `o.s.util.ConcurrencyThrottleSupport` | `ConcurrencyThrottleSupport.java:50` |
| Semaphore-like blocking | `ConcurrencyThrottleSupport.onLimitReached()` | `ConcurrencyThrottleSupport.java:145-165` |

---

## Mapping Table: Simplified → Zalobot SDK Target

| Chapter Concept | Target Zalobot Module | Target Package |
|----------------|-----------------------|----------------|
| `BackOff` + `BackOffExecution` | `zalobot-client` | `dev.linhvu.zalobot.client.retry` |
| `FixedBackOff` | `zalobot-client` | `dev.linhvu.zalobot.client.retry` |
| `ExponentialBackOff` (upgraded) | `zalobot-client` | `dev.linhvu.zalobot.client.retry` |
| `RetryPolicy` + Builder | `zalobot-client` | `dev.linhvu.zalobot.client.retry` |
| `RetryTemplate` | `zalobot-client` | `dev.linhvu.zalobot.client.retry` |
| `RetryListener` | `zalobot-client` | `dev.linhvu.zalobot.client.retry` |
| `RetryException` | `zalobot-client` | `dev.linhvu.zalobot.client.exception` |
| Client retry integration | `zalobot-client` | `dev.linhvu.zalobot.client` (modify `DefaultZaloBotClient`) |
| `ConcurrencyThrottle` | `zalobot-client` | `dev.linhvu.zalobot.client.throttle` |
| Listener backoff migration | `zalobot-listener` | `dev.linhvu.zalobot.listener` (update imports) |

---

## What We Simplified Away

| Omitted Feature | Why |
|-----------------|-----|
| **@Retryable annotation + AOP** | The zalobot SDK is not a Spring-only library; annotation-based retry via BeanPostProcessor is Spring-specific. We use programmatic retry instead. |
| **@ConcurrencyLimit annotation** | Same reason — AOP-based concurrency limiting is Spring-specific. We provide a programmatic throttle. |
| **Reactive retry (Mono/Flux)** | The zalobot SDK uses blocking HTTP clients (JDK HttpClient, OkHttp3). Reactive retry adds unnecessary complexity. |
| **MethodRetryEvent + ApplicationEventPublisher** | Spring's event system is not available in the SDK. RetryListener provides equivalent observability. |
| **ExceptionTypeFilter with cause-chain walking** | Spring's filter walks nested `getCause()` chains. We simplify to `instanceof` checking, which covers most SDK error types. |
| **CompositeRetryListener** | Can be added later if needed. Single listener is sufficient for initial implementation. |
| **SpEL expression support in retry config** | Spring-specific; not applicable to a standalone SDK. |
| **Serialization support in ConcurrencyThrottleSupport** | SDK clients are not typically serialized. |

---

## Chapter Progression

| Chapter | Title | Key Concept |
|---------|-------|-------------|
| ch01 | The Problem | Why the current zalobot SDK needs retry and rate limiting |
| ch02 | BackOff Strategy | Extract and generalize the backoff abstraction from Spring |
| ch03 | Retry Policy | Define when to retry based on exception type and attempt count |
| ch04 | Retry Template | The execution engine that combines policy + backoff + listener |
| ch05 | Client Integration | Wire retry into `DefaultZaloBotClient.exchangeInternal()` |
| ch06 | Concurrency Throttle | Rate-limit concurrent API calls |
| ch07 | Builder & Configuration | Expose retry/throttle config via `ZaloBotClient.Builder` |
| ch08 | Listener Migration | Update `zalobot-listener` to use the new shared BackOff |
