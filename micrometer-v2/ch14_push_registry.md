# Chapter 14: Push Registry — Backend Export

> **Feature 14** · Tier 3 · Depends on: Features 1–4 (Counter, Gauge, Timer, DistributionSummary)

After this chapter you will be able to **build a push-based meter registry** that
periodically exports metrics to an external monitoring backend. This is how Micrometer
supports systems like Datadog, InfluxDB, and OTLP — the registry handles scheduling,
lifecycle, and publish safety, and you just override `publish()`.

---

## 1. The API Contract

Before writing any implementation, let's define exactly what a client sees.

### What the client imports

```java
import simple.micrometer.api.PushMeterRegistry;
import simple.micrometer.api.PushRegistryConfig;
import simple.micrometer.api.Clock;
import simple.micrometer.internal.LoggingMeterRegistry;
```

### What the client writes

```java
// Step 1: Configure the push interval
PushRegistryConfig config = key -> switch (key) {
    case "push.step" -> "PT10S";   // publish every 10 seconds
    default -> null;               // use defaults
};

// Step 2: Create a push registry (LoggingMeterRegistry logs to stdout)
LoggingMeterRegistry registry = new LoggingMeterRegistry(config);
registry.start();   // begin scheduled publishing

// Step 3: Use it like any other MeterRegistry
Counter c = registry.counter("events");
c.increment();
// After 10s, publish() fires automatically and logs all meters

// Step 4: Graceful shutdown (stops scheduler + one final publish)
registry.close();
```

### Implementing a custom push backend

```java
public class MyBackendRegistry extends PushMeterRegistry {
    public MyBackendRegistry(PushRegistryConfig config, Clock clock) {
        super(config, clock);
    }

    @Override
    protected void publish() {
        getMeters().forEach(meter -> {
            // Send meter.getId() + meter.measure() to your backend
        });
    }

    // ... implement newCounter(), newTimer(), etc.
}
```

### Behavioral contract

| Promise | Detail |
|---------|--------|
| Scheduled publish | `start()` begins a fixed-rate scheduler that calls `publish()` every `config.step()` |
| Graceful close | `close()` stops the scheduler, does one final `publish()`, waits for completion |
| Publish safety | A `Semaphore` ensures only one `publish()` runs at a time; overlapping calls are skipped |
| Config as lambda | `PushRegistryConfig` is a `@FunctionalInterface` — supply config with a simple lambda |
| Disable support | Set `push.enabled=false` to prevent all publishing and scheduling |
| All meter types | Subclasses implement all 8 factory methods; `LoggingMeterRegistry` reuses cumulative meters |

---

## 2. Client Usage & Tests

These tests define the API's behavior — they were written **before** any implementation.

### Config via lambda with defaults

```java
@Test
void shouldCreateConfigWithLambda() {
    PushRegistryConfig config = key -> switch (key) {
        case "push.step" -> "PT10S";
        default -> null;
    };

    assertThat(config.step()).isEqualTo(Duration.ofSeconds(10));
    assertThat(config.enabled()).isTrue();
}

@Test
void shouldReturnDefaults_WhenConfigReturnsNull() {
    PushRegistryConfig config = key -> null;

    assertThat(config.step()).isEqualTo(Duration.ofMinutes(1));
    assertThat(config.enabled()).isTrue();
}
```

### Publish on close

```java
@Test
void shouldPublishAllMeters_WhenCloseIsCalled() {
    List<String> published = new ArrayList<>();
    PushRegistryConfig config = key -> null;
    var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

    Counter counter = Counter.builder("http.requests")
            .tag("method", "GET")
            .register(registry);
    counter.increment(5);

    Gauge.builder("jvm.memory", () -> 1024.0)
            .register(registry);

    registry.close();

    assertThat(published).hasSize(2);
    assertThat(published.get(0)).contains("http.requests");
    assertThat(published.get(0)).contains("COUNT=5.0");
    assertThat(published.get(1)).contains("jvm.memory");
    assertThat(published.get(1)).contains("VALUE=1024.0");
}
```

### Scheduled publishing

```java
@Test
void shouldPublishOnSchedule() throws InterruptedException {
    CopyOnWriteArrayList<String> published = new CopyOnWriteArrayList<>();
    CountDownLatch latch = new CountDownLatch(1);

    PushRegistryConfig config = key -> switch (key) {
        case "push.step" -> "PT0.1S";  // 100ms step for fast test
        default -> null;
    };
    var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, line -> {
        published.add(line);
        latch.countDown();
    });

    registry.counter("events").increment(3);
    registry.start();

    boolean fired = latch.await(2, TimeUnit.SECONDS);
    registry.close();

    assertThat(fired).isTrue();
    assertThat(published).isNotEmpty();
    assertThat(published.get(0)).contains("events");
    assertThat(published.get(0)).contains("COUNT=3.0");
}
```

### Disabled registry

```java
@Test
void shouldNotStartScheduler_WhenDisabled() {
    List<String> published = new ArrayList<>();
    PushRegistryConfig config = key -> switch (key) {
        case "push.enabled" -> "false";
        default -> null;
    };
    var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

    registry.counter("events").increment();
    registry.start();
    registry.close();

    assertThat(published).isEmpty();
}
```

### Stop prevents further publishes

```java
@Test
void shouldStopScheduler_WhenStopCalled() throws InterruptedException {
    CopyOnWriteArrayList<String> published = new CopyOnWriteArrayList<>();

    PushRegistryConfig config = key -> switch (key) {
        case "push.step" -> "PT0.05S";  // 50ms
        default -> null;
    };
    var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

    registry.counter("events").increment();
    registry.start();
    Thread.sleep(200);

    registry.stop();
    int countAfterStop = published.size();
    Thread.sleep(200);

    assertThat(published.size()).isEqualTo(countAfterStop);
}
```

---

## 3. Implementing the Call Chain

```
Client calls: registry.start() → registry.counter("x").increment() → [time passes] → publish()
  │
  ├─► [API Layer] PushRegistryConfig
  │     @FunctionalInterface — get(key) with default methods step(), enabled()
  │
  ├─► [API Layer] PushMeterRegistry
  │     Extends MeterRegistry. Manages:
  │     ├─ ScheduledExecutorService for periodic publish
  │     ├─ Semaphore for publish-safety
  │     └─ start()/stop()/close() lifecycle
  │
  └─► [Implementation Layer] LoggingMeterRegistry
        Extends PushMeterRegistry:
        ├─ Implements publish() — iterates getMeters(), logs each
        └─ Implements all newXxx() factory methods — reuses cumulative meters
```

### 3a. API Layer — PushRegistryConfig

The configuration interface follows a powerful pattern: a `@FunctionalInterface` with a
single abstract method `get(String key)`, plus `default` methods that parse specific keys.

```java
@FunctionalInterface
public interface PushRegistryConfig {

    String get(String key);

    default Duration step() {
        String v = get("push.step");
        return v != null ? Duration.parse(v) : Duration.ofMinutes(1);
    }

    default boolean enabled() {
        String v = get("push.enabled");
        return v != null ? Boolean.parseBoolean(v) : true;
    }
}
```

**Why `@FunctionalInterface`?** This is the key insight. Because there's only one abstract
method (`get`), callers can provide the entire configuration as a lambda:

```java
PushRegistryConfig config = key -> switch (key) {
    case "push.step" -> "PT10S";
    default -> null;
};
```

The default methods on `step()` and `enabled()` call `get()` and parse the result with
fallback defaults. This pattern — one lookup method plus typed default accessors — is used
throughout the real Micrometer for all backend configurations (Prometheus, Datadog, etc.).

### 3b. API Layer — PushMeterRegistry

This is the core of the feature: an abstract `MeterRegistry` subclass that adds scheduled
publishing and lifecycle management.

```java
public abstract class PushMeterRegistry extends MeterRegistry {

    private final PushRegistryConfig config;
    private final Semaphore publishingSemaphore = new Semaphore(1);
    private volatile ScheduledExecutorService scheduledExecutorService;

    protected PushMeterRegistry(PushRegistryConfig config, Clock clock) {
        super(clock);
        this.config = config;
    }

    protected abstract void publish();
```

**Why `Semaphore` instead of `synchronized`?** The semaphore serves a different purpose
than mutual exclusion. When a scheduled publish fires while the previous one is still
running (because the backend is slow), we want to **skip** the overlapping call, not
queue it. `Semaphore.tryAcquire()` returns `false` immediately if the permit is taken,
giving us non-blocking skip semantics:

```java
private void publishSafelyOrSkip() {
    if (!publishingSemaphore.tryAcquire()) {
        return; // skip — previous publish still running
    }
    try {
        publish();
    } catch (Exception e) {
        System.err.println("Error publishing metrics: " + e.getMessage());
    } finally {
        publishingSemaphore.release();
    }
}
```

With `synchronized`, a slow publish would cause the scheduler thread to block, creating
a backlog. With `tryAcquire`, slow publishes simply cause skipped intervals — much better
for a monitoring system that should observe without interfering.

**Lifecycle — `start()`, `stop()`, `close()`:**

```java
public void start() {
    if (scheduledExecutorService != null) {
        stop();
    }
    if (!config.enabled()) {
        return;
    }

    scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread t = new Thread(r, "simple-push-publish");
        t.setDaemon(true);
        return t;
    });

    long stepMillis = config.step().toMillis();
    scheduledExecutorService.scheduleAtFixedRate(
            this::publishSafelyOrSkip,
            stepMillis, stepMillis, TimeUnit.MILLISECONDS
    );
}
```

**Why daemon threads?** If the application forgets to call `close()`, daemon threads
won't prevent JVM shutdown. The scheduler thread is expendable — metrics are
observability, not critical business logic.

**Why `scheduleAtFixedRate`?** This ensures publishes happen at regular intervals
regardless of how long each publish takes. If a publish takes 3 seconds and the step
is 10 seconds, the next publish fires at the 10-second mark (not 13 seconds later).

The `close()` method implements a graceful shutdown protocol:

```java
public void close() {
    stop();                    // 1. Stop the scheduler
    if (config.enabled()) {
        publishSafelyOrSkip(); // 2. One final publish (flush remaining data)
        waitForPublish();      // 3. Block until publish completes
    }
}
```

### 3c. Implementation Layer — LoggingMeterRegistry

The concrete example registry extends `PushMeterRegistry` and provides two things:
the `publish()` implementation and the meter factory methods.

```java
public class LoggingMeterRegistry extends PushMeterRegistry {

    private final Consumer<String> loggingSink;

    public LoggingMeterRegistry(PushRegistryConfig config, Clock clock,
                                Consumer<String> loggingSink) {
        super(config, clock);
        this.loggingSink = loggingSink;
    }

    @Override
    protected void publish() {
        getMeters().stream()
                .sorted((a, b) -> a.getId().getName().compareTo(b.getId().getName()))
                .forEach(meter -> {
                    StringJoiner measurements = new StringJoiner(", ");
                    for (Measurement m : meter.measure()) {
                        measurements.add(m.getStatistic() + "=" + m.getValue());
                    }
                    loggingSink.accept(formatId(meter.getId()) + " → " + measurements);
                });
    }
```

**Why `Consumer<String>` for the sink?** This decouples "what to publish" from "where to
send it." In production, the sink would be `log::info`. In tests, we use `list::add` to
capture output for assertions — no log capture framework needed.

The meter factory methods are identical to `SimpleMeterRegistry` — both registries use
cumulative implementations:

```java
@Override
protected Counter newCounter(Meter.Id id) {
    return new CumulativeCounter(id);
}

@Override
protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionConfig) {
    return new CumulativeTimer(id, clock, distributionConfig);
}
// ... same pattern for all 8 meter types
```

---

## 4. Try It Yourself

**Challenge 1: Step-normalized counters.** In the real framework, push registries use
`StepCounter` instead of `CumulativeCounter`. A step counter tracks the count *since the
last publish* (delta), not the total. Try modifying `LoggingMeterRegistry` to track the
previous count and report the delta in `publish()`. This is the key difference between
cumulative and step-based meters.

**Challenge 2: Batch publishing.** The real `PushRegistryConfig` has a `batchSize()`
method (default 10,000). Add this to our config and modify `publish()` to process meters
in batches — useful when backends have payload size limits.

**Challenge 3: Backoff on failure.** Our `publishSafelyOrSkip` logs errors but keeps
the same schedule. Implement exponential backoff: if `publish()` throws, double the delay
(up to a maximum), and reset on success.

---

## 5. Why This Works

**The Template Method pattern drives push registries**: `PushMeterRegistry` is a textbook
Template Method — the base class owns the algorithm (schedule → acquire semaphore → call
publish → release), and subclasses provide only the variable part (`publish()`). This means
every push backend automatically gets correct scheduling, publish safety, and lifecycle
management for free.

**Config-as-lambda is a powerful extensibility pattern**: By making `PushRegistryConfig`
a `@FunctionalInterface`, Micrometer makes the simplest case trivially easy (`key -> null`
for all defaults) while supporting sophisticated implementations (lookup from properties
files, environment variables, or configuration servers) through the same interface. This
is the Strategy pattern expressed through Java's functional interface feature.

**Daemon threads are the right default for observability**: A monitoring system should
never be the reason your application fails to shut down. By using daemon threads for the
publish scheduler, a forgotten `close()` call degrades gracefully (last interval's data
may be lost) rather than catastrophically (JVM hangs on shutdown).

---

## 6. What We Enhanced

| Previous Feature | What Changed | Why |
|-----------------|-------------|-----|
| None | No existing code was modified | `PushMeterRegistry` and `PushRegistryConfig` are entirely new; `LoggingMeterRegistry` reuses existing cumulative meter implementations without changes |

Like Feature 13 (Metrics facade), Feature 14 is purely additive. It introduces a new
abstract base class and configuration interface in the API layer, plus a concrete example
in the internal layer. All existing meter types and registries remain unchanged.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Key Simplification |
|-----------------|---------------------|-------------------|
| `PushRegistryConfig` | `io.micrometer.core.instrument.push.PushRegistryConfig` | No `batchSize()`, `numThreads()`, `connectTimeout()`, `readTimeout()`, or `validate()` — we keep only `step()` and `enabled()` |
| `PushMeterRegistry` | `io.micrometer.core.instrument.push.PushMeterRegistry` | No random initial delay jitter, no `StepMeterRegistry` layer, no step-aligned scheduling |
| `LoggingMeterRegistry` | `io.micrometer.core.instrument.logging.LoggingMeterRegistry` | Extends `PushMeterRegistry` directly instead of `StepMeterRegistry`; uses cumulative meters instead of step-normalized meters; no `Printer` inner class for formatted output |

### What we skipped and why

**`StepMeterRegistry`**: In the real framework, `PushMeterRegistry → StepMeterRegistry →
LoggingMeterRegistry`. `StepMeterRegistry` adds step-normalized meters (`StepCounter`,
`StepTimer`) that track deltas per interval rather than cumulative totals. It runs a
*second* scheduler just for meter rollover. This is important for production backends
(Datadog needs "5 requests in the last 10 seconds," not "5,000 total requests") but adds
significant complexity. We skip it to focus on the scheduling and lifecycle pattern.

**Random initial delay**: The real `PushMeterRegistry` calculates an initial delay aligned
to step boundaries plus random jitter (within 80% of the step). This prevents a
thundering-herd problem when many registries start simultaneously. We use a simple
`initialDelay = stepMillis` for clarity.

**Validation**: The real `PushRegistryConfig` has a `validate()` method that catches
configuration errors (e.g., unparseable durations) early. We rely on `Duration.parse()`
throwing `DateTimeParseException` naturally.

---

## 8. Complete Code

### `[NEW]` `src/main/java/simple/micrometer/api/PushRegistryConfig.java`

```java
package simple.micrometer.api;

import java.time.Duration;

/**
 * Configuration for push-based meter registries that periodically export
 * metrics to a monitoring backend. Implemented as a functional interface
 * so that callers can supply configuration as a simple lambda.
 *
 * <p>Usage:
 * <pre>{@code
 * PushRegistryConfig config = key -> switch (key) {
 *     case "push.step" -> "PT10S";  // publish every 10 seconds
 *     default -> null;              // use defaults for everything else
 * };
 * }</pre>
 *
 * <p>When {@link #get(String)} returns {@code null} for a key, the default
 * value for that property is used.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.push.PushRegistryConfig}.
 */
@FunctionalInterface
public interface PushRegistryConfig {

    /**
     * Look up a configuration value by key. Return {@code null} to
     * indicate "use the default."
     */
    String get(String key);

    /**
     * The frequency at which metrics are published to the backend.
     * Looked up via the key {@code "push.step"}.
     *
     * @return the step duration; defaults to 1 minute
     */
    default Duration step() {
        String v = get("push.step");
        return v != null ? Duration.parse(v) : Duration.ofMinutes(1);
    }

    /**
     * Whether this registry should publish metrics at all.
     * Looked up via the key {@code "push.enabled"}.
     *
     * @return {@code true} to enable publishing; defaults to {@code true}
     */
    default boolean enabled() {
        String v = get("push.enabled");
        return v != null ? Boolean.parseBoolean(v) : true;
    }

}
```

### `[NEW]` `src/main/java/simple/micrometer/api/PushMeterRegistry.java`

```java
package simple.micrometer.api;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * Abstract base class for meter registries that periodically <em>push</em>
 * (export) their metrics to an external monitoring system. Subclasses provide
 * the concrete meter implementations and the {@link #publish()} logic; this
 * class handles the scheduling, lifecycle, and publish-safety.
 *
 * <h3>Lifecycle</h3>
 * <pre>{@code
 * registry.start();     // begin scheduled publishing
 * // ... application runs, meters record data ...
 * registry.stop();      // stop the scheduler
 * registry.close();     // stop + final publish
 * }</pre>
 *
 * <h3>Publish guard</h3>
 * A {@link Semaphore} ensures only one {@link #publish()} call is in-flight
 * at a time. If a scheduled publish fires while a previous one is still running,
 * the duplicate is <em>skipped</em> rather than queued.
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.push.PushMeterRegistry}.
 * The real framework also has {@code StepMeterRegistry} (step-normalized counters
 * that reset each interval); we skip that layer entirely and reuse cumulative
 * meter implementations.
 */
public abstract class PushMeterRegistry extends MeterRegistry {

    private final PushRegistryConfig config;

    private final Semaphore publishingSemaphore = new Semaphore(1);

    private volatile ScheduledExecutorService scheduledExecutorService;

    protected PushMeterRegistry(PushRegistryConfig config, Clock clock) {
        super(clock);
        this.config = config;
    }

    // ── Abstract ──────────────────────────────────────────────────────

    /**
     * Publish all current meter data to the monitoring backend. Called
     * periodically by the scheduler and once during {@link #close()}.
     *
     * <p>Implementations should iterate {@link #getMeters()} and export
     * each meter's measurements.
     */
    protected abstract void publish();

    // ── Lifecycle ─────────────────────────────────────────────────────

    /**
     * Starts the periodic publishing scheduler. If the registry is already
     * started, stops and restarts.
     */
    public void start() {
        if (scheduledExecutorService != null) {
            stop();
        }

        if (!config.enabled()) {
            return;
        }

        scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "simple-push-publish");
            t.setDaemon(true);
            return t;
        });

        long stepMillis = config.step().toMillis();
        scheduledExecutorService.scheduleAtFixedRate(
                this::publishSafelyOrSkip,
                stepMillis,
                stepMillis,
                TimeUnit.MILLISECONDS
        );
    }

    /**
     * Stops the publishing scheduler. Does not trigger a final publish;
     * use {@link #close()} for a graceful shutdown.
     */
    public void stop() {
        ScheduledExecutorService ses = this.scheduledExecutorService;
        if (ses != null) {
            ses.shutdown();
            this.scheduledExecutorService = null;
        }
    }

    /**
     * Gracefully shuts down: stops the scheduler, performs one final
     * publish, and waits for it to complete.
     */
    public void close() {
        stop();
        if (config.enabled()) {
            publishSafelyOrSkip();
            waitForPublish();
        }
    }

    // ── Config access ────────────────────────────────────────────────

    /**
     * Returns the push configuration.
     */
    protected PushRegistryConfig pushConfig() {
        return config;
    }

    // ── Internal ──────────────────────────────────────────────────────

    /**
     * Attempts to publish. If another publish is already in flight, this
     * call is skipped silently. All exceptions from {@link #publish()}
     * are caught and printed to stderr.
     */
    private void publishSafelyOrSkip() {
        if (!publishingSemaphore.tryAcquire()) {
            return; // skip — previous publish still running
        }
        try {
            publish();
        } catch (Exception e) {
            System.err.println("Error publishing metrics: " + e.getMessage());
        } finally {
            publishingSemaphore.release();
        }
    }

    /**
     * Blocks until any in-progress publish completes.
     */
    private void waitForPublish() {
        try {
            publishingSemaphore.acquire();
            publishingSemaphore.release();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

}
```

### `[NEW]` `src/main/java/simple/micrometer/internal/LoggingMeterRegistry.java`

```java
package simple.micrometer.internal;

import simple.micrometer.api.Clock;
import simple.micrometer.api.Counter;
import simple.micrometer.api.DistributionStatisticConfig;
import simple.micrometer.api.DistributionSummary;
import simple.micrometer.api.FunctionCounter;
import simple.micrometer.api.FunctionTimer;
import simple.micrometer.api.Gauge;
import simple.micrometer.api.LongTaskTimer;
import simple.micrometer.api.Measurement;
import simple.micrometer.api.Meter;
import simple.micrometer.api.PushMeterRegistry;
import simple.micrometer.api.PushRegistryConfig;
import simple.micrometer.api.TimeGauge;
import simple.micrometer.api.Timer;

import java.util.StringJoiner;
import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A concrete push-based registry that logs all meter values to a configurable
 * sink on each publish. Useful for debugging, development, and demonstrating
 * the push-registry pattern.
 *
 * <p>Example:
 * <pre>{@code
 * var config = PushRegistryConfig.DEFAULT;
 * var registry = new LoggingMeterRegistry(config);
 * registry.start();
 *
 * Counter c = registry.counter("events");
 * c.increment(5);
 * // After one step interval, publish() fires and logs:
 * //   events → Measurement{COUNT=5.0}
 *
 * registry.close();
 * }</pre>
 *
 * <p>Simplified from {@code io.micrometer.core.instrument.logging.LoggingMeterRegistry}.
 * The real version extends {@code StepMeterRegistry} (step-normalized meters);
 * we extend {@link PushMeterRegistry} directly and use cumulative meters.
 */
public class LoggingMeterRegistry extends PushMeterRegistry {

    private final Consumer<String> loggingSink;

    /**
     * Creates a logging registry with the given config and default sink (stdout).
     */
    public LoggingMeterRegistry(PushRegistryConfig config) {
        this(config, Clock.SYSTEM, System.out::println);
    }

    /**
     * Creates a logging registry with a custom clock and logging sink.
     */
    public LoggingMeterRegistry(PushRegistryConfig config, Clock clock, Consumer<String> loggingSink) {
        super(config, clock);
        this.loggingSink = loggingSink;
    }

    // ── Publish ───────────────────────────────────────────────────────

    @Override
    protected void publish() {
        getMeters().stream()
                .sorted((a, b) -> a.getId().getName().compareTo(b.getId().getName()))
                .forEach(meter -> {
                    StringJoiner measurements = new StringJoiner(", ");
                    for (Measurement m : meter.measure()) {
                        measurements.add(m.getStatistic() + "=" + m.getValue());
                    }
                    loggingSink.accept(formatId(meter.getId()) + " → " + measurements);
                });
    }

    // ── Meter factory methods (reuse cumulative implementations) ─────

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new CumulativeCounter(id);
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return new DefaultGauge<>(id, obj, valueFunction);
    }

    @Override
    protected Timer newTimer(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        return new CumulativeTimer(id, clock, distributionConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, DistributionStatisticConfig distributionConfig) {
        return new CumulativeDistributionSummary(id, clock, distributionConfig);
    }

    @Override
    protected LongTaskTimer newLongTaskTimer(Meter.Id id) {
        return new DefaultLongTaskTimer(id, clock);
    }

    @Override
    protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj, ToDoubleFunction<T> countFunction) {
        return new CumulativeFunctionCounter<>(id, obj, countFunction);
    }

    @Override
    protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit) {
        return new CumulativeFunctionTimer<>(id, obj, countFunction, totalTimeFunction,
                totalTimeFunctionUnit, TimeUnit.NANOSECONDS);
    }

    @Override
    protected <T> TimeGauge newTimeGauge(Meter.Id id, T obj, TimeUnit fUnits, ToDoubleFunction<T> f) {
        return new DefaultTimeGauge<>(id, obj, fUnits, f);
    }

    // ── Internal ──────────────────────────────────────────────────────

    private String formatId(Meter.Id id) {
        StringBuilder sb = new StringBuilder(id.getName());
        var tags = id.getTags();
        if (!tags.isEmpty()) {
            sb.append('{');
            for (int i = 0; i < tags.size(); i++) {
                if (i > 0) sb.append(',');
                sb.append(tags.get(i).getKey()).append('=').append(tags.get(i).getValue());
            }
            sb.append('}');
        }
        return sb.toString();
    }

}
```

### `[NEW]` `src/test/java/simple/micrometer/api/PushMeterRegistryTest.java`

```java
package simple.micrometer.api;

import simple.micrometer.internal.LoggingMeterRegistry;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for the Push Registry API.
 *
 * <p>These tests exercise PushMeterRegistry exactly as a framework user
 * would — through the public API surface. A user creates a push registry
 * (like LoggingMeterRegistry), registers meters, and relies on scheduled
 * publishing to export data.
 */
class PushMeterRegistryTest {

    // ── Config as lambda ──────────────────────────────────────────────

    @Test
    void shouldCreateConfigWithLambda() {
        PushRegistryConfig config = key -> switch (key) {
            case "push.step" -> "PT10S";
            default -> null;
        };

        assertThat(config.step()).isEqualTo(Duration.ofSeconds(10));
        assertThat(config.enabled()).isTrue();
    }

    @Test
    void shouldReturnDefaults_WhenConfigReturnsNull() {
        PushRegistryConfig config = key -> null;

        assertThat(config.step()).isEqualTo(Duration.ofMinutes(1));
        assertThat(config.enabled()).isTrue();
    }

    @Test
    void shouldDisablePublishing_WhenEnabledIsFalse() {
        PushRegistryConfig config = key -> switch (key) {
            case "push.enabled" -> "false";
            default -> null;
        };

        assertThat(config.enabled()).isFalse();
    }

    // ── LoggingMeterRegistry as concrete push registry ────────────────

    @Test
    void shouldPublishAllMeters_WhenCloseIsCalled() {
        List<String> published = new ArrayList<>();
        PushRegistryConfig config = key -> null;
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        // Register meters
        Counter counter = Counter.builder("http.requests")
                .tag("method", "GET")
                .register(registry);
        counter.increment(5);

        Gauge.builder("jvm.memory", () -> 1024.0)
                .register(registry);

        // close() triggers a final publish
        registry.close();

        assertThat(published).hasSize(2);
        assertThat(published.get(0)).contains("http.requests");
        assertThat(published.get(0)).contains("COUNT=5.0");
        assertThat(published.get(1)).contains("jvm.memory");
        assertThat(published.get(1)).contains("VALUE=1024.0");
    }

    @Test
    void shouldFormatTagsInPublishOutput() {
        List<String> published = new ArrayList<>();
        PushRegistryConfig config = key -> null;
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        Counter.builder("http.requests")
                .tag("method", "GET")
                .tag("status", "200")
                .register(registry)
                .increment();

        registry.close();

        assertThat(published).hasSize(1);
        assertThat(published.get(0)).contains("http.requests{method=GET,status=200}");
    }

    @Test
    void shouldPublishTimerMeasurements() {
        List<String> published = new ArrayList<>();
        PushRegistryConfig config = key -> null;
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        Timer timer = Timer.builder("http.latency")
                .register(registry);
        timer.record(Duration.ofMillis(150));

        registry.close();

        assertThat(published).hasSize(1);
        String output = published.get(0);
        assertThat(output).contains("http.latency");
        assertThat(output).contains("COUNT=1.0");
        assertThat(output).contains("TOTAL_TIME=");
        assertThat(output).contains("MAX=");
    }

    @Test
    void shouldNotStartScheduler_WhenDisabled() {
        List<String> published = new ArrayList<>();
        PushRegistryConfig config = key -> switch (key) {
            case "push.enabled" -> "false";
            default -> null;
        };
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        registry.counter("events").increment();
        registry.start(); // should be a no-op when disabled

        // close() with disabled config should not publish
        registry.close();

        assertThat(published).isEmpty();
    }

    @Test
    void shouldPublishOnSchedule() throws InterruptedException {
        CopyOnWriteArrayList<String> published = new CopyOnWriteArrayList<>();
        CountDownLatch latch = new CountDownLatch(1);

        PushRegistryConfig config = key -> switch (key) {
            case "push.step" -> "PT0.1S";  // 100ms step for fast test
            default -> null;
        };
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, line -> {
            published.add(line);
            latch.countDown();
        });

        registry.counter("events").increment(3);
        registry.start();

        // Wait for at least one scheduled publish
        boolean fired = latch.await(2, TimeUnit.SECONDS);
        registry.close();

        assertThat(fired).isTrue();
        assertThat(published).isNotEmpty();
        assertThat(published.get(0)).contains("events");
        assertThat(published.get(0)).contains("COUNT=3.0");
    }

    @Test
    void shouldStopScheduler_WhenStopCalled() throws InterruptedException {
        CopyOnWriteArrayList<String> published = new CopyOnWriteArrayList<>();

        PushRegistryConfig config = key -> switch (key) {
            case "push.step" -> "PT0.05S";  // 50ms
            default -> null;
        };
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        registry.counter("events").increment();
        registry.start();

        // Let a couple publishes fire
        Thread.sleep(200);

        registry.stop();
        int countAfterStop = published.size();

        // Wait more — no new publishes should fire
        Thread.sleep(200);

        assertThat(published.size()).isEqualTo(countAfterStop);
    }

    @Test
    void shouldSupportMultipleMeterTypes() {
        List<String> published = new ArrayList<>();
        PushRegistryConfig config = key -> null;
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        registry.counter("requests").increment(10);
        Gauge.builder("queue.size", () -> 42.0).register(registry);
        Timer.builder("latency").register(registry).record(Duration.ofMillis(100));
        DistributionSummary.builder("payload.size").register(registry).record(256);

        registry.close();

        assertThat(published).hasSize(4);
        // Sorted alphabetically by name
        assertThat(published.get(0)).contains("latency");
        assertThat(published.get(1)).contains("payload.size");
        assertThat(published.get(2)).contains("queue.size");
        assertThat(published.get(3)).contains("requests");
    }

    @Test
    void shouldNotPublishToEmptyRegistry() {
        List<String> published = new ArrayList<>();
        PushRegistryConfig config = key -> null;
        var registry = new LoggingMeterRegistry(config, Clock.SYSTEM, published::add);

        registry.close();

        assertThat(published).isEmpty();
    }

}
```
