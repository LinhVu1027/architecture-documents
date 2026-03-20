# Chapter 15: Push Registry

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Registries hold meters in memory and expose them for reading (pull model). `SimpleMeterRegistry` is great for testing; `CompositeMeterRegistry` fans out to multiple backends. But neither proactively sends data anywhere | No mechanism to periodically push metrics to a remote backend (Datadog, InfluxDB, CloudWatch) — the application must manually call some export method | Build an abstract `PushMeterRegistry` with a `ScheduledExecutorService`, `Semaphore`-based publish guard, graceful shutdown, and randomized initial delay to prevent thundering herds |

---

## 15.1 The Integration Point: Overriding `close()` in MeterRegistry

The push registry connects to the existing system by **overriding `MeterRegistry.close()`** to add graceful shutdown behavior: stop the scheduler, do a final publish, wait for in-progress work, then delegate to `super.close()`.

**Modifying:** No existing files are modified. `PushMeterRegistry` extends `MeterRegistry` and overrides `close()`.

```java
@Override
public void close() {
    stop();                                    // 1. Stop the scheduler
    if (config.enabled() && !isClosed()) {
        publishSafelyOrSkipIfInProgress();     // 2. Final publish
        waitForInProgressScheduledPublish();   // 3. Wait for in-progress
    }
    super.close();                             // 4. Set closed flag
}
```

Two key decisions here:

1. **Why override `close()` instead of adding a shutdown hook?** JVM shutdown hooks are unreliable — they don't run on `kill -9`, and their execution order is undefined. By overriding `close()`, the lifecycle is explicit: whoever creates the registry calls `close()` when done. Spring Boot calls it automatically via `@PreDestroy`.

2. **Why check `config.enabled()` AND `!isClosed()`?** If publishing is disabled, there's no point in a final publish. The `!isClosed()` check prevents double-publishing if `close()` is called twice — after the first close, `super.close()` sets the flag, and the second call skips the publish.

This connects the **MeterRegistry lifecycle** to the **push export pipeline**. To make it work, we need:
- `PushRegistryConfig` — configures step interval, enabled flag, batch size
- `PushMeterRegistry` — abstract class with scheduler, semaphore guard, and shutdown logic

## 15.2 PushRegistryConfig — The Configuration Interface

**New file:** `src/main/java/dev/linhvu/micrometer/push/PushRegistryConfig.java`

```java
public interface PushRegistryConfig {

    PushRegistryConfig DEFAULT = new PushRegistryConfig() {};

    default Duration step() {
        return Duration.ofMinutes(1);
    }

    default boolean enabled() {
        return true;
    }

    default int batchSize() {
        return 10_000;
    }
}
```

Three design choices:

1. **Interface with default methods** — implement and override only what you need. The `DEFAULT` constant provides sensible defaults for all settings.
2. **`step()` returns `Duration`** — not milliseconds or seconds, because `Duration` is self-documenting: `Duration.ofSeconds(30)` is clearer than `30000L`.
3. **`batchSize()` for chunked exports** — backends like Datadog have request size limits. A batch size of 10,000 means the `publish()` implementation should chunk large meter lists into multiple HTTP requests.

## 15.3 PushMeterRegistry — The Scheduling Engine

**New file:** `src/main/java/dev/linhvu/micrometer/push/PushMeterRegistry.java`

### The Semaphore Guard

```java
private final Semaphore publishingSemaphore = new Semaphore(1);

void publishSafelyOrSkipIfInProgress() {
    if (publishingSemaphore.tryAcquire()) {  // non-blocking
        try {
            publish();
        } catch (Throwable e) {
            System.err.println("Unexpected exception: " + e.getMessage());
        } finally {
            publishingSemaphore.release();
        }
    }
    // else: another publish is running — skip this one
}
```

The semaphore serves two roles:
- **`tryAcquire()` (non-blocking)** — during scheduled publishing, if a publish is slow and the next scheduled call fires, the second call is silently skipped instead of blocking.
- **`acquire()` (blocking)** — during `close()`, the `waitForInProgressScheduledPublish()` method blocks until any in-progress publish completes.

### The Scheduler

```java
public void start(ThreadFactory threadFactory) {
    if (scheduledExecutorService != null) {
        stop();
    }
    if (config.enabled()) {
        scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(threadFactory);
        long stepMillis = config.step().toMillis();
        long initialDelayMillis = calculateInitialDelay();
        scheduledExecutorService.scheduleAtFixedRate(
                this::publishSafelyOrSkipIfInProgress,
                initialDelayMillis, stepMillis, TimeUnit.MILLISECONDS);
    }
}
```

### The Randomized Initial Delay

```java
long calculateInitialDelay() {
    long stepMillis = config.step().toMillis();
    long randomOffset = Math.max(0,
            (long) (stepMillis * Math.random() * 0.8) - 2);
    long offsetToStartOfNextStep = stepMillis - (clock.wallTime() % stepMillis);
    return offsetToStartOfNextStep + 2 + randomOffset;
}
```

The delay has two components:
1. **Align to next step boundary** — `offsetToStartOfNextStep` ensures the first publish happens at the start of a step window, not mid-step.
2. **Random jitter within 80%** — spreads the first publish across the first 80% of the step, preventing all instances from publishing at exactly the same moment. The 2ms minimum offset leaves room for step-based meter polling (Feature 16).

---

## 15.4 Try It Yourself

<details><summary>Challenge 1: Implement a concrete PushMeterRegistry that prints meter counts to System.out</summary>

```java
public class PrintingMeterRegistry extends PushMeterRegistry {

    public PrintingMeterRegistry(PushRegistryConfig config, Clock clock) {
        super(config, clock);
    }

    @Override
    protected void publish() {
        System.out.println("=== Publishing " + getMeters().size() + " meters ===");
        for (Meter meter : getMeters()) {
            meter.use(
                counter -> System.out.println("  COUNTER " + counter.getId().getName() + " = " + counter.count()),
                gauge -> System.out.println("  GAUGE " + gauge.getId().getName() + " = " + gauge.value()),
                timer -> System.out.println("  TIMER " + timer.getId().getName() + " count=" + timer.count()),
                ds -> System.out.println("  SUMMARY " + ds.getId().getName() + " count=" + ds.count()),
                ltt -> System.out.println("  LTT " + ltt.getId().getName() + " active=" + ltt.activeTasks()),
                fc -> System.out.println("  FC " + fc.getId().getName() + " = " + fc.count()),
                ft -> System.out.println("  FT " + ft.getId().getName() + " count=" + ft.count()),
                m -> System.out.println("  METER " + m.getId().getName())
            );
        }
    }

    // Use cumulative meter implementations (same as SimpleMeterRegistry)
    @Override protected Counter newCounter(Meter.Id id) { return new CumulativeCounter(id); }
    @Override protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> f) { return new DefaultGauge<>(id, obj, f); }
    @Override protected Timer newTimer(Meter.Id id, DistributionStatisticConfig c) { return new CumulativeTimer(id, clock, getBaseTimeUnit(), c); }
    @Override protected DistributionSummary newDistributionSummary(Meter.Id id, double s, DistributionStatisticConfig c) { return new CumulativeDistributionSummary(id, clock, s, c); }
    @Override protected LongTaskTimer newLongTaskTimer(Meter.Id id, DistributionStatisticConfig c) { return new DefaultLongTaskTimer(id, clock, getBaseTimeUnit()); }
    @Override protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> f) { return new CumulativeFunctionCounter<>(id, obj, f); }
    @Override protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj, ToLongFunction<T> cf, ToDoubleFunction<T> tf, TimeUnit u) { return new CumulativeFunctionTimer<>(id, obj, cf, tf, u, getBaseTimeUnit()); }
    @Override protected TimeUnit getBaseTimeUnit() { return TimeUnit.SECONDS; }
}
```

</details>

<details><summary>Challenge 2: Use the PushMeterRegistry with a short step to see periodic publishing</summary>

```java
PushRegistryConfig config = new PushRegistryConfig() {
    @Override public Duration step() { return Duration.ofSeconds(5); }
};

PrintingMeterRegistry registry = new PrintingMeterRegistry(config, Clock.SYSTEM);
registry.counter("http.requests", "method", "GET").increment(100);
registry.start();

// Let it run for 15 seconds — should see ~3 publish cycles
Thread.sleep(15_000);

registry.close();  // Final publish + shutdown
```

</details>

<details><summary>Challenge 3: Verify that the semaphore prevents overlapping publishes</summary>

```java
PushMeterRegistry registry = new SlowPublishRegistry(PushRegistryConfig.DEFAULT, Clock.SYSTEM);
// SlowPublishRegistry.publish() sleeps for 2 seconds

// Start two publishes concurrently
Thread t1 = new Thread(registry::publishSafelyOrSkipIfInProgress);
Thread t2 = new Thread(registry::publishSafelyOrSkipIfInProgress);
t1.start();
Thread.sleep(100); // Ensure t1 starts first
t2.start();        // This one should be skipped

t1.join();
t2.join();
// Only one publish should have executed
```

</details>

---

## 15.5 Tests

### Unit Tests

#### PushRegistryConfigTest

```java
@Test
void shouldReturnOneMinuteStep_WhenDefault() {
    assertThat(PushRegistryConfig.DEFAULT.step()).isEqualTo(Duration.ofMinutes(1));
}

@Test
void shouldAllowCustomStep_WhenOverridden() {
    PushRegistryConfig config = new PushRegistryConfig() {
        @Override
        public Duration step() { return Duration.ofSeconds(30); }
    };
    assertThat(config.step()).isEqualTo(Duration.ofSeconds(30));
    assertThat(config.enabled()).isTrue();  // Other defaults remain
}
```

#### PushMeterRegistryTest

```java
@Test
void shouldPublishOnce_WhenClosed() {
    TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
    reg.counter("test.counter").increment(5);
    reg.close();
    assertThat(publishCount.get()).isEqualTo(1);
}

@Test
void shouldNotPublishOnClose_WhenDisabled() {
    PushRegistryConfig config = new PushRegistryConfig() {
        @Override public boolean enabled() { return false; }
    };
    TestPushMeterRegistry reg = createRegistry(config);
    reg.close();
    assertThat(publishCount.get()).isEqualTo(0);
}

@Test
void shouldSkipOverlappingPublish_WhenAlreadyInProgress() throws InterruptedException {
    CountDownLatch publishStarted = new CountDownLatch(1);
    CountDownLatch publishCanProceed = new CountDownLatch(1);
    TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
    reg.setPublishAction(() -> {
        publishStarted.countDown();
        try { publishCanProceed.await(5, TimeUnit.SECONDS); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    });
    Thread slowPublish = new Thread(reg::publishSafelyOrSkipIfInProgress);
    slowPublish.start();
    publishStarted.await(5, TimeUnit.SECONDS);
    reg.publishSafelyOrSkipIfInProgress();  // Should be skipped
    publishCanProceed.countDown();
    slowPublish.join(5000);
    assertThat(publishCount.get()).isEqualTo(1);  // Only one publish
}

@Test
void shouldNotDie_WhenPublishThrowsException() {
    TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
    reg.setPublishAction(() -> { throw new RuntimeException("Boom"); });
    assertThatCode(reg::publishSafelyOrSkipIfInProgress).doesNotThrowAnyException();
}
```

### Integration Tests

```java
@Test
void shouldApplyCommonTags_WhenFilterConfigured() {
    registry.meterFilter(MeterFilter.commonTags(Tags.of("app", "myapp")));
    Counter counter = registry.counter("test.counter");
    assertThat(counter.getId().getTag("app")).isEqualTo("myapp");
}

@Test
void shouldNotPublishTwice_WhenCloseCalledMultipleTimes() {
    registry.counter("test").increment();
    registry.close();
    int firstClose = publishCount.get();
    registry.close();  // Second close
    assertThat(publishCount.get()).isEqualTo(firstClose);
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 15.6 Why This Works

> ★ **Insight** -------------------------------------------
> **Why a `Semaphore` instead of `synchronized`?** The semaphore provides two modes that `synchronized` cannot: `tryAcquire()` is **non-blocking** (returns false immediately if the lock is held), while `acquire()` is **blocking** (waits until released). During scheduled publishing, we want to skip if busy (non-blocking). During `close()`, we want to wait for completion (blocking). A `synchronized` block would block in both cases, causing the scheduler's thread pool to back up.
>
> **Trade-off:** The semaphore doesn't track which thread holds it — if a bug causes `publish()` to hang forever, `close()` will also hang forever. The real Micrometer mitigates this by logging warnings. In production, you'd add a timeout to `waitForInProgressScheduledPublish()`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Catching `Throwable`, not just `Exception`.** `publishSafelyOrSkipIfInProgress()` catches `Throwable` because `ScheduledExecutorService.scheduleAtFixedRate` has a dangerous behavior: if the task throws *any* uncaught exception, the executor silently cancels all future executions — no error log, no retry. By catching `Throwable`, we ensure that even `OutOfMemoryError` or `NoClassDefFoundError` from a buggy publish implementation doesn't kill the entire metrics pipeline. The error is logged and the next scheduled publish will still fire.
>
> **Trade-off:** Catching `Throwable` can mask serious errors. In production, you'd want to count consecutive failures and escalate (e.g., disable publishing after N failures).
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **The randomized initial delay is step-aligned, not arbitrary.** The delay formula `offsetToStartOfNextStep + 2 + randomOffset` ensures the first publish happens *within a step window*, not at a random absolute time. This matters for Step-based registries (Feature 16): StepCounter accumulates values during a step and "rolls over" at step boundaries. Publishing must happen *after* the rollover to see the completed step's data. The 2ms minimum offset ensures the publish doesn't race with the step boundary.
> -----------------------------------------------------------

## 15.7 What We Enhanced

| Aspect | Before (ch14) | Current (ch15) | Real Framework |
|--------|---------------|----------------|----------------|
| Metric export | Pull model only — meters live in memory, consumers must read them | Push model — `PushMeterRegistry` periodically calls `publish()` to export | Same — `PushMeterRegistry.java:37-173` |
| Publish scheduling | Not available | `ScheduledExecutorService` with configurable step interval | Same — `PushMeterRegistry.java:103-118` |
| Overlapping publish prevention | Not needed | `Semaphore.tryAcquire()` skips if busy | Same — `PushMeterRegistry.java:58-73` |
| Graceful shutdown | `close()` just sets a boolean flag | `close()` stops scheduler, does final publish, waits for in-progress | Same — `PushMeterRegistry.java:130-143` |
| Thundering-herd prevention | Not needed | Randomized initial delay within 80% of step | Same — `PushMeterRegistry.java:155-168`, uses `PERCENT_RANGE_OF_RANDOM_PUBLISHING_OFFSET = 0.8` |
| Configuration | Not applicable | `PushRegistryConfig` with `step()`, `enabled()`, `batchSize()` | Same pattern — `PushRegistryConfig.java:28-76`, real version adds property-binding via `MeterRegistryConfig` |

## 15.8 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `PushRegistryConfig` | `PushRegistryConfig` | `PushRegistryConfig.java:28-76` | Real extends `MeterRegistryConfig` with property-binding framework (`getDuration(this, "step")`), prefix-based key lookup, and validation via `Validated<?>`. Also includes deprecated `connectTimeout`, `readTimeout`, `numThreads` |
| `PushMeterRegistry` constructor | `PushMeterRegistry` constructor | `PushMeterRegistry.java:47-53` | Real calls `config.requireValid()` for construction-time validation |
| `publishSafelyOrSkipIfInProgress()` | `publishSafelyOrSkipIfInProgress()` | `PushMeterRegistry.java:60-73` | Real records `lastScheduledPublishStartTime` and uses `InternalLogger` for warnings |
| `start(ThreadFactory)` | `start(ThreadFactory)` | `PushMeterRegistry.java:103-118` | Real logs a `startMessage()` (customizable by subclasses) on start |
| `close()` | `close()` | `PushMeterRegistry.java:130-143` | Identical structure: stop → final publish → wait → super.close() |
| `calculateInitialDelay()` | `calculateInitialDelay()` | `PushMeterRegistry.java:155-168` | Real uses `new Random()` instead of `Math.random()`. Same formula otherwise |
| `System.err.println(...)` | `InternalLogger.warn(...)` | `PushMeterRegistry.java:63-65` | Real uses a logging facade; we use stderr for simplicity |

## 15.9 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/push/PushRegistryConfig.java` [NEW]

```java
package dev.linhvu.micrometer.push;

import java.time.Duration;

/**
 * Configuration for push-based meter registries that periodically export metrics
 * to a remote backend.
 * <p>
 * Implement this interface and override methods to customize the push behavior.
 * For default settings, use {@link #DEFAULT}:
 * <pre>{@code
 * PushMeterRegistry registry = new MyPushRegistry(PushRegistryConfig.DEFAULT, Clock.SYSTEM);
 * }</pre>
 * <p>
 * For custom settings, implement the interface:
 * <pre>{@code
 * PushRegistryConfig config = new PushRegistryConfig() {
 *     @Override
 *     public Duration step() { return Duration.ofSeconds(30); }
 * };
 * }</pre>
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code PushRegistryConfig} extends
 * {@code MeterRegistryConfig} with a property-binding framework that resolves config
 * from {@code prefix() + "." + key} lookups (e.g., {@code "datadog.step"} from
 * environment variables or properties files). We use simple default methods instead.
 */
public interface PushRegistryConfig {

    /**
     * Default configuration instance with all default values:
     * step = 1 minute, enabled = true, batchSize = 10,000.
     */
    PushRegistryConfig DEFAULT = new PushRegistryConfig() {
    };

    /**
     * The step size (reporting frequency) to use. The default is 1 minute.
     * <p>
     * This controls how often {@link PushMeterRegistry#publish()} is called
     * by the scheduled executor.
     *
     * @return The publishing interval.
     */
    default Duration step() {
        return Duration.ofMinutes(1);
    }

    /**
     * Whether publishing is enabled. When {@code false}, the scheduler is not
     * started and no final publish occurs on close.
     *
     * @return {@code true} if publishing is enabled.
     */
    default boolean enabled() {
        return true;
    }

    /**
     * The number of measurements per request to use for the backend. If more
     * measurements are found, then multiple requests will be made.
     *
     * @return The batch size. Default is 10,000.
     */
    default int batchSize() {
        return 10_000;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/push/PushMeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.push;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.MeterRegistry;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;

/**
 * Base class for registries that periodically push/export metrics to a remote
 * backend on a schedule.
 * <p>
 * The publishing lifecycle:
 * <ol>
 *   <li>{@link #start(ThreadFactory)} — creates a {@link ScheduledExecutorService}
 *       that calls {@link #publish()} at the configured {@link PushRegistryConfig#step()}</li>
 *   <li>{@link #publishSafelyOrSkipIfInProgress()} — wraps {@link #publish()} with
 *       a {@link Semaphore} guard (prevents overlapping publishes) and exception handling
 *       (prevents the scheduler from dying)</li>
 *   <li>{@link #close()} — stops the scheduler, does one final publish, waits for
 *       any in-progress publish to complete, then marks the registry as closed</li>
 * </ol>
 * <p>
 * Subclasses implement:
 * <ul>
 *   <li>{@link #publish()} — serialize and send all meters to the backend</li>
 *   <li>{@code newCounter()}, {@code newGauge()}, etc. — create backend-specific meter implementations</li>
 *   <li>{@link #getBaseTimeUnit()} — return the backend's expected time unit</li>
 * </ul>
 * <p>
 * <b>Simplification:</b> The real Micrometer uses an internal logging framework
 * ({@code InternalLogger}) for warnings. We use {@code System.err} for simplicity.
 */
public abstract class PushMeterRegistry extends MeterRegistry {

    /**
     * Schedule publishing in the beginning 80% of the step to avoid spill-over
     * into the next step. The remaining 20% provides buffer for clock skew and
     * publish duration.
     */
    private static final double PERCENT_RANGE_OF_RANDOM_PUBLISHING_OFFSET = 0.8;

    private final PushRegistryConfig config;

    /**
     * Binary semaphore that prevents overlapping publishes. {@code tryAcquire()}
     * is used during scheduled publishing (non-blocking skip if busy), while
     * {@code acquire()} is used during {@code close()} (blocking wait for
     * in-progress publish to complete).
     */
    private final Semaphore publishingSemaphore = new Semaphore(1);

    private volatile ScheduledExecutorService scheduledExecutorService;

    protected PushMeterRegistry(PushRegistryConfig config, Clock clock) {
        super(clock);
        this.config = config;
    }

    // -----------------------------------------------------------------------
    // Abstract publish method — subclasses implement the export logic
    // -----------------------------------------------------------------------

    /**
     * Serializes and sends all registered meters to the backend. Called periodically
     * by the scheduler and once on graceful shutdown.
     * <p>
     * Implementations should iterate {@link #getMeters()} and format each meter
     * for the target system. Use {@link PushRegistryConfig#batchSize()} to chunk
     * large meter lists into multiple requests.
     */
    protected abstract void publish();

    // -----------------------------------------------------------------------
    // Safe publishing with semaphore guard
    // -----------------------------------------------------------------------

    /**
     * Calls {@link #publish()} with two safety guards:
     * <ol>
     *   <li><b>Semaphore guard:</b> If another publish is already in progress,
     *       this call is skipped (non-blocking). This prevents slow publishes
     *       from causing a backlog of waiting threads.</li>
     *   <li><b>Exception handling:</b> All exceptions from {@code publish()} are
     *       caught and logged. This is critical because
     *       {@link ScheduledExecutorService#scheduleAtFixedRate} silently stops
     *       future executions if a task throws an uncaught exception.</li>
     * </ol>
     */
    void publishSafelyOrSkipIfInProgress() {
        if (publishingSemaphore.tryAcquire()) {
            try {
                publish();
            }
            catch (Throwable e) {
                System.err.println("Unexpected exception while publishing metrics for "
                        + getClass().getSimpleName() + ": " + e.getMessage());
            }
            finally {
                publishingSemaphore.release();
            }
        }
    }

    /**
     * Returns whether a publish is currently in progress.
     */
    protected boolean isPublishing() {
        return publishingSemaphore.availablePermits() == 0;
    }

    // -----------------------------------------------------------------------
    // Scheduler lifecycle: start / stop / close
    // -----------------------------------------------------------------------

    /**
     * Starts the scheduled publisher using the default thread factory.
     */
    public void start() {
        start(Executors.defaultThreadFactory());
    }

    /**
     * Starts the scheduled publisher. If publishing is already running, it is
     * stopped first. If {@link PushRegistryConfig#enabled()} is {@code false},
     * this method does nothing.
     * <p>
     * The scheduler calls {@link #publishSafelyOrSkipIfInProgress()} at a fixed
     * rate of {@link PushRegistryConfig#step()}. A randomized initial delay
     * prevents thundering-herd effects when many instances start simultaneously.
     *
     * @param threadFactory Factory for creating the scheduler thread.
     */
    public void start(ThreadFactory threadFactory) {
        if (scheduledExecutorService != null) {
            stop();
        }

        if (config.enabled()) {
            scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(threadFactory);
            long stepMillis = config.step().toMillis();
            long initialDelayMillis = calculateInitialDelay();
            scheduledExecutorService.scheduleAtFixedRate(
                    this::publishSafelyOrSkipIfInProgress,
                    initialDelayMillis, stepMillis, TimeUnit.MILLISECONDS);
        }
    }

    /**
     * Stops the scheduled publisher. Does not trigger a final publish — use
     * {@link #close()} for graceful shutdown.
     */
    public void stop() {
        if (scheduledExecutorService != null) {
            scheduledExecutorService.shutdown();
            scheduledExecutorService = null;
        }
    }

    /**
     * Graceful shutdown:
     * <ol>
     *   <li>Stop the scheduler (no more triggered publishes)</li>
     *   <li>If enabled, do one final publish (or skip if one is in progress)</li>
     *   <li>Wait for any in-progress publish to complete</li>
     *   <li>Mark the registry as closed (no-op meters for new registrations)</li>
     * </ol>
     */
    @Override
    public void close() {
        stop();
        if (config.enabled() && !isClosed()) {
            // Do a final publish on close, or skip if one is in progress
            publishSafelyOrSkipIfInProgress();
            waitForInProgressScheduledPublish();
        }
        super.close();
    }

    /**
     * Blocks until any in-progress publish completes. Used during {@link #close()}
     * to ensure data is not lost.
     */
    protected void waitForInProgressScheduledPublish() {
        try {
            // Block by acquiring the semaphore (will wait if publish is in progress),
            // then immediately release it
            publishingSemaphore.acquire();
            publishingSemaphore.release();
        }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    // -----------------------------------------------------------------------
    // Initial delay calculation
    // -----------------------------------------------------------------------

    /**
     * Calculates a randomized initial delay for the first publish.
     * <p>
     * The delay places the first publish within the first 80% of the next step
     * boundary, at least 2ms into the step. This serves two purposes:
     * <ol>
     *   <li><b>Thundering-herd prevention:</b> When many instances start at the
     *       same time, each gets a different random delay, spreading the publish
     *       load across the step interval.</li>
     *   <li><b>Step alignment:</b> The 2ms minimum offset ensures the publish runs
     *       after the step boundary has been crossed, giving step-based meters
     *       time to roll over their values.</li>
     * </ol>
     *
     * @return The initial delay in milliseconds.
     */
    long calculateInitialDelay() {
        long stepMillis = config.step().toMillis();
        // Random offset within [0, 80% of step - 2)
        long randomOffset = Math.max(0,
                (long) (stepMillis * Math.random() * PERCENT_RANGE_OF_RANDOM_PUBLISHING_OFFSET) - 2);
        // Offset to the start of the next step boundary
        long offsetToStartOfNextStep = stepMillis - (clock.wallTime() % stepMillis);
        // At least 2ms into the step
        return offsetToStartOfNextStep + 2 + randomOffset;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/push/PushRegistryConfigTest.java` [NEW]

```java
package dev.linhvu.micrometer.push;

import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link PushRegistryConfig}.
 */
class PushRegistryConfigTest {

    @Test
    void shouldReturnOneMinuteStep_WhenDefault() {
        assertThat(PushRegistryConfig.DEFAULT.step()).isEqualTo(Duration.ofMinutes(1));
    }

    @Test
    void shouldReturnEnabled_WhenDefault() {
        assertThat(PushRegistryConfig.DEFAULT.enabled()).isTrue();
    }

    @Test
    void shouldReturnTenThousandBatchSize_WhenDefault() {
        assertThat(PushRegistryConfig.DEFAULT.batchSize()).isEqualTo(10_000);
    }

    @Test
    void shouldAllowCustomStep_WhenOverridden() {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override
            public Duration step() {
                return Duration.ofSeconds(30);
            }
        };

        assertThat(config.step()).isEqualTo(Duration.ofSeconds(30));
        // Other defaults should remain
        assertThat(config.enabled()).isTrue();
        assertThat(config.batchSize()).isEqualTo(10_000);
    }

    @Test
    void shouldAllowDisabling_WhenEnabledOverridden() {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override
            public boolean enabled() {
                return false;
            }
        };

        assertThat(config.enabled()).isFalse();
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/push/PushMeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.push;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.cumulative.CumulativeCounter;
import dev.linhvu.micrometer.cumulative.CumulativeDistributionSummary;
import dev.linhvu.micrometer.cumulative.CumulativeFunctionCounter;
import dev.linhvu.micrometer.cumulative.CumulativeFunctionTimer;
import dev.linhvu.micrometer.cumulative.CumulativeTimer;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.internal.DefaultGauge;
import dev.linhvu.micrometer.internal.DefaultLongTaskTimer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link PushMeterRegistry}.
 */
public class PushMeterRegistryTest {

    private MockClock clock;
    private AtomicInteger publishCount;
    private List<String> publishLog;
    private TestPushMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        publishCount = new AtomicInteger(0);
        publishLog = Collections.synchronizedList(new ArrayList<>());
    }

    @AfterEach
    void tearDown() {
        if (registry != null && !registry.isClosed()) {
            registry.close();
        }
    }

    private TestPushMeterRegistry createRegistry(PushRegistryConfig config) {
        registry = new TestPushMeterRegistry(config, clock, publishCount, publishLog);
        return registry;
    }

    @Test
    void shouldPublishOnce_WhenClosed() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        reg.counter("test.counter").increment(5);
        reg.close();
        assertThat(publishCount.get()).isEqualTo(1);
    }

    @Test
    void shouldNotPublishOnClose_WhenDisabled() {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override public boolean enabled() { return false; }
        };
        TestPushMeterRegistry reg = createRegistry(config);
        reg.counter("test.counter").increment(5);
        reg.close();
        assertThat(publishCount.get()).isEqualTo(0);
    }

    @Test
    void shouldMarkRegistryAsClosed_WhenCloseIsCalled() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        assertThat(reg.isClosed()).isFalse();
        reg.close();
        assertThat(reg.isClosed()).isTrue();
    }

    @Test
    void shouldSkipOverlappingPublish_WhenAlreadyInProgress() throws InterruptedException {
        CountDownLatch publishStarted = new CountDownLatch(1);
        CountDownLatch publishCanProceed = new CountDownLatch(1);
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        reg.setPublishAction(() -> {
            publishStarted.countDown();
            try { publishCanProceed.await(5, TimeUnit.SECONDS); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        Thread slowPublish = new Thread(reg::publishSafelyOrSkipIfInProgress);
        slowPublish.start();
        publishStarted.await(5, TimeUnit.SECONDS);
        reg.publishSafelyOrSkipIfInProgress();
        publishCanProceed.countDown();
        slowPublish.join(5000);
        assertThat(publishCount.get()).isEqualTo(1);
    }

    @Test
    void shouldNotDie_WhenPublishThrowsException() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        reg.setPublishAction(() -> { throw new RuntimeException("Simulated publish failure"); });
        assertThatCode(reg::publishSafelyOrSkipIfInProgress).doesNotThrowAnyException();
    }

    @Test
    void shouldSchedulePublishing_WhenStarted() throws InterruptedException {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override public Duration step() { return Duration.ofMillis(50); }
        };
        TestPushMeterRegistry reg = createRegistry(config);
        reg.start(r -> { Thread t = new Thread(r); t.setDaemon(true); return t; });
        Thread.sleep(500);
        assertThat(publishCount.get()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void shouldNotStartScheduler_WhenDisabled() {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override public boolean enabled() { return false; }
        };
        TestPushMeterRegistry reg = createRegistry(config);
        reg.start();
        assertThatCode(reg::stop).doesNotThrowAnyException();
    }

    @Test
    void shouldStopScheduler_WhenStopCalled() throws InterruptedException {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override public Duration step() { return Duration.ofMillis(50); }
        };
        TestPushMeterRegistry reg = createRegistry(config);
        reg.start(r -> { Thread t = new Thread(r); t.setDaemon(true); return t; });
        Thread.sleep(200);
        reg.stop();
        int countAfterStop = publishCount.get();
        Thread.sleep(200);
        assertThat(publishCount.get()).isEqualTo(countAfterStop);
    }

    @Test
    void shouldCalculatePositiveInitialDelay_WhenCalled() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        assertThat(reg.calculateInitialDelay()).isPositive();
    }

    @Test
    void shouldCalculateDelayWithinTwoSteps_WhenCalled() {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(10); }
        };
        TestPushMeterRegistry reg = createRegistry(config);
        assertThat(reg.calculateInitialDelay()).isLessThanOrEqualTo(config.step().toMillis() * 2);
    }

    @Test
    void shouldRegisterMeters_WhenUsed() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        Counter counter = reg.counter("test.counter");
        counter.increment(5);
        Timer timer = reg.timer("test.timer");
        timer.record(100, TimeUnit.MILLISECONDS);
        assertThat(reg.getMeters()).hasSize(2);
        assertThat(counter.count()).isEqualTo(5.0);
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldReturnNoopMeters_WhenClosedAndNewMetersRegistered() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        reg.close();
        Counter counter = reg.counter("after.close");
        counter.increment(100);
        assertThat(reg.getMeters()).isEmpty();
    }

    @Test
    void shouldReportNotPublishing_WhenIdle() {
        TestPushMeterRegistry reg = createRegistry(PushRegistryConfig.DEFAULT);
        assertThat(reg.isPublishing()).isFalse();
    }

    /**
     * A concrete PushMeterRegistry for testing. Uses cumulative meter types
     * (same as SimpleMeterRegistry) and records publish calls.
     */
    public static class TestPushMeterRegistry extends PushMeterRegistry {

        private final AtomicInteger publishCount;
        private final List<String> publishLog;
        private volatile Runnable publishAction;

        public TestPushMeterRegistry(PushRegistryConfig config, Clock clock,
                AtomicInteger publishCount, List<String> publishLog) {
            super(config, clock);
            this.publishCount = publishCount;
            this.publishLog = publishLog;
        }

        void setPublishAction(Runnable action) {
            this.publishAction = action;
        }

        @Override
        protected void publish() {
            if (publishAction != null) {
                publishAction.run();
            }
            publishCount.incrementAndGet();
            publishLog.add("publish@" + System.currentTimeMillis());
        }

        @Override protected Counter newCounter(Meter.Id id) { return new CumulativeCounter(id); }
        @Override protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> f) { return new DefaultGauge<>(id, obj, f); }
        @Override protected Timer newTimer(Meter.Id id, DistributionStatisticConfig config) { return new CumulativeTimer(id, clock, getBaseTimeUnit(), config); }
        @Override protected DistributionSummary newDistributionSummary(Meter.Id id, double scale, DistributionStatisticConfig config) { return new CumulativeDistributionSummary(id, clock, scale, config); }
        @Override protected LongTaskTimer newLongTaskTimer(Meter.Id id, DistributionStatisticConfig config) { return new DefaultLongTaskTimer(id, clock, getBaseTimeUnit()); }
        @Override protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) { return new CumulativeFunctionCounter<>(id, obj, countFunction); }
        @Override protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj, ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction, TimeUnit totalTimeFunctionUnit) { return new CumulativeFunctionTimer<>(id, obj, countFunction, totalTimeFunction, totalTimeFunctionUnit, getBaseTimeUnit()); }
        @Override protected TimeUnit getBaseTimeUnit() { return TimeUnit.SECONDS; }
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/PushRegistryIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.binder.MeterBinder;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.push.PushMeterRegistryTest.TestPushMeterRegistry;
import dev.linhvu.micrometer.push.PushRegistryConfig;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for PushMeterRegistry — verifies interaction with
 * MeterFilter, MeterBinder, and graceful shutdown behavior.
 */
class PushRegistryIntegrationTest {

    private AtomicInteger publishCount;
    private List<String> publishLog;
    private TestPushMeterRegistry registry;

    @BeforeEach
    void setUp() {
        publishCount = new AtomicInteger(0);
        publishLog = Collections.synchronizedList(new ArrayList<>());
        registry = new TestPushMeterRegistry(PushRegistryConfig.DEFAULT, Clock.SYSTEM,
                publishCount, publishLog);
    }

    @AfterEach
    void tearDown() {
        if (!registry.isClosed()) {
            registry.close();
        }
    }

    @Test
    void shouldApplyCommonTags_WhenFilterConfigured() {
        registry.meterFilter(MeterFilter.commonTags(Tags.of("app", "myapp")));
        Counter counter = registry.counter("test.counter");
        counter.increment();
        assertThat(counter.getId().getTag("app")).isEqualTo("myapp");
    }

    @Test
    void shouldDenyMeters_WhenDenyFilterConfigured() {
        registry.meterFilter(MeterFilter.denyNameStartsWith("denied"));
        Counter denied = registry.counter("denied.counter");
        Counter allowed = registry.counter("allowed.counter");
        denied.increment();
        allowed.increment();
        assertThat(registry.getMeters().stream().map(m -> m.getId().getName()))
                .contains("allowed.counter")
                .doesNotContain("denied.counter");
    }

    @Test
    void shouldPublishBoundMeters_WhenBinderUsed() {
        AtomicLong queueSize = new AtomicLong(42);
        MeterBinder binder = r ->
                Gauge.builder("queue.size", queueSize, AtomicLong::doubleValue).register(r);
        binder.bindTo(registry);
        assertThat(registry.getMeters()).hasSize(1);
        registry.close();
        assertThat(publishCount.get()).isEqualTo(1);
    }

    @Test
    void shouldPublishAllMeters_WhenGracefullyClosed() {
        registry.counter("c1").increment(10);
        registry.counter("c2").increment(20);
        registry.timer("t1").record(100, TimeUnit.MILLISECONDS);
        assertThat(registry.getMeters()).hasSize(3);
        registry.close();
        assertThat(publishCount.get()).isEqualTo(1);
        assertThat(registry.isClosed()).isTrue();
    }

    @Test
    void shouldNotPublishTwice_WhenCloseCalledMultipleTimes() {
        registry.counter("test").increment();
        registry.close();
        int firstClose = publishCount.get();
        registry.close();
        assertThat(publishCount.get()).isEqualTo(firstClose);
    }

    @Test
    void shouldAcceptCustomStep_WhenConfigOverridden() {
        PushRegistryConfig config = new PushRegistryConfig() {
            @Override public Duration step() { return Duration.ofSeconds(5); }
        };
        TestPushMeterRegistry customRegistry = new TestPushMeterRegistry(
                config, Clock.SYSTEM, publishCount, publishLog);
        customRegistry.counter("test").increment();
        customRegistry.close();
        assertThat(publishCount.get()).isEqualTo(1);
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **PushMeterRegistry** | Abstract base for registries that periodically export metrics to a remote backend on a schedule |
| **PushRegistryConfig** | Configuration interface with `step()` (publish frequency), `enabled()`, and `batchSize()` |
| **`publish()`** | Abstract method subclasses implement to serialize and send all meters to their backend |
| **Semaphore guard** | `Semaphore(1)` prevents overlapping publishes — `tryAcquire()` for non-blocking skip, `acquire()` for blocking wait |
| **Graceful shutdown** | `close()` stops scheduler, does final publish, waits for in-progress, then sets closed flag |
| **Randomized initial delay** | Prevents thundering-herd when many instances start simultaneously; step-aligned with 2ms minimum offset |
| **Exception swallowing** | Catches `Throwable` from `publish()` to prevent `ScheduledExecutorService` from silently dying |

**Next: Chapter 16 — Step Registry** — Extends push registry with step/interval-based meter semantics where counts and totals reset each step for rate calculation, using `StepValue<T>` and `StepDouble` primitives.
