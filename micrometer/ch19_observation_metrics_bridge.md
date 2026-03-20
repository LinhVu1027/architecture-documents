# Chapter 19: Observation-Metrics Bridge

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| 18 chapters complete: a full metrics infrastructure (ch1-17) and a standalone Observation API (ch18) with lifecycle handlers, key values, conventions, and filters. But the Observation API and the Metrics API are two parallel universes — observations produce no metrics. | Calling `Observation.start()` and `Observation.stop()` notifies handlers, but no handler exists that translates these lifecycle events into actual Timer, Counter, or LongTaskTimer recordings. The "instrument once" promise is hollow without the bridge. | Build a `DefaultMeterObservationHandler` that implements `ObservationHandler` and automatically translates observation lifecycle events into metric recordings: `onStart()` → Timer.Sample + LongTaskTimer, `onStop()` → stop both with error tag, `onEvent()` → increment Counter. |

---

## 19.1 The Integration Point: `ObservationHandler` Meets `MeterRegistry`

This feature connects two subsystems that have existed independently until now:
- The **Observation API** (ch18): lifecycle events (`start`/`stop`/`event`) with key values
- The **Metrics API** (ch1-17): Timer, LongTaskTimer, Counter, and MeterRegistry

The integration point is a **new implementation** of the existing `ObservationHandler<Observation.Context>` interface. Unlike previous features that modified existing files, this one is entirely additive — a new class that bridges both worlds:

**New file:** `src/main/java/dev/linhvu/micrometer/observation/DefaultMeterObservationHandler.java`

```java
public class DefaultMeterObservationHandler implements ObservationHandler<Observation.Context> {

    private final MeterRegistry meterRegistry;

    public DefaultMeterObservationHandler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void onStart(Observation.Context context) {
        // Start LongTaskTimer for in-flight tracking
        // Start Timer.Sample for duration measurement
        // Store both in the context's generic map
    }

    @Override
    public void onStop(Observation.Context context) {
        // Retrieve Timer.Sample, stop against a Timer with error tag
        // Retrieve LongTaskTimer.Sample, stop it
    }

    @Override
    public void onEvent(Observation.Event event, Observation.Context context) {
        // Increment a Counter named <observation>.<event>
    }

    @Override
    public boolean supportsContext(Observation.Context context) {
        return true;  // handles all observation types
    }
}
```

**Direction:** This handler connects `ObservationHandler` lifecycle → `MeterRegistry` meter creation. The `Observation.Context` serves as the shared data carrier — the handler stores meter-specific state (Timer.Sample, LongTaskTimer.Sample) into the context's generic `put()`/`get()` map during `onStart()`, then retrieves it during `onStop()`. To make this work, we need to implement:

- The lifecycle translation: `onStart()` → start timing, `onStop()` → stop and record, `onEvent()` → count
- Tag conversion: `KeyValue` (observation world) → `Tag` (metrics world), low-cardinality only
- Error handling: an `error` tag added at stop time
- Optional LongTaskTimer suppression via `IgnoredMeters` enum

Two key decisions:

1. **Why store samples in the Context, not as handler fields?** Multiple observations can be active concurrently on different threads. If the handler stored the Timer.Sample as a field, it would be overwritten by concurrent observations. The Context is per-observation, so each observation gets its own isolated storage.

2. **Why `supportsContext()` returns `true` unconditionally?** This is a general-purpose metrics handler — it should produce metrics for ANY observation, not just specific context subtypes. Specialized handlers (e.g., for HTTP-specific metrics) would narrow this to their context type.

## 19.2 The `onStart()` Method — Starting Timing Instruments

When an observation starts, we need to begin two kinds of timing:

```java
@Override
public void onStart(Observation.Context context) {
    // Start the LongTaskTimer to track in-flight observations.
    // Created at start time — can only use tags available NOW.
    if (shouldCreateLongTaskTimer) {
        LongTaskTimer.Sample longTaskSample = LongTaskTimer
                .builder(context.getName() + ".active")
                .tags(createTags(context))
                .register(meterRegistry)
                .start();
        context.put(LongTaskTimer.Sample.class, longTaskSample);
    }

    // Start a Timer.Sample to measure total observation duration.
    // The sample captures the current monotonic time.
    Timer.Sample sample = Timer.start(meterRegistry);
    context.put(Timer.Sample.class, sample);
}
```

The **LongTaskTimer** is named `<observation>.active` (e.g., `http.requests.active`) and tracks currently in-flight observations. It must be created here because it needs to be *running* to report active tasks.

The **Timer.Sample** just captures the current monotonic time — the actual Timer doesn't exist yet. It will be created at `onStop()` when we know the full tag set (including the `error` tag).

Note the asymmetry: the LongTaskTimer gets whatever tags exist at start time, while the Timer gets all tags at stop time. This is an inherent trade-off — the LongTaskTimer must exist from the start to report active tasks.

## 19.3 The `onStop()` and `onEvent()` Methods — Recording Results

```java
@Override
public void onStop(Observation.Context context) {
    // Build tags from low-cardinality key values (available at stop time,
    // so we get all key values including those added after start).
    List<Tag> tags = createTags(context);

    // Add the error tag — either the exception's simple class name or "none".
    tags.add(Tag.of("error", getErrorValue(context)));

    // Stop the Timer.Sample against a Timer named after the observation.
    Timer.Sample sample = context.getRequired(Timer.Sample.class);
    sample.stop(meterRegistry.timer(context.getName(), tags));

    // Stop the LongTaskTimer to remove this observation from the active set.
    if (shouldCreateLongTaskTimer) {
        LongTaskTimer.Sample longTaskSample = context.getRequired(LongTaskTimer.Sample.class);
        longTaskSample.stop();
    }
}

@Override
public void onEvent(Observation.Event event, Observation.Context context) {
    // Each event increments a counter named <observation>.<event>.
    Counter.builder(context.getName() + "." + event.getName())
            .tags(createTags(context))
            .register(meterRegistry)
            .increment();
}
```

The `createTags()` helper converts low-cardinality key values to Micrometer Tags:

```java
private List<Tag> createTags(Observation.Context context) {
    List<Tag> tags = new ArrayList<>();
    for (KeyValue keyValue : context.getLowCardinalityKeyValues()) {
        tags.add(Tag.of(keyValue.getKey(), keyValue.getValue()));
    }
    return tags;
}
```

Critical detail: **only low-cardinality key values become metric tags**. High-cardinality values (user IDs, request URLs) are completely excluded — they would create an unbounded number of time series, overwhelming monitoring backends.

The `error` tag uses the exception's simple class name (e.g., `"IllegalArgumentException"`) or `"none"` if no error occurred. This gives you error-rate breakdowns without cardinality explosion (there's a finite number of exception classes).

## 19.4 The `IgnoredMeters` Opt-Out

Some applications don't need the LongTaskTimer overhead. An `IgnoredMeters` enum allows selective suppression:

```java
public DefaultMeterObservationHandler(MeterRegistry meterRegistry,
        IgnoredMeters... metersToIgnore) {
    this.meterRegistry = meterRegistry;
    this.shouldCreateLongTaskTimer = Arrays.stream(metersToIgnore)
            .noneMatch(ignored -> ignored == IgnoredMeters.LONG_TASK_TIMER);
}

public enum IgnoredMeters {
    LONG_TASK_TIMER
}
```

This is designed as an enum (not a boolean) so it can be extended to support disabling other meter types in the future.

## 19.5 Try It Yourself

<details>
<summary>Challenge: Implement the DefaultMeterObservationHandler from scratch</summary>

Given these requirements:
1. Implement `ObservationHandler<Observation.Context>`
2. On start: begin a `Timer.Sample` and a `LongTaskTimer` (named `<name>.active`)
3. On stop: stop both, adding an `error` tag to the Timer
4. On event: increment a `Counter` named `<name>.<event-name>`
5. Only use low-cardinality key values as tags

Try writing it yourself, then compare with the solution in `src/main/java/dev/linhvu/micrometer/observation/DefaultMeterObservationHandler.java`.

Hints:
- Use `context.put(Timer.Sample.class, sample)` to store the sample — the Class object makes a convenient map key
- Use `context.getRequired(Timer.Sample.class)` to retrieve it (throws if missing)
- `context.getLowCardinalityKeyValues()` returns `KeyValues` (Iterable) — iterate and convert to `Tag`
- Return a **mutable** `ArrayList` from your tag-building method so `onStop` can append the error tag

</details>

<details>
<summary>Challenge: Wire the handler into an ObservationRegistry</summary>

Write a test that:
1. Creates a `SimpleMeterRegistry` and an `ObservationRegistry`
2. Registers a `DefaultMeterObservationHandler` on the observation registry
3. Runs `Observation.createNotStarted("test", registry).observe(() -> doSomething())`
4. Asserts that a Timer named `"test"` exists with `count() == 1`

```java
@Test
void shouldBridgeObservationToMetrics() {
    SimpleMeterRegistry meterRegistry = new SimpleMeterRegistry();
    ObservationRegistry observationRegistry = ObservationRegistry.create();
    observationRegistry.observationConfig()
            .observationHandler(new DefaultMeterObservationHandler(meterRegistry));

    Observation.createNotStarted("test.operation", observationRegistry)
            .lowCardinalityKeyValue("method", "GET")
            .observe(() -> { /* work */ });

    Timer timer = meterRegistry.getMeters().stream()
            .filter(m -> m instanceof Timer && m.getId().getName().equals("test.operation"))
            .map(Timer.class::cast)
            .findFirst()
            .orElse(null);

    assertThat(timer).isNotNull();
    assertThat(timer.count()).isEqualTo(1);
    assertThat(timer.getId().getTags().stream()
            .filter(t -> t.getKey().equals("method"))
            .findFirst().orElseThrow().getValue())
            .isEqualTo("GET");
}
```

</details>

## 19.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/observation/DefaultMeterObservationHandlerTest.java`

Tests the handler directly by calling `onStart()`/`onStop()`/`onEvent()` with manually constructed contexts:

```java
@Test
void shouldRecordTimer_WhenObservationStartedAndStopped() {
    Observation.Context context = new Observation.Context();
    context.setName("http.requests");

    handler.onStart(context);
    handler.onStop(context);

    Timer timer = findTimer("http.requests");
    assertThat(timer).isNotNull();
    assertThat(timer.count()).isEqualTo(1);
}

@Test
void shouldExcludeHighCardinalityKeyValues_FromTimerTags() {
    Observation.Context context = new Observation.Context();
    context.setName("http.requests");
    context.addLowCardinalityKeyValue(KeyValue.of("method", "GET"));
    context.addHighCardinalityKeyValue(KeyValue.of("user.id", "12345"));

    handler.onStart(context);
    handler.onStop(context);

    Timer timer = findTimer("http.requests");
    // High-cardinality "user.id" should NOT be present as a tag
    assertThat(timer.getId().getTags().stream()
            .noneMatch(t -> t.getKey().equals("user.id")))
            .isTrue();
}

@Test
void shouldNotCreateLongTaskTimer_WhenIgnored() {
    handler = new DefaultMeterObservationHandler(meterRegistry,
            DefaultMeterObservationHandler.IgnoredMeters.LONG_TASK_TIMER);

    Observation.Context context = new Observation.Context();
    context.setName("http.requests");

    handler.onStart(context);
    handler.onStop(context);

    LongTaskTimer ltt = findLongTaskTimer("http.requests.active");
    assertThat(ltt).isNull();

    // But the Timer should still be recorded
    assertThat(findTimer("http.requests").count()).isEqualTo(1);
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/ObservationMetricsBridgeIntegrationTest.java`

Tests the full pipeline: `ObservationRegistry` → `Observation` lifecycle → `DefaultMeterObservationHandler` → `SimpleMeterRegistry`:

```java
@Test
void shouldRecordTimerAndLongTaskTimer_WhenObservationCompletes() {
    Observation observation = Observation.createNotStarted("http.requests", observationRegistry);
    observation.lowCardinalityKeyValue("method", "GET");
    observation.start();

    // While observation is active, LongTaskTimer should show 1 active task
    LongTaskTimer ltt = findLongTaskTimer("http.requests.active");
    assertThat(ltt).isNotNull();
    assertThat(ltt.activeTasks()).isEqualTo(1);

    observation.stop();

    // After stop: Timer recorded, LongTaskTimer back to 0 active
    Timer timer = findTimer("http.requests");
    assertThat(timer.count()).isEqualTo(1);
    assertThat(ltt.activeTasks()).isEqualTo(0);
    assertThat(tagValue(timer, "method")).isEqualTo("GET");
    assertThat(tagValue(timer, "error")).isEqualTo("none");
}

@Test
void shouldWorkWithObserveConvenience_ForRunnable() {
    Observation.createNotStarted("batch.job", observationRegistry)
            .lowCardinalityKeyValue("job.type", "import")
            .observe(() -> {
                LongTaskTimer ltt = findLongTaskTimer("batch.job.active");
                assertThat(ltt.activeTasks()).isEqualTo(1);
            });

    Timer timer = findTimer("batch.job");
    assertThat(timer.count()).isEqualTo(1);
    assertThat(tagValue(timer, "error")).isEqualTo("none");
    assertThat(tagValue(timer, "job.type")).isEqualTo("import");
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all 18 prior features' tests)

---

## 19.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Context as a shared data carrier eliminates coupling.** The handler stores `Timer.Sample` and `LongTaskTimer.Sample` into `Observation.Context`'s generic map using their `Class` objects as keys. This means the Observation API has zero compile-time dependency on Timer/LongTaskTimer — it just sees opaque `put(key, value)` calls. Multiple handlers (metrics, tracing, logging) can each store their own data in the same context without interfering. This is the Extension Objects pattern applied to instrumentation.
> - **Low-cardinality filtering is the critical safety valve.** Without it, a single observation with `highCardinalityKeyValue("request.uri", "/users/12345")` would create a new time series for every unique URI — millions of series in a busy system. Monitoring backends have hard limits (typically 10K-100K series per metric name). The handler's `createTags()` method only reads `getLowCardinalityKeyValues()`, making cardinality explosion structurally impossible at the bridge layer.
> - **Trade-off: the handler produces three meters per observation.** Each observed operation creates a Timer, a LongTaskTimer, and potentially Counters for events. This is intentional — Timer measures completed duration, LongTaskTimer tracks in-flight operations, and Counters track discrete events. But if your application has thousands of distinct observation names, the meter count multiplies. The `IgnoredMeters` escape hatch lets you opt out of LongTaskTimer when in-flight tracking isn't needed.
> -----------------------------------------------------------

## 19.8 What We Enhanced

| Aspect | Before (ch18) | Current (this chapter) | Real Framework |
|--------|---------------|----------------------|----------------|
| Observation → Metrics | Observation API existed but produced no metrics; handlers were abstract | `DefaultMeterObservationHandler` translates lifecycle events into Timer, LongTaskTimer, and Counter recordings | Same class `DefaultMeterObservationHandler.java:42` — also implements `MeterObservationHandler` marker interface for type-safe composite handler selection |
| Error tracking | Errors recorded on context but not propagated to metrics | Error tag (`error=RuntimeException` or `error=none`) automatically added to Timer tags at stop time | Same pattern — real version also adds error tag only to Timer, not to LongTaskTimer |
| Event counting | Events signaled to handlers but had no metric effect | Each event produces a Counter named `<observation>.<event>` | Same approach — `DefaultMeterObservationHandler.java:94` |
| Opt-out | N/A | `IgnoredMeters.LONG_TASK_TIMER` disables LongTaskTimer creation | `IgnoredMeters` enum at `DefaultMeterObservationHandler.java:147` — identical design |

## 19.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `DefaultMeterObservationHandler` | `DefaultMeterObservationHandler` | `DefaultMeterObservationHandler.java:42` | Real version implements `MeterObservationHandler<Observation.Context>` (a marker interface) instead of bare `ObservationHandler`. This enables `FirstMatchingCompositeObservationHandler` to select between multiple metric handlers. |
| `createTags()` using loop | `createTags()` using loop | `DefaultMeterObservationHandler.java:131` | Identical approach — iterates `getLowCardinalityKeyValues()` and converts to Tags |
| `getErrorValue()` | `getErrorValue()` | `DefaultMeterObservationHandler.java:122` | Identical — uses `error.getClass().getSimpleName()` or `KeyValue.NONE_VALUE` |
| `LongTaskTimer.builder(...).register(registry).start()` | `meterRegistry.more().longTaskTimer(name, tags).start()` | `DefaultMeterObservationHandler.java:73` | Real version uses a convenience method on `MeterRegistry.More`; we use the builder pattern (same registration pipeline) |
| `IgnoredMeters` enum | `IgnoredMeters` enum | `DefaultMeterObservationHandler.java:147` | Identical design — enum with single `LONG_TASK_TIMER` constant |

## 19.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/observation/DefaultMeterObservationHandler.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Timer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * Bridges the {@link Observation} API to Micrometer metrics — translates
 * observation lifecycle events into {@link Timer}, {@link Counter}, and
 * {@link LongTaskTimer} recordings.
 * <p>
 * This is the "instrument once, export anywhere" glue: frameworks instrument
 * code via the Observation API, and this handler automatically produces
 * the corresponding metrics.
 * <p>
 * <b>Lifecycle mapping:</b>
 * <ul>
 *   <li>{@code onStart()} → starts a {@link Timer.Sample} and a {@link LongTaskTimer}
 *       (for tracking in-flight observations)</li>
 *   <li>{@code onStop()} → stops both the Timer.Sample and the LongTaskTimer.Sample,
 *       recording to a Timer named after the observation with an {@code error} tag</li>
 *   <li>{@code onEvent()} → increments a Counter named
 *       {@code <observation-name>.<event-name>}</li>
 * </ul>
 * <p>
 * <b>Tag cardinality:</b> Only {@linkplain Observation.Context#getLowCardinalityKeyValues()
 * low-cardinality key values} are translated to metric tags. High-cardinality key values
 * are intentionally excluded — they would cause unbounded time series (cardinality
 * explosion) in the monitoring backend.
 * <p>
 * <b>WARNING:</b> The {@link LongTaskTimer} is created in {@code onStart()}, so it can
 * only contain tags available at that time. Low-cardinality key values added after
 * {@code start()} will NOT appear on the LongTaskTimer (but will appear on the Timer,
 * which is resolved at stop time).
 */
public class DefaultMeterObservationHandler implements ObservationHandler<Observation.Context> {

    private final MeterRegistry meterRegistry;

    private final boolean shouldCreateLongTaskTimer;

    /**
     * Creates a handler that produces Timer, LongTaskTimer, and Counter metrics.
     *
     * @param meterRegistry The registry to create meters in.
     */
    public DefaultMeterObservationHandler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.shouldCreateLongTaskTimer = true;
    }

    /**
     * Creates a handler with the option to disable specific meter types.
     *
     * @param meterRegistry  The registry to create meters in.
     * @param metersToIgnore Meter types to suppress (e.g., {@link IgnoredMeters#LONG_TASK_TIMER}).
     */
    public DefaultMeterObservationHandler(MeterRegistry meterRegistry,
            IgnoredMeters... metersToIgnore) {
        this.meterRegistry = meterRegistry;
        this.shouldCreateLongTaskTimer = Arrays.stream(metersToIgnore)
                .noneMatch(ignored -> ignored == IgnoredMeters.LONG_TASK_TIMER);
    }

    @Override
    public void onStart(Observation.Context context) {
        // Start the LongTaskTimer to track in-flight observations.
        // Created at start time — can only use tags available NOW.
        if (shouldCreateLongTaskTimer) {
            LongTaskTimer.Sample longTaskSample = LongTaskTimer
                    .builder(context.getName() + ".active")
                    .tags(createTags(context))
                    .register(meterRegistry)
                    .start();
            context.put(LongTaskTimer.Sample.class, longTaskSample);
        }

        // Start a Timer.Sample to measure total observation duration.
        // The sample captures the current monotonic time.
        Timer.Sample sample = Timer.start(meterRegistry);
        context.put(Timer.Sample.class, sample);
    }

    @Override
    public void onStop(Observation.Context context) {
        // Build tags from low-cardinality key values (available at stop time,
        // so we get all key values including those added after start).
        List<Tag> tags = createTags(context);

        // Add the error tag — either the exception's simple class name or "none".
        tags.add(Tag.of("error", getErrorValue(context)));

        // Stop the Timer.Sample against a Timer named after the observation.
        Timer.Sample sample = context.getRequired(Timer.Sample.class);
        sample.stop(meterRegistry.timer(context.getName(), tags));

        // Stop the LongTaskTimer to remove this observation from the active set.
        if (shouldCreateLongTaskTimer) {
            LongTaskTimer.Sample longTaskSample = context.getRequired(LongTaskTimer.Sample.class);
            longTaskSample.stop();
        }
    }

    @Override
    public void onEvent(Observation.Event event, Observation.Context context) {
        // Each event increments a counter named <observation>.<event>.
        Counter.builder(context.getName() + "." + event.getName())
                .tags(createTags(context))
                .register(meterRegistry)
                .increment();
    }

    @Override
    public boolean supportsContext(Observation.Context context) {
        return true;
    }

    /**
     * Returns the error tag value: the exception's simple class name, or
     * {@value KeyValue#NONE_VALUE} if no error occurred.
     */
    private String getErrorValue(Observation.Context context) {
        Throwable error = context.getError();
        return error != null ? error.getClass().getSimpleName() : KeyValue.NONE_VALUE;
    }

    /**
     * Converts low-cardinality key values from the observation context into
     * Micrometer {@link Tag}s. Returns a <b>mutable</b> list so that
     * {@code onStop()} can append the error tag.
     * <p>
     * Only low-cardinality key values are used — high-cardinality values would
     * cause cardinality explosion in monitoring backends.
     */
    private List<Tag> createTags(Observation.Context context) {
        List<Tag> tags = new ArrayList<>();
        for (KeyValue keyValue : context.getLowCardinalityKeyValues()) {
            tags.add(Tag.of(keyValue.getKey(), keyValue.getValue()));
        }
        return tags;
    }

    /**
     * Meter types that can be selectively disabled.
     * <p>
     * Designed as an enum (not a boolean) so it can be extended in the future
     * to support disabling additional meter types.
     */
    public enum IgnoredMeters {

        /**
         * Suppress the {@link LongTaskTimer} that tracks in-flight observations.
         * Useful when the overhead of maintaining active task tracking is not needed.
         */
        LONG_TASK_TIMER

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/observation/DefaultMeterObservationHandlerTest.java` [NEW]

```java
package dev.linhvu.micrometer.observation;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link DefaultMeterObservationHandler} — the bridge that translates
 * Observation lifecycle events into Timer, LongTaskTimer, and Counter recordings.
 */
class DefaultMeterObservationHandlerTest {

    private SimpleMeterRegistry meterRegistry;
    private DefaultMeterObservationHandler handler;

    @BeforeEach
    void setUp() {
        meterRegistry = new SimpleMeterRegistry();
        handler = new DefaultMeterObservationHandler(meterRegistry);
    }

    // -----------------------------------------------------------------------
    // Timer (onStart → onStop)
    // -----------------------------------------------------------------------

    @Test
    void shouldRecordTimer_WhenObservationStartedAndStopped() {
        Observation.Context context = new Observation.Context();
        context.setName("http.requests");

        handler.onStart(context);
        handler.onStop(context);

        // A timer named "http.requests" should be registered
        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldAddErrorTagNone_WhenNoErrorOccurred() {
        Observation.Context context = new Observation.Context();
        context.setName("http.requests");

        handler.onStart(context);
        handler.onStop(context);

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        // The error tag should be "none" when no error occurred
        assertThat(timer.getId().getTags().stream()
                .filter(t -> t.getKey().equals("error"))
                .findFirst()
                .orElseThrow()
                .getValue())
                .isEqualTo("none");
    }

    @Test
    void shouldAddErrorTag_WhenErrorOccurred() {
        Observation.Context context = new Observation.Context();
        context.setName("http.requests");
        context.setError(new IllegalArgumentException("bad input"));

        handler.onStart(context);
        handler.onStop(context);

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(timer.getId().getTags().stream()
                .filter(t -> t.getKey().equals("error"))
                .findFirst()
                .orElseThrow()
                .getValue())
                .isEqualTo("IllegalArgumentException");
    }

    @Test
    void shouldUseLowCardinalityKeyValuesAsTags() {
        Observation.Context context = new Observation.Context();
        context.setName("http.requests");
        context.addLowCardinalityKeyValue(KeyValue.of("method", "GET"));
        context.addLowCardinalityKeyValue(KeyValue.of("status", "200"));

        handler.onStart(context);
        handler.onStop(context);

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(timer.getId().getTags().stream()
                .filter(t -> t.getKey().equals("method"))
                .findFirst()
                .orElseThrow()
                .getValue())
                .isEqualTo("GET");
        assertThat(timer.getId().getTags().stream()
                .filter(t -> t.getKey().equals("status"))
                .findFirst()
                .orElseThrow()
                .getValue())
                .isEqualTo("200");
    }

    @Test
    void shouldExcludeHighCardinalityKeyValues_FromTimerTags() {
        Observation.Context context = new Observation.Context();
        context.setName("http.requests");
        context.addLowCardinalityKeyValue(KeyValue.of("method", "GET"));
        context.addHighCardinalityKeyValue(KeyValue.of("user.id", "12345"));

        handler.onStart(context);
        handler.onStop(context);

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        // High-cardinality "user.id" should NOT be present as a tag
        assertThat(timer.getId().getTags().stream()
                .noneMatch(t -> t.getKey().equals("user.id")))
                .isTrue();
    }

    // -----------------------------------------------------------------------
    // LongTaskTimer (onStart → onStop)
    // -----------------------------------------------------------------------

    @Test
    void shouldCreateLongTaskTimer_WhenObservationStarted() {
        Observation.Context context = new Observation.Context();
        context.setName("batch.import");

        handler.onStart(context);

        // A long task timer named "batch.import.active" should exist
        LongTaskTimer ltt = findLongTaskTimer("batch.import.active");
        assertThat(ltt).isNotNull();
        assertThat(ltt.activeTasks()).isEqualTo(1);
    }

    @Test
    void shouldStopLongTaskTimer_WhenObservationStopped() {
        Observation.Context context = new Observation.Context();
        context.setName("batch.import");

        handler.onStart(context);
        assertThat(findLongTaskTimer("batch.import.active").activeTasks()).isEqualTo(1);

        handler.onStop(context);
        assertThat(findLongTaskTimer("batch.import.active").activeTasks()).isEqualTo(0);
    }

    @Test
    void shouldNotCreateLongTaskTimer_WhenIgnored() {
        handler = new DefaultMeterObservationHandler(meterRegistry,
                DefaultMeterObservationHandler.IgnoredMeters.LONG_TASK_TIMER);

        Observation.Context context = new Observation.Context();
        context.setName("http.requests");

        handler.onStart(context);
        handler.onStop(context);

        // No LongTaskTimer should exist
        LongTaskTimer ltt = findLongTaskTimer("http.requests.active");
        assertThat(ltt).isNull();

        // But the Timer should still be recorded
        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(1);
    }

    // -----------------------------------------------------------------------
    // Counter (onEvent)
    // -----------------------------------------------------------------------

    @Test
    void shouldIncrementCounter_WhenEventSignaled() {
        Observation.Context context = new Observation.Context();
        context.setName("message.processing");

        handler.onStart(context);

        Observation.Event event = Observation.Event.of("message.received");
        handler.onEvent(event, context);

        // Counter named "message.processing.message.received" should exist
        Counter counter = findCounter("message.processing.message.received");
        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(1.0);
    }

    @Test
    void shouldIncrementCounter_WhenMultipleEventsFired() {
        Observation.Context context = new Observation.Context();
        context.setName("queue");

        handler.onStart(context);

        Observation.Event event = Observation.Event.of("retry");
        handler.onEvent(event, context);
        handler.onEvent(event, context);
        handler.onEvent(event, context);

        Counter counter = findCounter("queue.retry");
        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(3.0);
    }

    @Test
    void shouldUseContextTags_ForEventCounter() {
        Observation.Context context = new Observation.Context();
        context.setName("queue");
        context.addLowCardinalityKeyValue(KeyValue.of("queue.name", "orders"));

        handler.onStart(context);

        handler.onEvent(Observation.Event.of("retry"), context);

        Counter counter = findCounter("queue.retry");
        assertThat(counter).isNotNull();
        assertThat(counter.getId().getTags().stream()
                .filter(t -> t.getKey().equals("queue.name"))
                .findFirst()
                .orElseThrow()
                .getValue())
                .isEqualTo("orders");
    }

    // -----------------------------------------------------------------------
    // supportsContext
    // -----------------------------------------------------------------------

    @Test
    void shouldSupportAllContextTypes() {
        assertThat(handler.supportsContext(new Observation.Context())).isTrue();
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    private Timer findTimer(String name) {
        return meterRegistry.getMeters().stream()
                .filter(m -> m instanceof Timer && m.getId().getName().equals(name))
                .map(Timer.class::cast)
                .findFirst()
                .orElse(null);
    }

    private LongTaskTimer findLongTaskTimer(String name) {
        return meterRegistry.getMeters().stream()
                .filter(m -> m instanceof LongTaskTimer && m.getId().getName().equals(name))
                .map(LongTaskTimer.class::cast)
                .findFirst()
                .orElse(null);
    }

    private Counter findCounter(String name) {
        return meterRegistry.getMeters().stream()
                .filter(m -> m instanceof Counter && m.getId().getName().equals(name))
                .map(Counter.class::cast)
                .findFirst()
                .orElse(null);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/ObservationMetricsBridgeIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.observation.DefaultMeterObservationHandler;
import dev.linhvu.micrometer.observation.Observation;
import dev.linhvu.micrometer.observation.ObservationRegistry;
import dev.linhvu.micrometer.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Integration tests for the Observation-Metrics Bridge. Wires a
 * {@link DefaultMeterObservationHandler} into an {@link ObservationRegistry}
 * and verifies that the full observation lifecycle produces the expected
 * Timer, LongTaskTimer, and Counter meters in a {@link SimpleMeterRegistry}.
 */
class ObservationMetricsBridgeIntegrationTest {

    private SimpleMeterRegistry meterRegistry;
    private ObservationRegistry observationRegistry;

    @BeforeEach
    void setUp() {
        meterRegistry = new SimpleMeterRegistry();
        observationRegistry = ObservationRegistry.create();
        observationRegistry.observationConfig()
                .observationHandler(new DefaultMeterObservationHandler(meterRegistry));
    }

    @Test
    void shouldRecordTimerAndLongTaskTimer_WhenObservationCompletes() {
        Observation observation = Observation.createNotStarted("http.requests", observationRegistry);
        observation.lowCardinalityKeyValue("method", "GET");
        observation.start();

        // While observation is active, LongTaskTimer should show 1 active task
        LongTaskTimer ltt = findLongTaskTimer("http.requests.active");
        assertThat(ltt).isNotNull();
        assertThat(ltt.activeTasks()).isEqualTo(1);

        observation.stop();

        // After stop: Timer recorded, LongTaskTimer back to 0 active
        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(ltt.activeTasks()).isEqualTo(0);

        // Timer should have method=GET and error=none tags
        assertThat(tagValue(timer, "method")).isEqualTo("GET");
        assertThat(tagValue(timer, "error")).isEqualTo("none");
    }

    @Test
    void shouldRecordErrorTag_WhenObservationHasError() {
        Observation observation = Observation.createNotStarted("http.requests", observationRegistry);
        observation.lowCardinalityKeyValue("method", "POST");
        observation.start();
        observation.error(new RuntimeException("server error"));
        observation.stop();

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(tagValue(timer, "error")).isEqualTo("RuntimeException");
    }

    @Test
    void shouldIncrementCounter_WhenEventSignaledDuringObservation() {
        Observation observation = Observation.createNotStarted("message.processing", observationRegistry);
        observation.start();
        observation.event(Observation.Event.of("message.received"));
        observation.event(Observation.Event.of("message.received"));
        observation.stop();

        Counter counter = findCounter("message.processing.message.received");
        assertThat(counter).isNotNull();
        assertThat(counter.count()).isEqualTo(2.0);
    }

    @Test
    void shouldWorkWithObserveConvenience_ForRunnable() {
        Observation.createNotStarted("batch.job", observationRegistry)
                .lowCardinalityKeyValue("job.type", "import")
                .observe(() -> {
                    // Simulate work — the observation is active here
                    LongTaskTimer ltt = findLongTaskTimer("batch.job.active");
                    assertThat(ltt).isNotNull();
                    assertThat(ltt.activeTasks()).isEqualTo(1);
                });

        Timer timer = findTimer("batch.job");
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(1);
        assertThat(tagValue(timer, "error")).isEqualTo("none");
        assertThat(tagValue(timer, "job.type")).isEqualTo("import");
    }

    @Test
    void shouldWorkWithObserveConvenience_ForSupplier() {
        String result = Observation.createNotStarted("compute", observationRegistry)
                .observe(() -> "hello world");

        assertThat(result).isEqualTo("hello world");
        Timer timer = findTimer("compute");
        assertThat(timer).isNotNull();
        assertThat(timer.count()).isEqualTo(1);
    }

    @Test
    void shouldRecordErrorTag_WhenObserveThrows() {
        RuntimeException error = new RuntimeException("boom");

        assertThatThrownBy(() ->
                Observation.createNotStarted("http.requests", observationRegistry)
                        .observe(() -> { throw error; })
        ).isSameAs(error);

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(tagValue(timer, "error")).isEqualTo("RuntimeException");
    }

    @Test
    void shouldNotProduceMetrics_WhenObservationIsNoop() {
        // An empty registry (no handlers) returns noop observations
        ObservationRegistry emptyRegistry = ObservationRegistry.create();

        Observation.createNotStarted("noop.operation", emptyRegistry)
                .observe(() -> {
                    // no-op, nothing should be recorded
                });

        // No meters should exist
        assertThat(meterRegistry.getMeters()).isEmpty();
    }

    @Test
    void shouldIncludeKeyValuesAddedAfterStart_InTimerTags() {
        Observation observation = Observation.createNotStarted("http.requests", observationRegistry);
        observation.lowCardinalityKeyValue("method", "GET");
        observation.start();

        // Add a key value AFTER start — should appear on Timer but NOT on LongTaskTimer
        observation.lowCardinalityKeyValue("status", "200");
        observation.stop();

        Timer timer = findTimer("http.requests");
        assertThat(timer).isNotNull();
        assertThat(tagValue(timer, "status")).isEqualTo("200");
    }

    @Test
    void shouldHandleMultipleObservationsIndependently() {
        Observation obs1 = Observation.createNotStarted("op.a", observationRegistry);
        Observation obs2 = Observation.createNotStarted("op.b", observationRegistry);

        obs1.start();
        obs2.start();
        obs1.stop();
        obs2.stop();

        assertThat(findTimer("op.a")).isNotNull();
        assertThat(findTimer("op.a").count()).isEqualTo(1);
        assertThat(findTimer("op.b")).isNotNull();
        assertThat(findTimer("op.b").count()).isEqualTo(1);
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    private Timer findTimer(String name) {
        return meterRegistry.getMeters().stream()
                .filter(m -> m instanceof Timer && m.getId().getName().equals(name))
                .map(Timer.class::cast)
                .findFirst()
                .orElse(null);
    }

    private LongTaskTimer findLongTaskTimer(String name) {
        return meterRegistry.getMeters().stream()
                .filter(m -> m instanceof LongTaskTimer && m.getId().getName().equals(name))
                .map(LongTaskTimer.class::cast)
                .findFirst()
                .orElse(null);
    }

    private Counter findCounter(String name) {
        return meterRegistry.getMeters().stream()
                .filter(m -> m instanceof Counter && m.getId().getName().equals(name))
                .map(Counter.class::cast)
                .findFirst()
                .orElse(null);
    }

    private String tagValue(Meter meter, String tagKey) {
        return meter.getId().getTags().stream()
                .filter(t -> t.getKey().equals(tagKey))
                .findFirst()
                .map(t -> t.getValue())
                .orElse(null);
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Observation-Metrics Bridge** | An `ObservationHandler` that translates observation lifecycle events (start/stop/event) into Micrometer meter recordings (Timer, LongTaskTimer, Counter) |
| **Low-cardinality filtering** | Only key values with bounded value sets become metric tags — high-cardinality values are excluded to prevent unbounded time series |
| **Context as data carrier** | The `Observation.Context`'s generic `put()`/`get()` map stores handler-specific state (Timer.Sample, LongTaskTimer.Sample) without coupling the observation API to any specific meter type |
| **Error tag** | A tag added at stop time with the exception's simple class name (or `"none"`), enabling error-rate breakdowns in dashboards |
| **IgnoredMeters** | An extensible enum for selectively disabling meter types (currently only LongTaskTimer) when their overhead isn't needed |

This is the **final chapter** — all 19 features of Simple Micrometer are now complete. You've built the full pipeline from low-level identity and tags (ch1) through meter types and registries (ch2-17) to the high-level Observation API (ch18) and its bridge to metrics (ch19). The "instrument once, export anywhere" promise is now fully realized.
