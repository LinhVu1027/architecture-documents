# Chapter 8: Listener Migration — Unifying the BackOff Layer

## What You'll Learn
- How to migrate `zalobot-listener`'s existing `ExponentialBackOff` to the new shared abstraction
- Why the listener module should depend on `zalobot-client` for retry primitives
- How `ZaloUpdateListenerContainer` benefits from `BackOff.start()` + `STOP` signal
- The complete modified listener with the new backoff integration
- What changes for existing users (backward compatibility)

---

## 8.1 The Problem: Two ExponentialBackOff Classes

After Chapters 2-7, we now have:

```
zalobot-client/
  └── dev.linhvu.zalobot.client.retry/
      ├── BackOff              (interface)
      ├── BackOffExecution     (interface, with STOP)
      ├── FixedBackOff         (implementation)
      └── ExponentialBackOff   (implementation, with jitter + maxAttempts)

zalobot-listener/
  └── dev.linhvu.zalobot.listener/
      └── ExponentialBackOff   ← the OLD one, still here
```

The old `ExponentialBackOff` in `zalobot-listener` is:
- Less capable (no jitter, no STOP signal, no interface)
- Redundant (duplicates functionality from `zalobot-client`)
- Inconsistent (different API shape from the new one)

We need to migrate the listener to use the new `BackOff` abstraction.

---

## 8.2 Dependency Direction

```
Before:
  zalobot-listener → zalobot-client → zalobot-core

After (same direction, listener already depends on client):
  zalobot-listener → zalobot-client → zalobot-core
                     ↑
                     Now also provides BackOff/BackOffExecution
```

Since `zalobot-listener` already depends on `zalobot-client` (it uses `ZaloBotClient` to make API calls), importing the new `BackOff` classes from `zalobot-client` requires no new dependency edges. This is clean.

> ★ **Insight** ─────────────────────────────────────
> - Placing `BackOff` in `zalobot-client` rather than a separate `zalobot-retry` module is a pragmatic choice. Creating a new module adds build complexity, and backoff is only meaningful in the context of client operations. If the project grows, extracting to a shared module is always possible later — but premature module splitting is worse than premature abstraction.
> - Spring places `BackOff` in `spring-core` because it's used by both messaging (`spring-jms`, `spring-kafka`) and retry (`spring-core/retry`). In our case, `zalobot-client` is the common dependency, so it's the right home.
> ─────────────────────────────────────────────────────

---

## 8.3 Modified ZaloUpdateListenerContainer

The key change is in the `ListenerConsumer.run()` method. Instead of:

```java
// OLD: Direct mutable backoff
ExponentialBackOff backOff = new ExponentialBackOff(
    properties.getBackOffInterval(),
    properties.getMaxBackOffInterval());

// In catch block:
long backOffMillis = backOff.nextBackOffMillis();
// ... and later:
backOff.reset();
```

We now use:

```java
// NEW: BackOff.start() creates fresh execution; STOP signal ends retry
BackOff backOff = createBackOff(properties);
BackOffExecution backOffExecution = backOff.start();

// In catch block:
long backOffMillis = backOffExecution.nextBackOff();
if (backOffMillis == BackOffExecution.STOP) {
    // BackOff says "too many errors" — could stop or restart
    backOffExecution = backOff.start();  // restart the sequence
}
// ... and on success:
backOffExecution = backOff.start();  // fresh execution for next error sequence
```

Full modified `ListenerConsumer`:

```java
private final class ListenerConsumer implements Runnable {

    private final UpdateListener listener;
    private volatile boolean consumerPaused = false;

    ListenerConsumer(UpdateListener listener) {
        this.listener = listener;
    }

    @Override
    public void run() {
        ZaloUpdateListenerContainer.this.startLatch.countDown();

        ContainerProperties properties = getContainerProperties();
        Duration pollInterval = properties.getPollInterval();
        Duration pollTimeout = properties.getPollTimeout();

        ErrorHandler errorHandler = properties.getErrorHandler();
        if (errorHandler == null) {
            errorHandler = new LoggingErrorHandler();
        }

        // ═══════════════════════════════════════
        // NEW: Use BackOff interface from zalobot-client
        // ═══════════════════════════════════════
        BackOff backOff = createBackOff(properties);
        BackOffExecution backOffExecution = backOff.start();

        while (isRunning()) {
            try {
                if (isPauseRequested()) {
                    if (!this.consumerPaused) {
                        this.consumerPaused = true;
                    }
                    TimeUnit.MILLISECONDS.sleep(pollInterval.toMillis());
                    continue;
                }
                else if (this.consumerPaused) {
                    this.consumerPaused = false;
                }

                pollAndInvoke(pollTimeout);

                // ═══════════════════════════════════════
                // NEW: Reset by creating fresh execution
                // (replaces backOff.reset())
                // ═══════════════════════════════════════
                backOffExecution = backOff.start();

                TimeUnit.MILLISECONDS.sleep(pollInterval.toMillis());
            }
            catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
            catch (Exception e) {
                errorHandler.handleError(e,
                    ZaloUpdateListenerContainer.this.thisOrParentContainer);

                // ═══════════════════════════════════════
                // NEW: Use BackOffExecution with STOP
                // ═══════════════════════════════════════
                long backOffMillis = backOffExecution.nextBackOff();
                if (backOffMillis == BackOffExecution.STOP) {
                    // Max attempts reached — restart the sequence
                    // (listener containers typically don't stop permanently)
                    logger.warn("BackOff exhausted; resetting for next error cycle");
                    backOffExecution = backOff.start();
                    backOffMillis = backOffExecution.nextBackOff();
                }

                try {
                    TimeUnit.MILLISECONDS.sleep(backOffMillis);
                }
                catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }

    // pollAndInvoke() — unchanged
}

/**
 * Create a BackOff from container properties. Uses the new shared
 * ExponentialBackOff from zalobot-client.
 */
private static BackOff createBackOff(ContainerProperties properties) {
    dev.linhvu.zalobot.client.retry.ExponentialBackOff backOff =
        new dev.linhvu.zalobot.client.retry.ExponentialBackOff();
    backOff.setInitialInterval(properties.getBackOffInterval().toMillis());
    backOff.setMaxInterval(properties.getMaxBackOffInterval().toMillis());
    backOff.setMultiplier(2.0);
    // No maxAttempts limit for listener — it should keep running
    return backOff;
}
```

---

## 8.4 What Happens to the Old ExponentialBackOff?

The old `dev.linhvu.zalobot.listener.ExponentialBackOff` class should be **deprecated then removed**:

```java
/**
 * @deprecated Use {@link dev.linhvu.zalobot.client.retry.ExponentialBackOff}
 * from the zalobot-client module instead. This class will be removed in the
 * next major version.
 */
@Deprecated(since = "next", forRemoval = true)
public class ExponentialBackOff {
    // ... existing code unchanged for backward compatibility ...
}
```

If any external code references the old class, the deprecation warning guides them to migrate.

---

## 8.5 ContainerProperties Changes

The `ContainerProperties` class needs no API changes — `backOffInterval` and `maxBackOffInterval` continue to work. The translation happens in `createBackOff()`:

```
ContainerProperties                    ExponentialBackOff (new)
─────────────────                      ─────────────────────────
backOffInterval (Duration)    ──→      initialInterval (long ms)
maxBackOffInterval (Duration) ──→      maxInterval (long ms)
(hardcoded 2.0)               ──→      multiplier (double)
```

Optionally, `ContainerProperties` could be extended to expose:
- `backOffMultiplier` (currently hardcoded to 2.0)
- `backOffJitter` (new capability from the shared BackOff)
- `maxBackOffAttempts` (for STOP behavior)

But these are optional enhancements, not required for migration.

> ★ **Insight** ─────────────────────────────────────
> - The listener handles `STOP` differently from the client: when `BackOffExecution` returns `STOP`, the listener **resets and continues** (it must keep running), while the client **gives up** (individual request failure). This is a key architectural difference: listeners are long-lived processes, clients handle individual requests.
> - This is why `BackOff` and `RetryTemplate` are separate: `RetryTemplate` uses `STOP` to mean "policy exhausted, throw RetryException." The listener ignores `STOP` for the container lifecycle but could use it to trigger an alert or metric.
> ─────────────────────────────────────────────────────

---

## 8.6 Before/After Comparison

### Before (old ExponentialBackOff)
```java
ExponentialBackOff backOff = new ExponentialBackOff(
    properties.getBackOffInterval(),
    properties.getMaxBackOffInterval());

// On success:
backOff.reset();                    // manual reset, easy to forget

// On error:
long millis = backOff.nextBackOffMillis();  // always returns a value
Thread.sleep(millis);                        // never stops
```

### After (new BackOff interface)
```java
BackOff backOff = createBackOff(properties);
BackOffExecution execution = backOff.start();

// On success:
execution = backOff.start();         // fresh execution (no "forget to reset" bug)

// On error:
long millis = execution.nextBackOff();
if (millis == BackOffExecution.STOP) {  // can signal "too many errors"
    execution = backOff.start();         // restart for listener context
    millis = execution.nextBackOff();
}
Thread.sleep(millis);
```

**Improvements:**
1. **No forgotten reset**: `backOff.start()` always creates clean state
2. **STOP signal**: BackOff can signal exhaustion (useful for alerting)
3. **Interface-based**: Can swap FixedBackOff, ExponentialBackOff, or custom implementations
4. **Jitter available**: Optional jitter prevents synchronized polling across concurrent containers

---

## 8.7 Connection to Real Spring Framework

Spring's messaging modules (`spring-jms`, `spring-kafka`) use the same `BackOff`/`BackOffExecution` pattern for their listener containers:

| Spring Module | Listener Container | BackOff Usage |
|--------------|-------------------|---------------|
| `spring-jms` | `DefaultMessageListenerContainer` | `BackOff` for connection recovery |
| `spring-kafka` | `KafkaMessageListenerContainer` | `BackOff` for consumer error recovery |
| `spring-amqp` | `SimpleMessageListenerContainer` | `BackOff` for connection retry |

Our `ZaloUpdateListenerContainer` follows the same pattern.

| Simplified | Real Spring Source |
|-----------|-------------------|
| `createBackOff()` | Similar to `KafkaMessageListenerContainer` backoff configuration |
| STOP → restart | Kafka also restarts the BackOff sequence on STOP for long-lived consumers |
| `backOff.start()` on success | Same pattern: fresh execution after each successful poll cycle |

---

## 8.8 Complete File Inventory

After all 8 chapters, here's the complete set of new and modified files:

### New Files (zalobot-client)
```
zalobot-client/src/main/java/dev/linhvu/zalobot/client/
├── retry/
│   ├── BackOff.java                    (ch02)
│   ├── BackOffExecution.java           (ch02)
│   ├── FixedBackOff.java               (ch02)
│   ├── ExponentialBackOff.java         (ch02)
│   ├── RetryPolicy.java               (ch03, includes Builder)
│   ├── DefaultRetryPolicy.java         (ch03)
│   ├── Retryable.java                  (ch04)
│   ├── RetryTemplate.java             (ch04)
│   ├── RetryListener.java             (ch04)
│   ├── RetryState.java                (ch04)
│   └── RetryException.java            (ch04)
└── throttle/
    ├── ConcurrencyThrottle.java        (ch06)
    ├── ThrottlePolicy.java             (ch06)
    └── ConcurrencyRejectedException.java (ch06)
```

### Modified Files
```
zalobot-client/
  DefaultZaloBotClient.java             (ch05: add retry + throttle)
  DefaultZaloBotClientBuilder.java      (ch07: add retry/throttle config)
  ZaloBotClient.java                    (ch07: add builder methods)

zalobot-listener/
  ZaloUpdateListenerContainer.java      (ch08: use new BackOff)
  ExponentialBackOff.java               (ch08: deprecated)

zalobot-spring-boot/
  ZaloBotClientAutoConfiguration.java   (ch07: add retry properties)
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Unified BackOff** | One `BackOff` interface used by both client (retry) and listener (error recovery) |
| **STOP in listener** | Listener resets on STOP (keeps running); client gives up on STOP (request fails) |
| **createBackOff()** | Factory method translating ContainerProperties to new BackOff instance |
| **No forgotten reset** | `backOff.start()` replaces manual `reset()` — impossible to forget |
| **Deprecation path** | Old `ExponentialBackOff` deprecated, not deleted — gives users time to migrate |
| **No dependency change** | Listener already depends on client; no new module edges |

---

## Final Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          zalobot-client                                      │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────┐                │
│  │ retry/                                                   │               │
│  │  BackOff ← FixedBackOff, ExponentialBackOff (with jitter)│               │
│  │  RetryPolicy ← DefaultRetryPolicy (includes/excludes)   │               │
│  │  RetryTemplate (execute loop + listener + timeout)       │               │
│  └──────────────────────────┬──────────────────────────────┘                │
│                             │                                               │
│  ┌──────────────────────────┼──────────────────────────────┐                │
│  │ throttle/                │                               │               │
│  │  ConcurrencyThrottle (BLOCK/REJECT)                     │               │
│  └──────────────────────────┼──────────────────────────────┘                │
│                             │                                               │
│  ┌──────────────────────────▼──────────────────────────────┐                │
│  │ DefaultZaloBotClient                                     │               │
│  │  exchangeInternal() {                                    │               │
│  │    throttle.beforeAccess();                              │               │
│  │    try {                                                  │               │
│  │      retryTemplate.execute(() -> doExchange(...));       │               │
│  │    } finally {                                            │               │
│  │      throttle.afterAccess();                             │               │
│  │    }                                                      │               │
│  │  }                                                        │               │
│  └──────────────────────────────────────────────────────────┘                │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────┐                │
│  │ DefaultZaloBotClientBuilder                               │               │
│  │  .retryMaxAttempts(3)  .concurrencyLimit(10)             │               │
│  │  .retryDelay(1s)       .throttlePolicy(BLOCK)            │               │
│  │  .retryMultiplier(2.0)                                    │               │
│  └──────────────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────────────┘
                              ↑ uses BackOff
┌─────────────────────────────┴───────────────────────────────────────────────┐
│                          zalobot-listener                                    │
│                                                                             │
│  ZaloUpdateListenerContainer                                                │
│    ListenerConsumer.run() {                                                 │
│      BackOff backOff = createBackOff(properties);                          │
│      BackOffExecution exec = backOff.start();                              │
│      while (isRunning()) {                                                  │
│        try { poll(); exec = backOff.start(); }  // reset on success        │
│        catch { sleep(exec.nextBackOff()); }     // backoff on error        │
│      }                                                                      │
│    }                                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

This architecture mirrors Spring Framework's resilience patterns while being tailored to the zalobot SDK's specific needs: built-in retry with sensible defaults, exception-aware filtering, and concurrency control — all exposed through a clean Builder API.
