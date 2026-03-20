# Chapter 17: Logging Registry

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| `StepMeterRegistry` provides a complete step-based infrastructure with step rollover, meter polling, and scheduled publishing — but `publish()` is abstract. There's no concrete registry that actually *does something* with the metrics. | You can record metrics into step meters all day, but there's no way to see the output. The push/step infrastructure is tested only with `TestStepMeterRegistry` stubs that count publish calls. | Build a `LoggingMeterRegistry` — the simplest concrete registry — that implements `publish()` by formatting every meter into a human-readable string and sending it to a configurable logging sink |

---

## 17.1 The Integration Point: Extending StepMeterRegistry with publish() and getBaseTimeUnit()

The logging registry connects to the existing system by **extending `StepMeterRegistry`** and implementing its two abstract requirements: `publish()` and `getBaseTimeUnit()`. This is the exact same extension pattern that any real backend registry (Datadog, InfluxDB, OTLP) would follow.

**New file:** `src/main/java/dev/linhvu/micrometer/logging/LoggingMeterRegistry.java`

The core integration is the `publish()` method:

```java
public class LoggingMeterRegistry extends StepMeterRegistry {

    @Override
    protected void publish() {
        if (!config.enabled()) { return; }

        List<Meter> meters = getMeters().stream()
                .sorted(Comparator
                        .comparing((Meter m) -> m.getId().getType().name())
                        .thenComparing(m -> m.getId().getName()))
                .toList();

        for (Meter meter : meters) {
            Printer print = new Printer(meter);
            meter.use(
                    counter   -> { /* format counter */ },
                    gauge     -> { /* format gauge or TimeGauge */ },
                    timer     -> { /* format timer */ },
                    summary   -> { /* format distribution summary */ },
                    ltt       -> { /* format long task timer */ },
                    funcCount -> { /* format function counter */ },
                    funcTimer -> { /* format function timer */ },
                    other     -> { /* fallback for custom meters */ }
            );
        }
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.MILLISECONDS;
    }
}
```

Two key decisions:

1. **Why use `Meter.use()` (the visitor pattern) instead of `instanceof` chains?** The visitor dispatches through the same 8-type interface that every meter supports. If a new meter type is added, `use()` will force a compile error at every call site, ensuring nothing is silently missed. It also keeps the formatting logic for each type clean and isolated.

2. **Why sort meters before logging?** Without sorting, the output order would depend on the internal `ConcurrentHashMap` ordering — different every JVM run. Sorting by type then name produces stable, predictable output that's easy to scan in logs and diff across runs.

This connects **the format-and-log output** to the **step-based push infrastructure**. To make it work, we also need:
- `LoggingRegistryConfig` — configuration with a `logInactive()` toggle
- `DoubleFormat` — utility for formatting doubles without trailing zeros
- `TimeUtils.format(Duration)` — human-readable duration formatting
- A `Printer` inner class that formats values relative to each meter's context

## 17.2 LoggingRegistryConfig — Configuration

**New file:** `src/main/java/dev/linhvu/micrometer/logging/LoggingRegistryConfig.java`

```java
public interface LoggingRegistryConfig extends StepRegistryConfig {

    LoggingRegistryConfig DEFAULT = new LoggingRegistryConfig() { };

    default boolean logInactive() {
        return false;
    }
}
```

The only property unique to the logging registry: whether to include meters with zero activity. All other config (step interval, enabled, batch size) is inherited from `StepRegistryConfig` → `PushRegistryConfig`.

## 17.3 Supporting Utilities

### DoubleFormat — Human-Readable Number Formatting

**New file:** `src/main/java/dev/linhvu/micrometer/DoubleFormat.java`

```java
public final class DoubleFormat {

    public static String decimalOrNan(double value) {
        if (Double.isNaN(value)) return "NaN";
        if (Double.isInfinite(value)) return Double.toString(value);
        String formatted = String.format("%.6f", value);
        if (formatted.contains(".")) {
            formatted = formatted.replaceAll("0+$", "");
            formatted = formatted.replaceAll("\\.$", "");
        }
        return formatted;
    }

    public static String wholeOrDecimal(double value) {
        if (Double.isNaN(value) || Double.isInfinite(value)) return Double.toString(value);
        if (value == Math.floor(value) && !Double.isInfinite(value)) return Long.toString((long) value);
        return decimalOrNan(value);
    }
}
```

### TimeUtils.format(Duration) — Human-Readable Duration Formatting

**Modifying:** `src/main/java/dev/linhvu/micrometer/TimeUtils.java`
**Change:** Add the `format(Duration)` method after the existing `convert()` method.

```java
public static String format(Duration duration) {
    long nanos = duration.toNanos();
    if (nanos == 0) return "0s";

    long hours = duration.toHours();
    long minutes = duration.toMinutesPart();
    long seconds = duration.toSecondsPart();
    long millis = duration.toMillisPart();
    // ... format as "1h 23m 45.678s", "150ms", "42.5s", etc.
}
```

## 17.4 LoggingMeterRegistry — The Full Implementation

**New file:** `src/main/java/dev/linhvu/micrometer/logging/LoggingMeterRegistry.java`

```java
public class LoggingMeterRegistry extends StepMeterRegistry {

    private final LoggingRegistryConfig config;
    private final Consumer<String> loggingSink;

    public LoggingMeterRegistry(LoggingRegistryConfig config, Clock clock,
            Consumer<String> loggingSink) {
        super(config, clock);
        this.config = config;
        this.loggingSink = loggingSink;
        namingConvention(NamingConvention.dot);
    }

    @Override
    protected void publish() {
        if (!config.enabled()) { return; }

        List<Meter> meters = getMeters().stream()
                .sorted(Comparator
                        .comparing((Meter m) -> m.getId().getType().name())
                        .thenComparing(m -> m.getId().getName()))
                .toList();

        for (Meter meter : meters) {
            Printer print = new Printer(meter);
            meter.use(
                    counter -> {
                        double count = counter.count();
                        if (count == 0 && !config.logInactive()) return;
                        loggingSink.accept(print.id()
                                + " delta_count=" + print.humanReadableCount(count)
                                + " throughput=" + print.rate(count));
                    },
                    gauge -> {
                        if (gauge instanceof TimeGauge timeGauge) {
                            double value = timeGauge.value(getBaseTimeUnit());
                            if (value == 0 && !config.logInactive()) return;
                            loggingSink.accept(print.id() + " value=" + print.time(value));
                        } else {
                            loggingSink.accept(print.id() + " value=" + print.value(gauge.value()));
                        }
                    },
                    timer -> {
                        HistogramSnapshot snapshot = timer.takeSnapshot();
                        long count = snapshot.count();
                        if (count == 0 && !config.logInactive()) return;
                        loggingSink.accept(print.id()
                                + " delta_count=" + count
                                + " throughput=" + print.unitlessRate(count)
                                + " mean=" + print.time(snapshot.mean(getBaseTimeUnit()))
                                + " max=" + print.time(snapshot.max(getBaseTimeUnit())));
                    },
                    // ... DistributionSummary, LongTaskTimer, FunctionCounter, FunctionTimer, fallback
            );
        }
    }

    @Override
    protected TimeUnit getBaseTimeUnit() { return TimeUnit.MILLISECONDS; }
}
```

### The Printer Inner Class

The `Printer` is created per-meter and formats values in that meter's context:

```java
class Printer {
    private final Meter meter;

    String id() {
        String conventionName = meter.getId().getConventionName(getNamingConvention());
        List<Tag> conventionTags = meter.getId().getConventionTags(getNamingConvention());
        String tagString = conventionTags.stream()
                .map(t -> t.getKey() + "=" + t.getValue())
                .collect(joining(",", "{", "}"));
        return conventionName + tagString;
    }

    String time(double timeInBaseUnit) {
        double nanos = TimeUtils.convert(timeInBaseUnit, getBaseTimeUnit(), TimeUnit.NANOSECONDS);
        return TimeUtils.format(Duration.ofNanos((long) nanos));
    }

    String rate(double count) {
        double perSecond = count / (config.step().toMillis() / 1000.0);
        return humanReadableBaseUnit(perSecond) + "/s";
    }

    private String humanReadableBaseUnit(double value) {
        String baseUnit = meter.getId().getBaseUnit();
        if ("bytes".equals(baseUnit)) return humanReadableByteCount(value);
        String formatted = DoubleFormat.wholeOrDecimal(value);
        return baseUnit != null ? formatted + " " + baseUnit : formatted;
    }
}
```

### Output Examples

| Meter Type | Example Output |
|---|---|
| **Counter** | `http.requests{method=GET} delta_count=150 throughput=2.5/s` |
| **Gauge** | `pool.size{} value=42` |
| **Timer** | `http.latency{} delta_count=10 throughput=0.166667/s mean=150ms max=500ms` |
| **DistributionSummary** | `payload.size{} delta_count=3 throughput=0.05/s mean=200 max=300` |
| **LongTaskTimer** | `batch.import{} active=1 duration=5s mean=5s max=5s` |
| **TimeGauge** | `jvm.uptime{} value=1m 5s` |

---

## 17.5 Try It Yourself

<details><summary>Challenge 1: Implement the counter formatting in publish()</summary>

Given a `Counter`, format it as: `{id} delta_count={count} throughput={count/step}/s`

```java
counter -> {
    double count = counter.count();
    if (count == 0 && !config.logInactive()) return;
    loggingSink.accept(print.id()
            + " delta_count=" + print.humanReadableCount(count)
            + " throughput=" + print.rate(count));
},
```

</details>

<details><summary>Challenge 2: Handle TimeGauge inside the Gauge visitor</summary>

The `Meter.use()` method routes `TimeGauge` to the Gauge consumer (since `TimeGauge extends Gauge`). Check `instanceof` to handle it separately:

```java
gauge -> {
    if (gauge instanceof TimeGauge timeGauge) {
        double value = timeGauge.value(getBaseTimeUnit());
        if (value == 0 && !config.logInactive()) return;
        loggingSink.accept(print.id() + " value=" + print.time(value));
    } else {
        loggingSink.accept(print.id() + " value=" + print.value(gauge.value()));
    }
},
```

</details>

<details><summary>Challenge 3: Implement human-readable byte formatting</summary>

```java
private String humanReadableByteCount(double bytes) {
    if (bytes < 1024) return DoubleFormat.wholeOrDecimal(bytes) + " B";
    int exp = (int) (Math.log(bytes) / Math.log(1024));
    String unit = "KMGTPE".charAt(exp - 1) + "iB";
    return String.format("%.1f %s", bytes / Math.pow(1024, exp), unit);
}
```

</details>

---

## 17.6 Tests

### Unit Tests

Test methods verify individual formatting and configuration behaviors:

| Test | Verifies |
|------|----------|
| `shouldFormatCounter_WithDeltaCountAndThroughput` | Counter output contains `delta_count=` and `throughput=` |
| `shouldSkipInactiveCounter_WhenLogInactiveIsFalse` | Zero-count counter is suppressed |
| `shouldLogInactiveCounter_WhenLogInactiveIsTrue` | Zero-count counter appears when config says so |
| `shouldFormatGauge_WithCurrentValue` | Gauge shows `value=42` |
| `shouldAlwaysLogGauge_EvenWhenValueIsZero` | Gauges are never suppressed |
| `shouldFormatTimer_WithCountMeanAndMax` | Timer includes count, mean, and max |
| `shouldFormatLongTaskTimer_WithActiveTasksAndDuration` | LTT shows `active=1` and `duration=` |
| `shouldIncludeTags_InOutputFormat` | Tags appear as `method=GET,status=200` |
| `shouldFormatBytesUnit_AsHumanReadable` | 1 MiB shows `MiB` in output |
| `shouldUseMillisecondsAsBaseTimeUnit` | `getBaseTimeUnit()` returns `MILLISECONDS` |

### Integration Tests

| Test | Verifies |
|------|----------|
| `shouldPublishStepValues_WhenFullLifecycleCompletes` | Full pipeline: register → record → step → publish |
| `shouldApplyCommonTags_WhenFilterConfigured` | MeterFilter integration |
| `shouldNotPublishDeniedMeter_WhenFilterDenies` | Filter deny works with logging registry |
| `shouldApplyNamingConvention_WhenCustomConventionSet` | snakeCase transforms names and tag keys |
| `shouldPublishDifferentValues_AcrossSteps` | Step rollover produces different deltas |
| `shouldPublishOnClose_IncludingPartialStepData` | Graceful shutdown publishes partial data |
| `shouldPublishFunctionCounterDelta_AcrossSteps` | FunctionCounter deltas published correctly |

---

## 17.7 Why This Works

> `★ Insight ─────────────────────────────────────`
> **The Two Methods That Make a Registry:**
> Every concrete Micrometer registry — whether it's Prometheus, Datadog, or our Logging registry — needs to implement exactly **two things**: `publish()` (how to format and send metrics) and `getBaseTimeUnit()` (what unit the backend expects). Everything else — meter creation, deduplication, step rollover, scheduled publishing, graceful shutdown — is inherited from the `StepMeterRegistry` → `PushMeterRegistry` → `MeterRegistry` chain. This is the Template Method pattern paying off: 16 chapters of infrastructure, and the concrete registry is just ~150 lines.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **Why `Consumer<String>` Instead of a Logger:**
> The real Micrometer uses its own `InternalLogger` abstraction. We use `Consumer<String>` — which is more flexible for both production use (`logger::info`) and testing (`logLines::add`). In tests, we inject a list collector and assert on the captured lines. This makes the tests deterministic and fast — no need to configure logging frameworks or parse log output.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **TimeGauge Dispatch — A Limitation of the Visitor:**
> `TimeGauge extends Gauge`, so `Meter.use()` routes it to the Gauge consumer. The real Micrometer has a 9-parameter overload of `use()` that separates TimeGauge from Gauge. Our simplified version uses `instanceof` inside the Gauge consumer — a pragmatic workaround that avoids changing the visitor interface for one consumer.
> `─────────────────────────────────────────────────`

---

## 17.8 What We Enhanced

| Component | Enhancement | Why |
|-----------|-------------|-----|
| `TimeUtils` | Added `format(Duration)` method | Human-readable duration output (e.g., "1m 5s") needed by the logging registry's `Printer.time()` |
| `DoubleFormat` (NEW) | Utility for formatting doubles | Consistent number formatting across all meter types — strips trailing zeros, handles NaN/Infinity |

---

## 17.9 Connection to Real Framework

| Simplified | Real Micrometer | Location |
|-----------|-----------------|----------|
| `LoggingMeterRegistry` | `LoggingMeterRegistry` | `micrometer-core/.../logging/LoggingMeterRegistry.java:1` |
| `LoggingRegistryConfig` | `LoggingRegistryConfig` | `micrometer-core/.../logging/LoggingRegistryConfig.java:1` |
| `Consumer<String> loggingSink` | `Consumer<String> loggingSink` + `InternalLogger` | Real version defaults to `InternalLogger.info()` |
| `Printer` inner class | `Printer` inner class | `LoggingMeterRegistry.java:280` — same per-meter helper pattern |
| `DoubleFormat` | `DoubleFormat` | `micrometer-commons/.../internal/DoubleFormat.java:1` |
| `TimeUtils.format(Duration)` | `TimeUtils.format(Duration)` | `micrometer-commons/.../internal/TimeUtils.java:40` |
| Direct `instanceof TimeGauge` check | Separate `TimeGauge` visitor slot | Real `Meter.use()` has 9 parameters separating TimeGauge from Gauge |
| No Builder pattern | `LoggingMeterRegistry.Builder` | Real version uses a builder for customizing `Clock`, `ThreadFactory`, `loggingSink`, and `meterIdPrinter` |

> Commit: `2c8a4606c`

---

## 17.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/DoubleFormat.java` [NEW]

```java
package dev.linhvu.micrometer;

/**
 * Utility for formatting double values in human-readable form for logging output.
 * <p>
 * Two styles:
 * <ul>
 *   <li>{@link #decimalOrNan(double)} — always shows up to 6 decimal places, stripping
 *       trailing zeros. {@code NaN} and {@code Infinity} pass through as strings.</li>
 *   <li>{@link #wholeOrDecimal(double)} — shows as an integer if the value has no
 *       fractional part; otherwise shows up to 6 decimal places.</li>
 * </ul>
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code DoubleFormat} lives in an internal
 * util package and handles additional edge cases. We keep only what
 * {@code LoggingMeterRegistry} needs.
 */
public final class DoubleFormat {

    private DoubleFormat() {
    }

    /**
     * Formats a double value, stripping unnecessary trailing zeros.
     * NaN and infinite values are returned as their standard string representations.
     *
     * @param value The value to format.
     * @return A human-readable string.
     */
    public static String decimalOrNan(double value) {
        if (Double.isNaN(value)) {
            return "NaN";
        }
        if (Double.isInfinite(value)) {
            return Double.toString(value);
        }
        // Format with up to 6 decimal places, then strip trailing zeros
        String formatted = String.format("%.6f", value);
        // Strip trailing zeros after the decimal point
        if (formatted.contains(".")) {
            formatted = formatted.replaceAll("0+$", "");
            formatted = formatted.replaceAll("\\.$", "");
        }
        return formatted;
    }

    /**
     * If the value is a whole number (no fractional part), format as an integer.
     * Otherwise, format with up to 6 decimal places.
     *
     * @param value The value to format.
     * @return A human-readable string.
     */
    public static String wholeOrDecimal(double value) {
        if (Double.isNaN(value) || Double.isInfinite(value)) {
            return Double.toString(value);
        }
        if (value == Math.floor(value) && !Double.isInfinite(value)) {
            return Long.toString((long) value);
        }
        return decimalOrNan(value);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/TimeUtils.java` [MODIFIED]

```java
package dev.linhvu.micrometer;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

/**
 * Utility for converting time values between {@link TimeUnit}s using
 * {@code double} precision, avoiding the truncation that
 * {@link TimeUnit#convert(long, TimeUnit)} causes with {@code long} values.
 * <p>
 * JDK's {@code TimeUnit.convert()} operates on longs and truncates toward zero:
 * converting 500 milliseconds to seconds yields 0, not 0.5. This utility
 * converts through nanoseconds as a common denominator to preserve fractional
 * precision.
 */
public final class TimeUtils {

    private TimeUtils() {
    }

    /**
     * Convert a time value from one unit to another with double precision.
     *
     * @param value The time value to convert.
     * @param from  The source time unit.
     * @param to    The destination time unit.
     * @return The converted value.
     */
    public static double convert(double value, TimeUnit from, TimeUnit to) {
        if (from == to) {
            return value;
        }
        // Convert to nanoseconds, then to the target unit
        double nanos = value * nanosPerUnit(from);
        return nanos / nanosPerUnit(to);
    }

    /**
     * Formats a {@link Duration} into a human-readable string like "1h 23m 45.678s",
     * "150ms", or "42.5s". Only the two most significant non-zero units are shown.
     * <p>
     * This is used by {@code LoggingMeterRegistry} to display timer durations
     * in a compact, readable form.
     *
     * @param duration The duration to format.
     * @return A human-readable duration string.
     */
    public static String format(Duration duration) {
        long nanos = duration.toNanos();
        if (nanos == 0) {
            return "0s";
        }

        long hours = duration.toHours();
        long minutes = duration.toMinutesPart();
        long seconds = duration.toSecondsPart();
        long millis = duration.toMillisPart();
        long micros = (nanos % 1_000_000) / 1_000;
        long remainingNanos = nanos % 1_000;

        StringBuilder sb = new StringBuilder();
        if (hours > 0) {
            sb.append(hours).append("h ");
        }
        if (minutes > 0 || hours > 0) {
            if (hours > 0 || minutes > 0) {
                if (minutes > 0) {
                    sb.append(minutes).append("m ");
                }
            }
        }
        if (hours > 0 || minutes > 0) {
            if (seconds > 0 || millis > 0) {
                sb.append(formatSeconds(seconds, millis, 0, 0)).append("s");
            }
        }
        else if (seconds > 0) {
            sb.append(formatSeconds(seconds, millis, 0, 0)).append("s");
        }
        else if (millis > 0) {
            sb.append(formatSubUnit(millis, micros)).append("ms");
        }
        else if (micros > 0) {
            sb.append(formatSubUnit(micros, remainingNanos)).append("µs");
        }
        else {
            sb.append(remainingNanos).append("ns");
        }

        return sb.toString().trim();
    }

    private static String formatSeconds(long seconds, long millis, long micros, long nanos) {
        if (millis == 0) {
            return String.valueOf(seconds);
        }
        // Show fractional seconds: e.g., "1.234"
        String fraction = String.format("%03d", millis);
        fraction = fraction.replaceAll("0+$", "");
        return seconds + "." + fraction;
    }

    private static String formatSubUnit(long major, long minor) {
        if (minor == 0) {
            return String.valueOf(major);
        }
        String fraction = String.format("%03d", minor);
        fraction = fraction.replaceAll("0+$", "");
        return major + "." + fraction;
    }

    /**
     * Returns the number of nanoseconds per one unit of the given TimeUnit.
     */
    private static double nanosPerUnit(TimeUnit unit) {
        return switch (unit) {
            case NANOSECONDS -> 1.0;
            case MICROSECONDS -> 1_000.0;
            case MILLISECONDS -> 1_000_000.0;
            case SECONDS -> 1_000_000_000.0;
            case MINUTES -> 60_000_000_000.0;
            case HOURS -> 3_600_000_000_000.0;
            case DAYS -> 86_400_000_000_000.0;
        };
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/logging/LoggingRegistryConfig.java` [NEW]

```java
package dev.linhvu.micrometer.logging;

import dev.linhvu.micrometer.step.StepRegistryConfig;

/**
 * Configuration for the {@link LoggingMeterRegistry}.
 * <p>
 * Extends {@link StepRegistryConfig} — all step/push configuration (step interval,
 * enabled, batch size) applies. The only addition is {@link #logInactive()}, which
 * controls whether meters with zero activity are included in the output.
 * <p>
 * <b>Simplification:</b> The real Micrometer's config uses a string-key-based
 * property resolution system (e.g., "logging.logInactive" from environment variables
 * or properties files). We use a simple default method returning {@code false}.
 */
public interface LoggingRegistryConfig extends StepRegistryConfig {

    /**
     * Default configuration instance: step = 1 minute, enabled = true,
     * logInactive = false.
     */
    LoggingRegistryConfig DEFAULT = new LoggingRegistryConfig() {
    };

    /**
     * Whether to log meters that have zero activity (count = 0, value = 0, etc.)
     * during the reporting interval.
     * <p>
     * When {@code false} (the default), inactive counters, timers, etc. are
     * suppressed from the output — only meters with actual activity are logged.
     * Gauges are always logged regardless of this setting, because a gauge's
     * value of 0 may be meaningful (e.g., "zero active connections").
     *
     * @return {@code true} to log all meters, {@code false} to suppress inactive ones.
     */
    default boolean logInactive() {
        return false;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/logging/LoggingMeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.logging;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.DoubleFormat;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.TimeUtils;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.step.StepMeterRegistry;

import java.time.Duration;
import java.util.Comparator;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;
import java.util.stream.Collectors;

import static java.util.stream.Collectors.joining;

/**
 * A concrete push registry that publishes metrics by logging them — serves as
 * both a debugging tool and a reference push implementation.
 * <p>
 * Extends {@link StepMeterRegistry}, so all meters use step-based semantics:
 * counters report per-interval deltas, timers report per-interval counts and totals,
 * etc. The {@link #publish()} method formats each meter into a human-readable
 * string and sends it to a configurable logging sink.
 * <p>
 * The registry uses the <b>visitor pattern</b> ({@link Meter#use(Consumer, Consumer,
 * Consumer, Consumer, Consumer, Consumer, Consumer, Consumer)}) to dispatch
 * formatting logic for each meter type. Each type gets its own formatting style:
 * <ul>
 *   <li><b>Counter:</b> shows delta count and throughput (count/s)</li>
 *   <li><b>Timer:</b> shows delta count, throughput, mean time, and max time</li>
 *   <li><b>Gauge:</b> shows the current value</li>
 *   <li><b>DistributionSummary:</b> shows delta count, throughput, mean, and max</li>
 *   <li><b>LongTaskTimer:</b> shows active tasks, cumulative duration, mean, and max</li>
 *   <li><b>FunctionCounter:</b> same as Counter</li>
 *   <li><b>FunctionTimer:</b> shows delta count, throughput, and mean</li>
 * </ul>
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code LoggingMeterRegistry} uses
 * an {@code InternalLogger} framework and a {@code Builder} pattern with a
 * customizable {@code meterIdPrinter}. We use a simple {@code Consumer<String>}
 * sink (defaulting to {@code System.out::println}) and format meter IDs directly.
 */
public class LoggingMeterRegistry extends StepMeterRegistry {

    private final LoggingRegistryConfig config;

    private final Consumer<String> loggingSink;

    /**
     * Creates a registry with default config and {@code System.out::println} as the sink.
     */
    public LoggingMeterRegistry() {
        this(LoggingRegistryConfig.DEFAULT, Clock.SYSTEM);
    }

    /**
     * Creates a registry with the given config and clock, using {@code System.out::println}
     * as the logging sink.
     */
    public LoggingMeterRegistry(LoggingRegistryConfig config, Clock clock) {
        this(config, clock, System.out::println);
    }

    /**
     * Creates a registry with full configuration.
     *
     * @param config      The logging registry configuration.
     * @param clock       The clock for time measurement.
     * @param loggingSink Where formatted meter strings are sent (e.g., {@code System.out::println}
     *                    or {@code logger::info}).
     */
    public LoggingMeterRegistry(LoggingRegistryConfig config, Clock clock,
            Consumer<String> loggingSink) {
        super(config, clock);
        this.config = config;
        this.loggingSink = loggingSink;
        // Use dot naming convention — the most readable for logging output
        namingConvention(NamingConvention.dot);
    }

    // -----------------------------------------------------------------------
    // The core: publish() — formats all meters and sends to the sink
    // -----------------------------------------------------------------------

    /**
     * Iterates all registered meters, formats each one based on its type, and
     * sends the formatted string to the logging sink.
     * <p>
     * Meters are sorted first by type (Gauge, Counter, Timer, etc.), then
     * alphabetically by name within each type. This produces stable, predictable
     * output that is easy to scan visually.
     * <p>
     * When {@link LoggingRegistryConfig#logInactive()} is {@code false}, meters
     * with zero activity are silently skipped. Gauges are always logged because
     * their current value (even zero) is always meaningful.
     */
    @Override
    protected void publish() {
        if (!config.enabled()) {
            return;
        }

        List<Meter> meters = getMeters().stream()
                .sorted(Comparator
                        .comparing((Meter m) -> m.getId().getType().name())
                        .thenComparing(m -> m.getId().getName()))
                .toList();

        for (Meter meter : meters) {
            Printer print = new Printer(meter);
            meter.use(
                    counter -> {
                        double count = counter.count();
                        if (count == 0 && !config.logInactive()) {
                            return;
                        }
                        loggingSink.accept(print.id()
                                + " delta_count=" + print.humanReadableCount(count)
                                + " throughput=" + print.rate(count));
                    },
                    gauge -> {
                        // Gauges are always logged — check for TimeGauge first
                        if (gauge instanceof TimeGauge timeGauge) {
                            double value = timeGauge.value(getBaseTimeUnit());
                            if (value == 0 && !config.logInactive()) {
                                return;
                            }
                            loggingSink.accept(print.id()
                                    + " value=" + print.time(value));
                        }
                        else {
                            loggingSink.accept(print.id()
                                    + " value=" + print.value(gauge.value()));
                        }
                    },
                    timer -> {
                        HistogramSnapshot snapshot = timer.takeSnapshot();
                        long count = snapshot.count();
                        if (count == 0 && !config.logInactive()) {
                            return;
                        }
                        loggingSink.accept(print.id()
                                + " delta_count=" + count
                                + " throughput=" + print.unitlessRate(count)
                                + " mean=" + print.time(snapshot.mean(getBaseTimeUnit()))
                                + " max=" + print.time(snapshot.max(getBaseTimeUnit())));
                    },
                    summary -> {
                        HistogramSnapshot snapshot = summary.takeSnapshot();
                        long count = snapshot.count();
                        if (count == 0 && !config.logInactive()) {
                            return;
                        }
                        loggingSink.accept(print.id()
                                + " delta_count=" + count
                                + " throughput=" + print.unitlessRate(count)
                                + " mean=" + print.value(snapshot.mean())
                                + " max=" + print.value(snapshot.max()));
                    },
                    longTaskTimer -> {
                        int activeTasks = longTaskTimer.activeTasks();
                        if (activeTasks == 0 && !config.logInactive()) {
                            return;
                        }
                        loggingSink.accept(print.id()
                                + " active=" + activeTasks
                                + " duration=" + print.time(longTaskTimer.duration(getBaseTimeUnit()))
                                + " mean=" + print.time(longTaskTimer.mean(getBaseTimeUnit()))
                                + " max=" + print.time(longTaskTimer.max(getBaseTimeUnit())));
                    },
                    functionCounter -> {
                        double count = functionCounter.count();
                        if (count == 0 && !config.logInactive()) {
                            return;
                        }
                        loggingSink.accept(print.id()
                                + " delta_count=" + print.humanReadableCount(count)
                                + " throughput=" + print.rate(count));
                    },
                    functionTimer -> {
                        double count = functionTimer.count();
                        if (count == 0 && !config.logInactive()) {
                            return;
                        }
                        loggingSink.accept(print.id()
                                + " delta_count=" + (long) count
                                + " throughput=" + print.unitlessRate(count)
                                + " mean=" + print.time(functionTimer.mean(getBaseTimeUnit())));
                    },
                    defaultMeter -> {
                        // Fallback for custom meter types
                        loggingSink.accept(writeMeter(defaultMeter, print));
                    }
            );
        }
    }

    /**
     * Formats a generic/custom meter by iterating its measurements.
     * Used as the fallback for meter types that don't match any specific visitor slot.
     */
    String writeMeter(Meter meter, Printer print) {
        StringBuilder sb = new StringBuilder(print.id());
        for (Measurement measurement : meter.measure()) {
            Statistic stat = measurement.getStatistic();
            String name = stat.getTagValueRepresentation();
            double value = measurement.getValue();
            sb.append(' ').append(name).append('=');
            switch (stat) {
                case TOTAL_TIME, DURATION -> sb.append(print.time(value));
                case TOTAL, MAX, VALUE -> sb.append(print.value(value));
                case COUNT -> sb.append(print.humanReadableCount(value));
                default -> sb.append(DoubleFormat.decimalOrNan(value));
            }
        }
        return sb.toString();
    }

    // -----------------------------------------------------------------------
    // Base time unit — milliseconds for logging readability
    // -----------------------------------------------------------------------

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.MILLISECONDS;
    }

    // -----------------------------------------------------------------------
    // Printer — per-meter formatting helper
    // -----------------------------------------------------------------------

    /**
     * Helper class that formats values in the context of a specific meter.
     * <p>
     * Created per-meter in {@link #publish()} to capture the meter's identity
     * (convention name + tags) and base unit for value formatting.
     */
    class Printer {

        private final Meter meter;

        Printer(Meter meter) {
            this.meter = meter;
        }

        /**
         * Formats the meter's identity: convention name + tags in curly braces.
         * Example: {@code "http.server.requests{method=GET,status=200}"}
         */
        String id() {
            String conventionName = meter.getId().getConventionName(getNamingConvention());
            List<Tag> conventionTags = meter.getId().getConventionTags(getNamingConvention());
            String tagString = conventionTags.stream()
                    .map(t -> t.getKey() + "=" + t.getValue())
                    .collect(joining(",", "{", "}"));
            return conventionName + tagString;
        }

        /**
         * Formats a time value in the registry's base time unit as a human-readable
         * duration string (e.g., "1.234s", "150ms").
         */
        String time(double timeInBaseUnit) {
            // Convert from base time unit to nanoseconds, then format as Duration
            double nanos = TimeUtils.convert(timeInBaseUnit, getBaseTimeUnit(), TimeUnit.NANOSECONDS);
            return TimeUtils.format(Duration.ofNanos((long) nanos));
        }

        /**
         * Formats a throughput rate as "X/s". Includes the base unit if present.
         */
        String rate(double count) {
            double perSecond = count / (config.step().toMillis() / 1000.0);
            return humanReadableBaseUnit(perSecond) + "/s";
        }

        /**
         * Formats a unitless throughput rate as "X/s".
         */
        String unitlessRate(double count) {
            double perSecond = count / (config.step().toMillis() / 1000.0);
            return DoubleFormat.decimalOrNan(perSecond) + "/s";
        }

        /**
         * Formats a value with its base unit suffix (if any).
         */
        String value(double value) {
            return humanReadableBaseUnit(value);
        }

        /**
         * Formats a count value as a human-readable string with base unit.
         */
        String humanReadableCount(double count) {
            return humanReadableBaseUnit(count);
        }

        /**
         * Appends the meter's base unit to the value, or just formats the value
         * if no base unit is set. Special-cases "bytes" for human-readable
         * byte formatting (KiB, MiB, GiB).
         */
        private String humanReadableBaseUnit(double value) {
            String baseUnit = meter.getId().getBaseUnit();
            if ("bytes".equals(baseUnit)) {
                return humanReadableByteCount(value);
            }
            String formatted = DoubleFormat.wholeOrDecimal(value);
            if (baseUnit != null) {
                return formatted + " " + baseUnit;
            }
            return formatted;
        }

        /**
         * Formats a byte count as a human-readable string using binary units
         * (KiB, MiB, GiB, etc.).
         */
        private String humanReadableByteCount(double bytes) {
            if (bytes < 1024) {
                return DoubleFormat.wholeOrDecimal(bytes) + " B";
            }
            int exp = (int) (Math.log(bytes) / Math.log(1024));
            String unit = "KMGTPE".charAt(exp - 1) + "iB";
            return String.format("%.1f %s", bytes / Math.pow(1024, exp), unit);
        }

    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/DoubleFormatTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DoubleFormatTest {

    @Test
    void shouldFormatWholeNumber_WithoutDecimalPoint() {
        assertThat(DoubleFormat.decimalOrNan(42.0)).isEqualTo("42");
    }

    @Test
    void shouldFormatFractionalNumber_StrippingTrailingZeros() {
        assertThat(DoubleFormat.decimalOrNan(3.14)).isEqualTo("3.14");
    }

    @Test
    void shouldFormatNaN_AsNaNString() {
        assertThat(DoubleFormat.decimalOrNan(Double.NaN)).isEqualTo("NaN");
    }

    @Test
    void shouldFormatInfinity_AsInfinityString() {
        assertThat(DoubleFormat.decimalOrNan(Double.POSITIVE_INFINITY)).isEqualTo("Infinity");
        assertThat(DoubleFormat.decimalOrNan(Double.NEGATIVE_INFINITY)).isEqualTo("-Infinity");
    }

    @Test
    void shouldFormatZero_AsZero() {
        assertThat(DoubleFormat.decimalOrNan(0.0)).isEqualTo("0");
    }

    @Test
    void shouldFormatSmallFraction_WithPrecision() {
        assertThat(DoubleFormat.decimalOrNan(0.000001)).isEqualTo("0.000001");
    }

    @Test
    void shouldReturnWholeNumber_WhenNoFractionalPart() {
        assertThat(DoubleFormat.wholeOrDecimal(100.0)).isEqualTo("100");
    }

    @Test
    void shouldReturnDecimal_WhenFractionalPartPresent() {
        assertThat(DoubleFormat.wholeOrDecimal(100.5)).isEqualTo("100.5");
    }

    @Test
    void shouldReturnNaN_WhenNaN() {
        assertThat(DoubleFormat.wholeOrDecimal(Double.NaN)).isEqualTo("NaN");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/TimeUtilsFormatTest.java` [NEW]

```java
package dev.linhvu.micrometer;

import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link TimeUtils#format(Duration)}.
 */
class TimeUtilsFormatTest {

    @Test
    void shouldFormatZeroDuration_AsZeroSeconds() {
        assertThat(TimeUtils.format(Duration.ZERO)).isEqualTo("0s");
    }

    @Test
    void shouldFormatNanoseconds() {
        assertThat(TimeUtils.format(Duration.ofNanos(500))).isEqualTo("500ns");
    }

    @Test
    void shouldFormatMicroseconds() {
        assertThat(TimeUtils.format(Duration.ofNanos(1_500_000))).isEqualTo("1.5ms");
    }

    @Test
    void shouldFormatMilliseconds() {
        assertThat(TimeUtils.format(Duration.ofMillis(150))).isEqualTo("150ms");
    }

    @Test
    void shouldFormatSeconds() {
        assertThat(TimeUtils.format(Duration.ofSeconds(5))).isEqualTo("5s");
    }

    @Test
    void shouldFormatSecondsWithMillis() {
        assertThat(TimeUtils.format(Duration.ofMillis(1_500))).isEqualTo("1.5s");
    }

    @Test
    void shouldFormatMinutesAndSeconds() {
        assertThat(TimeUtils.format(Duration.ofSeconds(65))).isEqualTo("1m 5s");
    }

    @Test
    void shouldFormatHoursMinutesAndSeconds() {
        assertThat(TimeUtils.format(Duration.ofSeconds(3661))).isEqualTo("1h 1m 1s");
    }

    @Test
    void shouldFormatWholeMinutes() {
        assertThat(TimeUtils.format(Duration.ofMinutes(5))).isEqualTo("5m");
    }

    @Test
    void shouldFormatWholeHours() {
        assertThat(TimeUtils.format(Duration.ofHours(2))).isEqualTo("2h");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/logging/LoggingMeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.logging;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.Timer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link LoggingMeterRegistry}.
 */
class LoggingMeterRegistryTest {

    private MockClock clock;
    private List<String> logLines;
    private LoggingMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        logLines = Collections.synchronizedList(new ArrayList<>());
    }

    @AfterEach
    void tearDown() {
        if (registry != null && !registry.isClosed()) {
            registry.close();
        }
    }

    private LoggingMeterRegistry createRegistry() {
        return createRegistry(LoggingRegistryConfig.DEFAULT);
    }

    private LoggingMeterRegistry createRegistry(LoggingRegistryConfig config) {
        registry = new LoggingMeterRegistry(config, clock, logLines::add);
        return registry;
    }

    @Test
    void shouldFormatCounter_WithDeltaCountAndThroughput() {
        LoggingMeterRegistry reg = createRegistry();
        Counter counter = reg.counter("http.requests");
        counter.increment(10);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("http.requests");
        assertThat(logLines.get(0)).contains("delta_count=");
        assertThat(logLines.get(0)).contains("throughput=");
    }

    @Test
    void shouldSkipInactiveCounter_WhenLogInactiveIsFalse() {
        LoggingMeterRegistry reg = createRegistry();
        reg.counter("idle.counter");
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).isEmpty();
    }

    @Test
    void shouldLogInactiveCounter_WhenLogInactiveIsTrue() {
        LoggingRegistryConfig config = new LoggingRegistryConfig() {
            @Override public boolean logInactive() { return true; }
        };
        LoggingMeterRegistry reg = createRegistry(config);
        reg.counter("idle.counter");
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("idle.counter");
    }

    @Test
    void shouldFormatGauge_WithCurrentValue() {
        LoggingMeterRegistry reg = createRegistry();
        AtomicLong value = new AtomicLong(42);
        reg.gauge("pool.size", value);
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("pool.size");
        assertThat(logLines.get(0)).contains("value=42");
    }

    @Test
    void shouldAlwaysLogGauge_EvenWhenValueIsZero() {
        LoggingMeterRegistry reg = createRegistry();
        AtomicLong value = new AtomicLong(0);
        reg.gauge("queue.depth", value);
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("queue.depth");
    }

    @Test
    void shouldFormatTimer_WithCountMeanAndMax() {
        LoggingMeterRegistry reg = createRegistry();
        Timer timer = reg.timer("http.latency");
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("http.latency");
        assertThat(logLines.get(0)).contains("delta_count=2");
        assertThat(logLines.get(0)).contains("mean=");
        assertThat(logLines.get(0)).contains("max=");
    }

    @Test
    void shouldSkipInactiveTimer_WhenLogInactiveIsFalse() {
        LoggingMeterRegistry reg = createRegistry();
        reg.timer("idle.timer");
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).isEmpty();
    }

    @Test
    void shouldFormatSummary_WithCountMeanAndMax() {
        LoggingMeterRegistry reg = createRegistry();
        DistributionSummary summary = reg.summary("payload.size");
        summary.record(100.0);
        summary.record(200.0);
        summary.record(300.0);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("payload.size");
        assertThat(logLines.get(0)).contains("delta_count=3");
        assertThat(logLines.get(0)).contains("mean=");
        assertThat(logLines.get(0)).contains("max=");
    }

    @Test
    void shouldFormatLongTaskTimer_WithActiveTasksAndDuration() {
        LoggingMeterRegistry reg = createRegistry();
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import").register(reg);
        LongTaskTimer.Sample sample = ltt.start();
        clock.add(5, TimeUnit.SECONDS);
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("batch.import");
        assertThat(logLines.get(0)).contains("active=1");
        assertThat(logLines.get(0)).contains("duration=");
        sample.stop();
    }

    @Test
    void shouldSkipInactiveLongTaskTimer_WhenLogInactiveIsFalse() {
        LoggingMeterRegistry reg = createRegistry();
        LongTaskTimer.builder("idle.batch").register(reg);
        reg.publish();
        assertThat(logLines).isEmpty();
    }

    @Test
    void shouldFormatTimeGauge_WithFormattedTimeValue() {
        LoggingMeterRegistry reg = createRegistry();
        AtomicLong uptimeMs = new AtomicLong(65_000);
        TimeGauge.builder("jvm.uptime", uptimeMs, TimeUnit.MILLISECONDS,
                AtomicLong::doubleValue).register(reg);
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("jvm.uptime");
        assertThat(logLines.get(0)).contains("value=");
    }

    @Test
    void shouldIncludeTags_InOutputFormat() {
        LoggingMeterRegistry reg = createRegistry();
        Counter counter = Counter.builder("http.requests")
                .tag("method", "GET").tag("status", "200").register(reg);
        counter.increment(5);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("method=GET");
        assertThat(logLines.get(0)).contains("status=200");
    }

    @Test
    void shouldFormatBytesUnit_AsHumanReadable() {
        LoggingMeterRegistry reg = createRegistry();
        DistributionSummary summary = DistributionSummary.builder("payload.size")
                .baseUnit("bytes").register(reg);
        summary.record(1_048_576);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("MiB");
    }

    @Test
    void shouldNotPublish_WhenDisabled() {
        LoggingRegistryConfig config = new LoggingRegistryConfig() {
            @Override public boolean enabled() { return false; }
        };
        LoggingMeterRegistry reg = createRegistry(config);
        reg.counter("test.counter").increment(5);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).isEmpty();
    }

    @Test
    void shouldUseMillisecondsAsBaseTimeUnit() {
        LoggingMeterRegistry reg = createRegistry();
        assertThat(reg.getBaseTimeUnit()).isEqualTo(TimeUnit.MILLISECONDS);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/logging/LoggingRegistryIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.logging;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.config.NamingConvention;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for {@link LoggingMeterRegistry} with the full meter
 * registration pipeline — MeterFilter, NamingConvention, step-based rollover,
 * and the publish lifecycle.
 */
class LoggingRegistryIntegrationTest {

    private MockClock clock;
    private List<String> logLines;
    private LoggingMeterRegistry registry;

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        logLines = Collections.synchronizedList(new ArrayList<>());
    }

    @AfterEach
    void tearDown() {
        if (registry != null && !registry.isClosed()) { registry.close(); }
    }

    private LoggingMeterRegistry createRegistry() {
        return createRegistry(LoggingRegistryConfig.DEFAULT);
    }

    private LoggingMeterRegistry createRegistry(LoggingRegistryConfig config) {
        registry = new LoggingMeterRegistry(config, clock, logLines::add);
        return registry;
    }

    @Test
    void shouldPublishStepValues_WhenFullLifecycleCompletes() {
        LoggingMeterRegistry reg = createRegistry();
        Counter counter = reg.counter("http.requests");
        Timer timer = reg.timer("http.latency");
        counter.increment(10);
        timer.record(150, TimeUnit.MILLISECONDS);
        timer.record(250, TimeUnit.MILLISECONDS);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(2);
        assertThat(logLines.stream().filter(l -> l.contains("http.requests")).count()).isEqualTo(1);
        assertThat(logLines.stream().filter(l -> l.contains("http.latency")).count()).isEqualTo(1);
    }

    @Test
    void shouldApplyCommonTags_WhenFilterConfigured() {
        LoggingMeterRegistry reg = createRegistry();
        reg.meterFilter(MeterFilter.commonTags(Tags.of("env", "prod")));
        Counter counter = reg.counter("http.requests");
        counter.increment(5);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("env=prod");
    }

    @Test
    void shouldNotPublishDeniedMeter_WhenFilterDenies() {
        LoggingMeterRegistry reg = createRegistry();
        reg.meterFilter(MeterFilter.denyNameStartsWith("internal."));
        Counter published = reg.counter("http.requests");
        Counter denied = reg.counter("internal.debug");
        published.increment(5);
        denied.increment(10);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("http.requests");
    }

    @Test
    void shouldApplyNamingConvention_WhenCustomConventionSet() {
        LoggingMeterRegistry reg = createRegistry();
        reg.namingConvention(NamingConvention.snakeCase);
        Counter counter = Counter.builder("http.server.requests")
                .tag("http.method", "GET").register(reg);
        counter.increment(3);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("http_server_requests");
        assertThat(logLines.get(0)).contains("http_method=GET");
    }

    @Test
    void shouldPublishDifferentValues_AcrossSteps() {
        LoggingMeterRegistry reg = createRegistry();
        Counter counter = reg.counter("requests");
        counter.increment(10);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines.get(0)).contains("delta_count=10");
        logLines.clear();
        counter.increment(3);
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines.get(0)).contains("delta_count=3");
    }

    @Test
    void shouldPublishOnClose_IncludingPartialStepData() {
        LoggingMeterRegistry reg = createRegistry();
        Counter counter = reg.counter("requests");
        counter.increment(7);
        reg.close();
        assertThat(logLines).isNotEmpty();
    }

    @Test
    void shouldPublishFunctionCounterDelta_AcrossSteps() {
        LoggingMeterRegistry reg = createRegistry();
        AtomicLong source = new AtomicLong(100);
        FunctionCounter fc = FunctionCounter.builder("cache.evictions", source, AtomicLong::get)
                .register(reg);
        fc.count();
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines.get(0)).contains("cache.evictions");
        logLines.clear();
        source.set(150);
        fc.count();
        clock.add(Duration.ofMinutes(1));
        reg.publish();
        assertThat(logLines.get(0)).contains("delta_count=50");
    }

    @Test
    void shouldFormatTimeGauge_WithTimeValue() {
        LoggingMeterRegistry reg = createRegistry();
        AtomicLong uptimeMs = new AtomicLong(65_000);
        TimeGauge.builder("jvm.uptime", uptimeMs, TimeUnit.MILLISECONDS,
                AtomicLong::doubleValue).register(reg);
        reg.publish();
        assertThat(logLines).hasSize(1);
        assertThat(logLines.get(0)).contains("jvm.uptime");
        assertThat(logLines.get(0)).contains("value=");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **LoggingMeterRegistry** | A concrete `StepMeterRegistry` that implements `publish()` by formatting meters as human-readable log lines |
| **LoggingRegistryConfig** | Extends `StepRegistryConfig` with a single `logInactive()` toggle |
| **Visitor pattern in publish()** | `Meter.use()` dispatches to 8 type-specific consumers for custom formatting per meter type |
| **Printer inner class** | Per-meter formatting helper that knows the meter's identity, base unit, and naming convention |
| **logInactive** | When `false` (default), meters with zero activity are suppressed; Gauges are always logged |
| **Consumer\<String\> loggingSink** | Strategy pattern — the output destination is pluggable (`System.out::println`, `logger::info`, `list::add`) |
| **DoubleFormat** | Utility for formatting doubles without trailing zeros, handling NaN/Infinity |
| **TimeUtils.format(Duration)** | Human-readable duration formatting: "1h 23m 45.678s", "150ms", etc. |
| **getBaseTimeUnit() = MILLISECONDS** | All time values in the logging output are in milliseconds |

**Next: Chapter 18 — Observation API** — A higher-level instrumentation API that unifies metrics, tracing, and logging under a single observation lifecycle.
