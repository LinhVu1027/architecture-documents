# Chapter 20: Prometheus Registry

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have push-based registries (StepMeterRegistry, LoggingMeterRegistry) that export metrics on a schedule | No way to expose metrics for scrape/pull-based monitoring systems like Prometheus — the most widely used monitoring backend in cloud-native environments | Build a `PrometheusMeterRegistry` that generates Prometheus text exposition format on demand, with cumulative histograms and Prometheus-compatible naming |

---

## 20.1 The Integration Point

The Prometheus registry plugs into the existing system through `MeterRegistry`'s template method pattern — the same extension point used by `SimpleMeterRegistry`, `StepMeterRegistry`, and `LoggingMeterRegistry`. But unlike push registries that override `publish()`, Prometheus introduces a fundamentally different export model: **pull**.

**New file:** `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusMeterRegistry.java`

The integration point is the set of abstract factory methods in `MeterRegistry`:

```java
public class PrometheusMeterRegistry extends MeterRegistry {

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new PrometheusCounter(id);
    }

    @Override
    protected Timer newTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new PrometheusTimer(id, clock, distributionStatisticConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new PrometheusDistributionSummary(id, clock, scale,
                distributionStatisticConfig);
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;  // Prometheus convention
    }

    // The new method — not in MeterRegistry, unique to Prometheus
    public String scrape() { ... }
}
```

Two key decisions here:

1. **Pull, not push:** Instead of a `publish()` method called by a scheduler (like `PushMeterRegistry`), Prometheus adds a `scrape()` method that returns the text exposition format. There is no scheduler — the external Prometheus server controls when data is collected.

2. **Cumulative histograms:** Standard Micrometer histograms use rolling windows (2-minute expiry, 3 buffers). Prometheus requires lifetime-cumulative histograms. We solve this by creating custom meter types (`PrometheusTimer`, `PrometheusDistributionSummary`) that manage their own cumulative histograms.

This connects the **Prometheus text format generator** to the **MeterRegistry lifecycle**. To make it work, we need to build:
- `PrometheusConfig` — configuration for the registry
- `PrometheusNamingConvention` — Prometheus-compatible naming (snake_case, `_seconds` suffix, etc.)
- `PrometheusCounter` — cumulative counter (no rolling window)
- `PrometheusTimer` — timer with cumulative histogram
- `PrometheusDistributionSummary` — distribution summary with cumulative histogram
- `PrometheusMeterRegistry` — the `scrape()` method that generates Prometheus text format

## 20.2 Configuration

**New file:** `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusConfig.java`

```java
public interface PrometheusConfig {

    PrometheusConfig DEFAULT = key -> null;

    String get(String key);

    default boolean descriptions() {
        String val = get("prometheus.descriptions");
        return val == null || Boolean.parseBoolean(val);
    }

    default Duration step() {
        String val = get("prometheus.step");
        return val == null ? Duration.ofMinutes(1) : Duration.parse(val);
    }
}
```

Unlike push registry configs where `step()` controls the publish interval, here `step()` controls the **window size for rolling statistics** like `max`. It should match your Prometheus scrape interval so that the reported max reflects the most recent scrape period.

## 20.3 Prometheus Naming Convention

**New file:** `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusNamingConvention.java`

Prometheus has strict naming rules that differ from Micrometer's dot-notation:

```java
public class PrometheusNamingConvention implements NamingConvention {

    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        String conventionName = name.replace('.', '_');  // dots → underscores

        switch (type) {
            case TIMER:
            case LONG_TASK_TIMER:
                if (!conventionName.endsWith("_seconds")) {
                    conventionName += "_seconds";
                }
                break;
            case COUNTER:
            case DISTRIBUTION_SUMMARY:
            case GAUGE:
                if (baseUnit != null && !conventionName.endsWith("_" + baseUnit)) {
                    conventionName += "_" + baseUnit;
                }
                break;
        }

        return sanitizeMetricName(conventionName);
    }

    @Override
    public String tagKey(String key) {
        return sanitizeLabelName(key.replace('.', '_'));
    }
}
```

Key rules:
- **Dots become underscores:** `http.server.requests` → `http_server_requests`
- **Timers get `_seconds`:** `http.request` → `http_request_seconds`
- **Base unit suffix appended:** `jvm.memory.used` with unit `bytes` → `jvm_memory_used_bytes`
- **`_total` is NOT added here** — it's added during scrape output (matching the real Prometheus client library behavior)
- **Names sanitized to `[a-zA-Z_:][a-zA-Z0-9_:]*`**, labels to `[a-zA-Z_][a-zA-Z0-9_]*`

## 20.4 Prometheus-Specific Meter Types

### PrometheusCounter

**New file:** `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusCounter.java`

```java
public class PrometheusCounter extends AbstractMeter implements Counter {

    private final DoubleAdder count = new DoubleAdder();

    public PrometheusCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public void increment(double amount) {
        if (amount > 0) {
            count.add(amount);
        }
    }

    @Override
    public double count() {
        return count.doubleValue();
    }
}
```

Prometheus counters are simple — they accumulate forever. No rolling window, no step reset. The Prometheus server computes rates from the raw cumulative values.

### PrometheusTimer

**New file:** `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusTimer.java`

This is the most interesting class. The challenge: Prometheus requires **cumulative** histograms, but `AbstractTimer.defaultHistogram()` creates rolling-window histograms.

```java
public class PrometheusTimer extends AbstractTimer {

    private final LongAdder count = new LongAdder();
    private final LongAdder totalNanos = new LongAdder();
    private final TimeWindowMax max;
    private final Histogram cumulativeHistogram;

    public PrometheusTimer(Meter.Id id, Clock clock, DistributionStatisticConfig config) {
        // Pass NoopHistogram to parent — we manage our own
        super(id, clock, TimeUnit.SECONDS);

        this.max = new TimeWindowMax(clock);

        if (config.isPublishingHistogram()) {
            // The trick: 5-year expiry + 1 buffer = never rotates
            DistributionStatisticConfig cumulativeConfig = DistributionStatisticConfig.builder()
                    .serviceLevelObjectives(config.getServiceLevelObjectives())
                    .expiry(Duration.ofDays(1825))
                    .bufferLength(1)
                    .build();
            this.cumulativeHistogram = new TimeWindowFixedBoundaryHistogram(
                    clock, cumulativeConfig);
        } else {
            this.cumulativeHistogram = NoopHistogram.INSTANCE;
        }
    }

    @Override
    protected void recordNonNegative(long amount, TimeUnit unit) {
        long nanos = unit.toNanos(amount);
        count.increment();
        totalNanos.add(nanos);
        max.record((double) nanos, TimeUnit.NANOSECONDS);
        cumulativeHistogram.recordLong(nanos);  // Record into our cumulative histogram
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        // Use our cumulative histogram, not the parent's rolling one
        return cumulativeHistogram.takeSnapshot(
                count(), totalTime(TimeUnit.NANOSECONDS), max(TimeUnit.NANOSECONDS));
    }
}
```

The key trick: we reuse `TimeWindowFixedBoundaryHistogram` with a ~5-year expiry and buffer length 1. The ring buffer never rotates, so counts accumulate forever. This is the same approach the real Micrometer uses via `PrometheusHistogram`.

### PrometheusDistributionSummary

**New file:** `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusDistributionSummary.java`

Same cumulative histogram pattern as `PrometheusTimer`, but for non-time values (request sizes, payload lengths, etc.). No time-unit conversion — bucket boundaries are in raw units.

## 20.5 The Scrape Method

The `scrape()` method is where everything comes together. It iterates all registered meters, groups them by convention name (creating Prometheus "metric families"), and formats each family according to the Prometheus text exposition format.

```java
public String scrape() {
    StringBuilder sb = new StringBuilder();
    NamingConvention convention = getNamingConvention();

    // Group meters by convention name → Prometheus metric families
    Map<String, List<Meter>> grouped = new LinkedHashMap<>();
    for (Meter meter : getMeters()) {
        String conventionName = meter.getId().getConventionName(convention);
        grouped.computeIfAbsent(conventionName, k -> new ArrayList<>()).add(meter);
    }

    for (Map.Entry<String, List<Meter>> entry : grouped.entrySet()) {
        writeMeterFamily(sb, entry.getKey(), entry.getValue(), convention);
    }

    return sb.toString();
}
```

Each meter type maps to a specific Prometheus type:

| Micrometer Type | Prometheus Type | Suffixes |
|----------------|-----------------|----------|
| Counter | counter | `_total` |
| Gauge | gauge | (none) |
| Timer (no histogram) | summary | `_count`, `_sum` + `_max` gauge |
| Timer (with SLOs) | histogram | `_bucket{le=...}`, `_count`, `_sum` + `_max` gauge |
| DistributionSummary | summary/histogram | Same as Timer but no time conversion |
| FunctionCounter | counter | `_total` |
| FunctionTimer | summary | `_count`, `_sum` |
| LongTaskTimer | gauge | `_active_count`, `_duration_sum`, `_max` |

The histogram output includes cumulative bucket counts with `le` (less-than-or-equal) labels, plus a mandatory `+Inf` bucket:

```
# TYPE http_request_seconds histogram
http_request_seconds_bucket{method="GET",le="0.05"} 24054
http_request_seconds_bucket{method="GET",le="0.1"} 33444
http_request_seconds_bucket{method="GET",le="+Inf"} 133988
http_request_seconds_sum{method="GET"} 53423.0
http_request_seconds_count{method="GET"} 133988
```

## 20.6 Try It Yourself

<details>
<summary>Challenge 1: Implement the Prometheus text format for a counter</summary>

Given a counter named `http.requests` with tag `method=GET` and count 42, write the expected Prometheus text output:

```
# HELP http_requests_total
# TYPE http_requests_total counter
http_requests_total{method="GET"} 42
```

Key details:
- Name converted from dots to underscores
- `_total` suffix added for counters
- Tags become `{key="value"}` labels
- Values are on the same line after a space

</details>

<details>
<summary>Challenge 2: Why does Prometheus need cumulative histograms?</summary>

Prometheus scrapes metrics at intervals (e.g., every 15 seconds). Between scrapes, the server has no visibility into what happened. If histograms used rolling windows:

1. **Data loss between scrapes:** A rolling window might reset while Prometheus hasn't scraped yet, losing observations.
2. **Rate computation breaks:** Prometheus computes rates as `(current - previous) / time_delta`. If values roll/reset unpredictably, the rate computation produces garbage.

Cumulative histograms solve this: values only go up (monotonically increasing). The Prometheus server can compute accurate rates from any two scrape points. Application restarts are detected as counter resets and handled gracefully.

The trick in our code: `TimeWindowFixedBoundaryHistogram` with 5-year expiry and buffer length 1 = effectively cumulative forever.

</details>

<details>
<summary>Challenge 3: Implement the label formatting with le label for histograms</summary>

For histogram buckets, the `le` label must be added alongside existing tags:

```java
private String formatLabelsWithLe(Meter.Id id, NamingConvention convention, String le) {
    List<Tag> tags = id.getConventionTags(convention);
    StringBuilder sb = new StringBuilder("{");
    for (Tag tag : tags) {
        sb.append(tag.getKey()).append("=\"")
          .append(escapeLabelValue(tag.getValue())).append("\",");
    }
    sb.append("le=\"").append(le).append("\"}");
    return sb.toString();
}
```

Note: the `le` label is always last, and if there are other tags, they come first with a trailing comma.

</details>

## 20.7 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusNamingConventionTest.java`

```java
@Test
void shouldConvertDotsToUnderscores() {
    String result = convention.name("http.server.requests", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("http_server_requests");
}

@Test
void shouldAppendSecondsForTimers() {
    String result = convention.name("http.request", Meter.Type.TIMER, null);
    assertThat(result).isEqualTo("http_request_seconds");
}

@Test
void shouldNotDoubleAppendSecondsForTimers() {
    String result = convention.name("http.request.seconds", Meter.Type.TIMER, null);
    assertThat(result).isEqualTo("http_request_seconds");
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusTimerTest.java`

```java
@Test
void shouldProduceCumulativeHistogramWithSlos() {
    DistributionStatisticConfig config = DistributionStatisticConfig.builder()
            .serviceLevelObjectives(
                    Duration.ofMillis(50).toNanos(),
                    Duration.ofMillis(100).toNanos(),
                    Duration.ofMillis(500).toNanos())
            .build()
            .merge(DistributionStatisticConfig.DEFAULT);

    PrometheusTimer timer = createTimer(config);
    timer.record(30, TimeUnit.MILLISECONDS);   // <= 50ms
    timer.record(80, TimeUnit.MILLISECONDS);   // <= 100ms
    timer.record(200, TimeUnit.MILLISECONDS);  // <= 500ms
    timer.record(800, TimeUnit.MILLISECONDS);  // > 500ms

    HistogramSnapshot snapshot = timer.takeSnapshot();
    CountAtBucket[] counts = snapshot.histogramCounts();

    assertThat(counts[0].count()).isEqualTo(1);  // <= 50ms: 1
    assertThat(counts[1].count()).isEqualTo(2);  // <= 100ms: 2 (cumulative)
    assertThat(counts[2].count()).isEqualTo(3);  // <= 500ms: 3 (cumulative)
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusMeterRegistryTest.java`

```java
@Test
void shouldScrapeTimerAsHistogram_WhenSlosConfigured() {
    Timer timer = Timer.builder("http.request")
            .serviceLevelObjectives(
                    Duration.ofMillis(50),
                    Duration.ofMillis(100),
                    Duration.ofMillis(500))
            .register(registry);

    timer.record(30, TimeUnit.MILLISECONDS);
    timer.record(80, TimeUnit.MILLISECONDS);
    timer.record(200, TimeUnit.MILLISECONDS);

    String scrape = registry.scrape();

    assertThat(scrape).contains("# TYPE http_request_seconds histogram");
    assertThat(scrape).contains("http_request_seconds_bucket{le=\"0.05\"} 1");
    assertThat(scrape).contains("http_request_seconds_bucket{le=\"0.1\"} 2");
    assertThat(scrape).contains("http_request_seconds_bucket{le=\"0.5\"} 3");
    assertThat(scrape).contains("http_request_seconds_bucket{le=\"+Inf\"} 3");
}
```

### Integration Tests

**New file:** `src/test/java/dev/linhvu/micrometer/integration/PrometheusRegistryIntegrationTest.java`

```java
@Test
void shouldShowCumulativeHistogram_AcrossMultipleScrapes() {
    Timer timer = Timer.builder("latency")
            .serviceLevelObjectives(Duration.ofMillis(100))
            .register(registry);

    timer.record(50, TimeUnit.MILLISECONDS);
    String scrape1 = registry.scrape();
    assertThat(scrape1).contains("latency_seconds_bucket{le=\"0.1\"} 1");

    timer.record(50, TimeUnit.MILLISECONDS);
    String scrape2 = registry.scrape();
    // Cumulative: bucket count should be 2, not reset to 1
    assertThat(scrape2).contains("latency_seconds_bucket{le=\"0.1\"} 2");
}
```

**Run:** `./gradlew test` — expected: all tests pass (including all prior features' tests)

---

## 20.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Pull vs Push — a fundamental architectural choice:**
> - Push registries (Datadog, OTLP) need step-based meters that compute rates client-side and publish deltas. The registry controls *when* data flows.
> - Pull registries (Prometheus) need cumulative meters where values only go up. The server computes rates from raw monotonic counters using `rate()` or `increase()`. The registry is passive — it just answers when asked.
> - This is why Prometheus counters never reset (except on restart), and histograms accumulate forever. Two scrapes at any interval can always compute an accurate rate via `(current - previous) / time_delta`.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Cumulative histograms via expiry hack:**
> Rather than implementing a separate cumulative histogram class, we reuse `TimeWindowFixedBoundaryHistogram` with a 5-year expiry and buffer length 1. The ring buffer has one slot that never rotates, so counts accumulate forever. This is the exact same trick the real Micrometer uses in `PrometheusHistogram` — elegant code reuse that avoids duplicating the entire histogram infrastructure.
> The trade-off: memory grows unboundedly over the application's lifetime. In practice, this is negligible — histogram memory is proportional to the number of buckets (typically 5-20), not the number of observations.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Text exposition format — simplicity as a feature:**
> Prometheus chose a text format (not JSON, not Protobuf) for its exposition endpoint. This makes it trivially debuggable — `curl http://app:8080/metrics` shows exactly what Prometheus sees. No external dependencies, no serialization libraries, no schema definitions. The format is simple enough that our entire scrape implementation is a single method with helper functions. The real cost of this simplicity: it's slower to parse at scale, which is why OpenMetrics (the successor format) adds Protobuf support.
> -----------------------------------------------------------

## 20.9 What We Enhanced

| Aspect | Before (ch17) | Current (this chapter) | Real Framework |
|--------|---------------|----------------------|----------------|
| Export model | Push only (scheduled publish) | Push AND pull (scrape on demand) | Same — `PrometheusMeterRegistry` extends `MeterRegistry` directly, not `PushMeterRegistry` (`PrometheusMeterRegistry.java:87`) |
| Histogram model | Rolling window (2-min expiry) | Rolling (push) + cumulative (Prometheus) | Same — `PrometheusHistogram` uses 5-year expiry (`PrometheusHistogram.java:43`) |
| Naming | Generic conventions (dot, snake, camel) | Backend-specific convention (snake_case + `_seconds`/`_total`/unit suffixes) | `PrometheusNamingConvention` delegates to `PrometheusNaming.prometheusName()` for full camelCase-to-snake conversion (`PrometheusNamingConvention.java:41`) |
| Output format | Logged to PrintStream | Prometheus text exposition format 0.0.4 | Also supports OpenMetrics, Protobuf, exemplars, and uses the Prometheus client library's `ExpositionFormats` (`PrometheusMeterRegistry.java:130`) |
| Collector bridge | Direct iteration of meters | Direct iteration of meters | Uses `MicrometerCollector` implementing Prometheus `MultiCollector` for proper metric family aggregation (`MicrometerCollector.java:36`) |

## 20.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `PrometheusMeterRegistry` | `PrometheusMeterRegistry` | `PrometheusMeterRegistry.java:87` | Real version uses Prometheus client library's `PrometheusRegistry` and `ExpositionFormats`; supports OpenMetrics, exemplars, filtered scrapes by metric name |
| `PrometheusConfig` | `PrometheusConfig` | `PrometheusConfig.java:25` | Real version extends `MeterRegistryConfig`, includes `prometheusProperties()` for client library config |
| `PrometheusNamingConvention` | `PrometheusNamingConvention` | `PrometheusNamingConvention.java:30` | Real version delegates to `PrometheusNaming.prometheusName()` which handles camelCase→snake_case; has configurable `timerSuffix` |
| `PrometheusCounter` | `PrometheusCounter` | `PrometheusCounter.java:32` | Real version includes `ExemplarSampler` for trace-to-metrics linking |
| `PrometheusTimer` | `PrometheusTimer` | `PrometheusTimer.java:40` | Real version uses `PrometheusHistogram` (a separate class), supports exemplars per histogram bucket, delegates to the Prometheus client library for snapshot model |
| `PrometheusDistributionSummary` | `PrometheusDistributionSummary` | `PrometheusDistributionSummary.java:33` | Same structure as `PrometheusTimer` — cumulative histogram + optional exemplar support |
| `scrape()` with direct text generation | `scrape()` via `MicrometerCollector` | `PrometheusMeterRegistry.java:125` | Real version goes through `MicrometerCollector.collect()` → `MetricSnapshots` → `ExpositionFormats.write()` pipeline; supports content negotiation and metric name filtering |
| (not implemented) | `MicrometerCollector` | `MicrometerCollector.java:36` | Bridges Micrometer's per-meter model to Prometheus's per-metric-family model; handles multiple tag combinations as `Child` instances within a single collector |

## 20.11 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusConfig.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import java.time.Duration;

/**
 * Configuration for the {@link PrometheusMeterRegistry}.
 * <p>
 * Unlike push-based registries (Datadog, InfluxDB) that have a {@code step()} controlling
 * how often metrics are published, Prometheus is <b>pull-based</b> — an external Prometheus
 * server scrapes the {@link PrometheusMeterRegistry#scrape()} endpoint at its own interval.
 * <p>
 * The {@link #step()} here controls the <b>window size</b> for rolling statistics like
 * {@code max}. It should match the Prometheus scrape interval so that the max value
 * reported at scrape time reflects the most recent scrape period.
 */
public interface PrometheusConfig {

    /**
     * Default configuration — accepts all defaults.
     */
    PrometheusConfig DEFAULT = key -> null;

    /**
     * Looks up a configuration value by key. Returns null if not found,
     * which causes the default to be used.
     */
    String get(String key);

    /**
     * Whether to include meter descriptions in the {@code # HELP} line of the
     * Prometheus text exposition output. Default: true.
     */
    default boolean descriptions() {
        String val = get("prometheus.descriptions");
        return val == null || Boolean.parseBoolean(val);
    }

    /**
     * The step size (window duration) for rolling statistics like max.
     * Should match the Prometheus scrape interval. Default: 1 minute.
     * <p>
     * This is NOT a publish interval (Prometheus is pull, not push).
     * It controls only the time window for windowed statistics.
     */
    default Duration step() {
        String val = get("prometheus.step");
        return val == null ? Duration.ofMinutes(1) : Duration.parse(val);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusNamingConvention.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.config.NamingConvention;

/**
 * Naming convention for Prometheus — converts Micrometer's dot-notation names into
 * Prometheus-compatible snake_case names with appropriate unit suffixes.
 * <p>
 * Prometheus has strict naming rules:
 * <ul>
 *   <li>Metric names must match {@code [a-zA-Z_:][a-zA-Z0-9_:]*}</li>
 *   <li>Label names must match {@code [a-zA-Z_][a-zA-Z0-9_]*}</li>
 *   <li>Timer/duration metrics must use seconds as the base unit</li>
 *   <li>Unit suffixes should be appended to the metric name (e.g., {@code _bytes}, {@code _seconds})</li>
 * </ul>
 * <p>
 * <b>Important:</b> This convention does NOT append {@code _total} to counter names.
 * The {@code _total} suffix is added during scrape output generation, matching the
 * real Prometheus client library's behavior.
 * <p>
 * <b>Simplification:</b> The real {@code PrometheusNamingConvention} delegates to
 * {@code PrometheusNaming.prometheusName()} which handles camelCase-to-snake_case
 * conversion (e.g., "httpRequests" → "http_requests"). We only handle dot-to-underscore
 * conversion since Micrometer recommends dot-notation in user code.
 */
public class PrometheusNamingConvention implements NamingConvention {

    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        String conventionName = toSnakeCase(name);

        switch (type) {
            case TIMER:
            case LONG_TASK_TIMER:
                // Prometheus requires duration metrics in seconds
                if (!conventionName.endsWith("_seconds")) {
                    conventionName += "_seconds";
                }
                break;
            case COUNTER:
            case DISTRIBUTION_SUMMARY:
            case GAUGE:
                // Append base unit suffix if present and not already there
                if (baseUnit != null && !conventionName.endsWith("_" + baseUnit)) {
                    conventionName += "_" + baseUnit;
                }
                break;
            default:
                break;
        }

        return sanitizeMetricName(conventionName);
    }

    @Override
    public String tagKey(String key) {
        return sanitizeLabelName(toSnakeCase(key));
    }

    /**
     * Converts dot-notation to snake_case by replacing dots with underscores.
     */
    private String toSnakeCase(String name) {
        return name.replace('.', '_');
    }

    /**
     * Ensures the metric name matches {@code [a-zA-Z_:][a-zA-Z0-9_:]*}.
     * Replaces any invalid character with underscore.
     */
    private String sanitizeMetricName(String name) {
        if (name.isEmpty()) {
            return name;
        }
        StringBuilder sb = new StringBuilder(name.length());
        for (int i = 0; i < name.length(); i++) {
            char c = name.charAt(i);
            if (i == 0) {
                sb.append(Character.isLetter(c) || c == '_' || c == ':' ? c : '_');
            }
            else {
                sb.append(Character.isLetterOrDigit(c) || c == '_' || c == ':' ? c : '_');
            }
        }
        return sb.toString();
    }

    /**
     * Ensures the label name matches {@code [a-zA-Z_][a-zA-Z0-9_]*}.
     * Replaces any invalid character with underscore.
     * <p>
     * Note: label names starting with {@code __} are reserved for Prometheus internal use,
     * but we don't enforce this restriction.
     */
    private String sanitizeLabelName(String name) {
        if (name.isEmpty()) {
            return name;
        }
        StringBuilder sb = new StringBuilder(name.length());
        for (int i = 0; i < name.length(); i++) {
            char c = name.charAt(i);
            if (i == 0) {
                sb.append(Character.isLetter(c) || c == '_' ? c : '_');
            }
            else {
                sb.append(Character.isLetterOrDigit(c) || c == '_' ? c : '_');
            }
        }
        return sb.toString();
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusCounter.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.AbstractMeter;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;

import java.util.Collections;
import java.util.concurrent.atomic.DoubleAdder;

/**
 * A Prometheus-specific counter implementation that accumulates values over
 * the entire application lifetime.
 * <p>
 * Unlike the rolling-window counters used by step-based push registries,
 * Prometheus counters are <b>monotonically increasing</b> and <b>cumulative</b>.
 * The Prometheus server computes rates on the server side using the raw
 * cumulative values — this is why Prometheus counters never reset to zero
 * (except on application restart, which Prometheus handles via {@code _created}
 * timestamps in OpenMetrics format).
 * <p>
 * Uses a {@link DoubleAdder} for lock-free concurrent accumulation — the same
 * approach as the real {@code PrometheusCounter}.
 */
public class PrometheusCounter extends AbstractMeter implements Counter {

    private final DoubleAdder count = new DoubleAdder();

    public PrometheusCounter(Meter.Id id) {
        super(id);
    }

    @Override
    public void increment(double amount) {
        if (amount > 0) {
            count.add(amount);
        }
    }

    @Override
    public double count() {
        return count.doubleValue();
    }

    @Override
    public Iterable<Measurement> measure() {
        return Collections.singletonList(new Measurement(this::count, Statistic.COUNT));
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusTimer.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.AbstractTimer;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.Histogram;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.distribution.NoopHistogram;
import dev.linhvu.micrometer.distribution.TimeWindowFixedBoundaryHistogram;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;

/**
 * A Prometheus-specific timer with <b>cumulative</b> histogram support.
 * <p>
 * The key difference from {@link dev.linhvu.micrometer.cumulative.CumulativeTimer}
 * is the histogram model. Standard Micrometer histograms use a rolling time window
 * (e.g., 2-minute window with 3 buffers), which is ideal for push-based registries
 * that export snapshots periodically. Prometheus, however, requires <b>lifetime-cumulative</b>
 * histograms — each bucket count includes ALL observations since the application started.
 * <p>
 * This is achieved by creating a {@link TimeWindowFixedBoundaryHistogram} with a
 * very long expiry (~5 years) and a buffer length of 1, making it effectively
 * a non-rolling histogram that accumulates forever. This matches the approach
 * used by the real {@code PrometheusTimer} via {@code PrometheusHistogram}.
 * <p>
 * <b>Why not use the parent's histogram?</b> The parent's
 * {@link AbstractTimer#defaultHistogram} creates a rolling-window histogram
 * using the registry's default config (step-sized expiry). We need a separate
 * cumulative histogram, so we pass {@link NoopHistogram} to the parent and
 * manage recording ourselves.
 */
public class PrometheusTimer extends AbstractTimer {

    private final LongAdder count = new LongAdder();

    private final LongAdder totalNanos = new LongAdder();

    private final TimeWindowMax max;

    private final Histogram cumulativeHistogram;

    public PrometheusTimer(Meter.Id id, Clock clock, DistributionStatisticConfig config) {
        // Pass NoopHistogram to parent — we manage our own cumulative histogram
        super(id, clock, TimeUnit.SECONDS);

        this.max = new TimeWindowMax(clock);

        if (config.isPublishingHistogram()) {
            // Create a cumulative histogram: 5-year expiry + 1 buffer = never rotates
            DistributionStatisticConfig cumulativeConfig = DistributionStatisticConfig.builder()
                    .serviceLevelObjectives(config.getServiceLevelObjectives())
                    .expiry(Duration.ofDays(1825))
                    .bufferLength(1)
                    .build();
            this.cumulativeHistogram = new TimeWindowFixedBoundaryHistogram(clock, cumulativeConfig);
        }
        else {
            this.cumulativeHistogram = NoopHistogram.INSTANCE;
        }
    }

    @Override
    protected void recordNonNegative(long amount, TimeUnit unit) {
        long nanos = unit.toNanos(amount);
        count.increment();
        totalNanos.add(nanos);
        max.record((double) nanos, TimeUnit.NANOSECONDS);
        cumulativeHistogram.recordLong(nanos);
    }

    @Override
    public long count() {
        return count.longValue();
    }

    @Override
    public double totalTime(TimeUnit unit) {
        return totalNanos.doubleValue() / unit.toNanos(1);
    }

    @Override
    public double max(TimeUnit unit) {
        return max.poll(unit);
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return cumulativeHistogram.takeSnapshot(count(),
                totalTime(TimeUnit.NANOSECONDS), max(TimeUnit.NANOSECONDS));
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusDistributionSummary.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.AbstractDistributionSummary;
import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.Histogram;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.distribution.NoopHistogram;
import dev.linhvu.micrometer.distribution.TimeWindowFixedBoundaryHistogram;
import dev.linhvu.micrometer.distribution.TimeWindowMax;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

/**
 * A Prometheus-specific distribution summary with <b>cumulative</b> histogram support.
 * <p>
 * Follows the same cumulative histogram pattern as {@link PrometheusTimer} — a
 * {@link TimeWindowFixedBoundaryHistogram} with ~5-year expiry that effectively
 * accumulates over the application's lifetime.
 * <p>
 * <b>Key difference from PrometheusTimer:</b> No time-unit conversion. Values
 * are raw doubles (bytes, items, request sizes, etc.), and histogram bucket
 * boundaries are in the same unit as the recorded values.
 */
public class PrometheusDistributionSummary extends AbstractDistributionSummary {

    private final LongAdder count = new LongAdder();

    private final DoubleAdder total = new DoubleAdder();

    private final TimeWindowMax max;

    private final Histogram cumulativeHistogram;

    public PrometheusDistributionSummary(Meter.Id id, Clock clock, double scale,
            DistributionStatisticConfig config) {
        // Pass NoopHistogram to parent via the scale-only constructor
        super(id, scale);

        this.max = new TimeWindowMax(clock);

        if (config.isPublishingHistogram()) {
            DistributionStatisticConfig cumulativeConfig = DistributionStatisticConfig.builder()
                    .serviceLevelObjectives(config.getServiceLevelObjectives())
                    .expiry(Duration.ofDays(1825))
                    .bufferLength(1)
                    .build();
            this.cumulativeHistogram = new TimeWindowFixedBoundaryHistogram(clock, cumulativeConfig);
        }
        else {
            this.cumulativeHistogram = NoopHistogram.INSTANCE;
        }
    }

    @Override
    protected void recordNonNegative(double amount) {
        count.increment();
        total.add(amount);
        max.record(amount);
        cumulativeHistogram.recordDouble(amount);
    }

    @Override
    public long count() {
        return count.longValue();
    }

    @Override
    public double totalAmount() {
        return total.sum();
    }

    @Override
    public double max() {
        return max.poll();
    }

    @Override
    public HistogramSnapshot takeSnapshot() {
        return cumulativeHistogram.takeSnapshot(count(), totalAmount(), max());
    }

    @Override
    public Iterable<Measurement> measure() {
        return Arrays.asList(
                new Measurement(() -> (double) count(), Statistic.COUNT),
                new Measurement(this::totalAmount, Statistic.TOTAL),
                new Measurement(this::max, Statistic.MAX));
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/prometheus/PrometheusMeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MeterRegistry;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.NamingConvention;
import dev.linhvu.micrometer.cumulative.CumulativeFunctionCounter;
import dev.linhvu.micrometer.cumulative.CumulativeFunctionTimer;
import dev.linhvu.micrometer.distribution.CountAtBucket;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import dev.linhvu.micrometer.internal.DefaultGauge;
import dev.linhvu.micrometer.internal.DefaultLongTaskTimer;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.function.ToDoubleFunction;
import java.util.function.ToLongFunction;

/**
 * A pull/scrape-based registry that exposes metrics in the
 * <b>Prometheus text exposition format</b> (version 0.0.4).
 * <p>
 * Unlike push-based registries ({@code StepMeterRegistry}, {@code LoggingMeterRegistry})
 * that periodically send metrics to a remote backend, this registry implements the
 * <b>pull model</b>: an external Prometheus server calls the {@link #scrape()} method
 * at its own interval (typically via an HTTP endpoint like {@code /actuator/prometheus}).
 * <p>
 * Key differences from other registries:
 * <ul>
 *   <li><b>Base time unit is seconds</b> — Prometheus convention for all durations</li>
 *   <li><b>Counters get {@code _total} suffix</b> — Prometheus counter naming convention</li>
 *   <li><b>Histograms are cumulative</b> — each bucket includes all observations &le; its boundary,
 *       and values accumulate over the application's lifetime (no rolling window)</li>
 *   <li><b>Names are snake_case</b> — via {@link PrometheusNamingConvention}</li>
 *   <li><b>No scheduler</b> — no publish thread, no step interval for export</li>
 * </ul>
 * <p>
 * <b>Simplification:</b> The real {@code PrometheusMeterRegistry} uses the Prometheus
 * Java client library ({@code io.prometheus:prometheus-metrics-core}) with a
 * {@code MicrometerCollector} bridge. We generate the text format directly without
 * any external dependency. We also omit exemplar support, OpenMetrics format,
 * and the {@code _created} timestamp.
 */
public class PrometheusMeterRegistry extends MeterRegistry {

    /**
     * The content type for Prometheus text exposition format 0.0.4.
     */
    public static final String CONTENT_TYPE_004 = "text/plain; version=0.0.4; charset=utf-8";

    private final PrometheusConfig prometheusConfig;

    public PrometheusMeterRegistry(PrometheusConfig config) {
        this(config, Clock.SYSTEM);
    }

    public PrometheusMeterRegistry(PrometheusConfig config, Clock clock) {
        super(clock);
        this.prometheusConfig = config;
        namingConvention(new PrometheusNamingConvention());
    }

    // -----------------------------------------------------------------------
    // Scrape — the heart of the pull model
    // -----------------------------------------------------------------------

    /**
     * Returns all registered metrics in the Prometheus text exposition format
     * (version 0.0.4).
     * <p>
     * This method is typically exposed via an HTTP endpoint (e.g., {@code /metrics}
     * or {@code /actuator/prometheus}) and called by the Prometheus server at its
     * configured scrape interval.
     * <p>
     * The output groups meters by their convention name into metric families,
     * with each family having a {@code # HELP} line (if descriptions are enabled),
     * a {@code # TYPE} line, and one or more sample lines.
     *
     * @return Prometheus text exposition format string
     */
    public String scrape() {
        StringBuilder sb = new StringBuilder();
        NamingConvention convention = getNamingConvention();

        // Group meters by convention name to form Prometheus "metric families"
        Map<String, List<Meter>> grouped = new LinkedHashMap<>();
        for (Meter meter : getMeters()) {
            String conventionName = meter.getId().getConventionName(convention);
            grouped.computeIfAbsent(conventionName, k -> new ArrayList<>()).add(meter);
        }

        for (Map.Entry<String, List<Meter>> entry : grouped.entrySet()) {
            writeMeterFamily(sb, entry.getKey(), entry.getValue(), convention);
        }

        return sb.toString();
    }

    // -----------------------------------------------------------------------
    // Per-type scrape formatters
    // -----------------------------------------------------------------------

    private void writeMeterFamily(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        Meter first = meters.get(0);
        first.use(
                counter -> writeCounter(sb, name, meters, convention),
                gauge -> writeGauge(sb, name, meters, convention),
                timer -> writeTimer(sb, name, meters, convention),
                summary -> writeSummary(sb, name, meters, convention),
                ltt -> writeLongTaskTimer(sb, name, meters, convention),
                fc -> writeFunctionCounter(sb, name, meters, convention),
                ft -> writeFunctionTimer(sb, name, meters, convention),
                custom -> writeCustomMeter(sb, name, meters, convention));
    }

    /**
     * Formats counters. Prometheus counters have a {@code _total} suffix:
     * <pre>
     * # HELP http_requests_total Total HTTP requests.
     * # TYPE http_requests_total counter
     * http_requests_total{method="GET"} 1234
     * </pre>
     */
    private void writeCounter(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        String totalName = name.endsWith("_total") ? name : name + "_total";
        writeHelp(sb, totalName, meters.get(0));
        writeType(sb, totalName, "counter");
        for (Meter meter : meters) {
            Counter counter = (Counter) meter;
            writeSample(sb, totalName, formatLabels(meter.getId(), convention),
                    counter.count());
        }
    }

    /**
     * Formats gauges. Gauges with NaN or infinite values are skipped.
     */
    private void writeGauge(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        writeHelp(sb, name, meters.get(0));
        writeType(sb, name, "gauge");
        for (Meter meter : meters) {
            Gauge gauge = (Gauge) meter;
            double value = gauge.value();
            if (Double.isFinite(value)) {
                writeSample(sb, name, formatLabels(meter.getId(), convention), value);
            }
        }
    }

    /**
     * Formats timers as either histogram (if SLO buckets configured) or summary.
     * Always includes a separate {@code _max} gauge.
     * <p>
     * Histogram format:
     * <pre>
     * # TYPE http_request_seconds histogram
     * http_request_seconds_bucket{le="0.05"} 24054
     * http_request_seconds_bucket{le="+Inf"} 133988
     * http_request_seconds_sum 53423.0
     * http_request_seconds_count 133988
     * </pre>
     */
    private void writeTimer(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        // Check histogram vs summary using the first timer
        Timer firstTimer = (Timer) meters.get(0);
        boolean hasHistogram = firstTimer.takeSnapshot().histogramCounts().length > 0;

        writeHelp(sb, name, meters.get(0));
        writeType(sb, name, hasHistogram ? "histogram" : "summary");

        for (Meter meter : meters) {
            Timer timer = (Timer) meter;
            String labels = formatLabels(meter.getId(), convention);

            if (hasHistogram) {
                HistogramSnapshot snapshot = timer.takeSnapshot();
                writeHistogramBuckets(sb, name, meter.getId(), convention,
                        snapshot.histogramCounts(), timer.count(), true);
            }

            writeSample(sb, name + "_count", labels, timer.count());
            writeSample(sb, name + "_sum", labels, timer.totalTime(getBaseTimeUnit()));
        }

        // Max as a separate gauge family
        writeMaxGauge(sb, name + "_max", meters, convention,
                meter -> ((Timer) meter).max(getBaseTimeUnit()));
    }

    /**
     * Formats distribution summaries — same structure as timers but without
     * time-unit conversion for bucket boundaries.
     */
    private void writeSummary(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        DistributionSummary firstSummary = (DistributionSummary) meters.get(0);
        boolean hasHistogram = firstSummary.takeSnapshot().histogramCounts().length > 0;

        writeHelp(sb, name, meters.get(0));
        writeType(sb, name, hasHistogram ? "histogram" : "summary");

        for (Meter meter : meters) {
            DistributionSummary summary = (DistributionSummary) meter;
            String labels = formatLabels(meter.getId(), convention);

            if (hasHistogram) {
                HistogramSnapshot snapshot = summary.takeSnapshot();
                writeHistogramBuckets(sb, name, meter.getId(), convention,
                        snapshot.histogramCounts(), summary.count(), false);
            }

            writeSample(sb, name + "_count", labels, summary.count());
            writeSample(sb, name + "_sum", labels, summary.totalAmount());
        }

        writeMaxGauge(sb, name + "_max", meters, convention,
                meter -> ((DistributionSummary) meter).max());
    }

    /**
     * Formats long task timers as gauges for active count, duration, and max.
     */
    private void writeLongTaskTimer(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        // Active count
        writeType(sb, name + "_active_count", "gauge");
        for (Meter meter : meters) {
            LongTaskTimer ltt = (LongTaskTimer) meter;
            writeSample(sb, name + "_active_count",
                    formatLabels(meter.getId(), convention), ltt.activeTasks());
        }

        // Duration sum
        writeType(sb, name + "_duration_sum", "gauge");
        for (Meter meter : meters) {
            LongTaskTimer ltt = (LongTaskTimer) meter;
            writeSample(sb, name + "_duration_sum",
                    formatLabels(meter.getId(), convention),
                    ltt.duration(getBaseTimeUnit()));
        }

        // Max
        writeMaxGauge(sb, name + "_max", meters, convention,
                meter -> ((LongTaskTimer) meter).max(getBaseTimeUnit()));
    }

    /**
     * Formats function counters — same as regular counters.
     */
    private void writeFunctionCounter(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        String totalName = name.endsWith("_total") ? name : name + "_total";
        writeHelp(sb, totalName, meters.get(0));
        writeType(sb, totalName, "counter");
        for (Meter meter : meters) {
            FunctionCounter fc = (FunctionCounter) meter;
            double value = fc.count();
            if (Double.isFinite(value)) {
                writeSample(sb, totalName, formatLabels(meter.getId(), convention), value);
            }
        }
    }

    /**
     * Formats function timers as summaries (count + sum, no quantiles).
     */
    private void writeFunctionTimer(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        writeHelp(sb, name, meters.get(0));
        writeType(sb, name, "summary");
        for (Meter meter : meters) {
            FunctionTimer ft = (FunctionTimer) meter;
            String labels = formatLabels(meter.getId(), convention);
            writeSample(sb, name + "_count", labels, ft.count());
            writeSample(sb, name + "_sum", labels, ft.totalTime(getBaseTimeUnit()));
        }
    }

    /**
     * Formats custom meters as untyped — each measurement as a separate line.
     */
    private void writeCustomMeter(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention) {
        writeHelp(sb, name, meters.get(0));
        writeType(sb, name, "untyped");
        for (Meter meter : meters) {
            for (Measurement m : meter.measure()) {
                String suffix = "_" + m.getStatistic().getTagValueRepresentation();
                writeSample(sb, name + suffix,
                        formatLabels(meter.getId(), convention), m.getValue());
            }
        }
    }

    // -----------------------------------------------------------------------
    // Scrape helpers
    // -----------------------------------------------------------------------

    /**
     * Writes histogram bucket lines with the {@code le} (less-than-or-equal) label.
     * Always adds a {@code +Inf} bucket if the last explicit bucket is finite.
     */
    private void writeHistogramBuckets(StringBuilder sb, String name,
            Meter.Id id, NamingConvention convention,
            CountAtBucket[] counts, long totalCount, boolean isTimer) {
        for (CountAtBucket cab : counts) {
            double boundary = isTimer ? cab.bucket(getBaseTimeUnit()) : cab.bucket();
            String le = formatDouble(boundary);
            sb.append(name).append("_bucket")
                    .append(formatLabelsWithLe(id, convention, le))
                    .append(' ')
                    .append(formatLong((long) cab.count()))
                    .append('\n');
        }

        // +Inf bucket — required by Prometheus, contains the total count
        if (counts.length == 0
                || counts[counts.length - 1].bucket() != Double.POSITIVE_INFINITY) {
            sb.append(name).append("_bucket")
                    .append(formatLabelsWithLe(id, convention, "+Inf"))
                    .append(' ')
                    .append(formatLong(totalCount))
                    .append('\n');
        }
    }

    /**
     * Writes a max gauge as a separate metric family (TYPE gauge, no HELP).
     */
    private void writeMaxGauge(StringBuilder sb, String name,
            List<Meter> meters, NamingConvention convention,
            ToDoubleFunction<Meter> maxFunction) {
        writeType(sb, name, "gauge");
        for (Meter meter : meters) {
            writeSample(sb, name, formatLabels(meter.getId(), convention),
                    maxFunction.applyAsDouble(meter));
        }
    }

    private void writeHelp(StringBuilder sb, String name, Meter meter) {
        if (prometheusConfig.descriptions()) {
            String desc = meter.getId().getDescription();
            if (desc != null && !desc.isEmpty()) {
                sb.append("# HELP ").append(name).append(' ')
                        .append(escapeHelp(desc)).append('\n');
            }
        }
    }

    private void writeType(StringBuilder sb, String name, String type) {
        sb.append("# TYPE ").append(name).append(' ').append(type).append('\n');
    }

    private void writeSample(StringBuilder sb, String name, String labels, double value) {
        sb.append(name).append(labels).append(' ')
                .append(formatDouble(value)).append('\n');
    }

    private void writeSample(StringBuilder sb, String name, String labels, long value) {
        sb.append(name).append(labels).append(' ')
                .append(formatLong(value)).append('\n');
    }

    // -----------------------------------------------------------------------
    // Label formatting
    // -----------------------------------------------------------------------

    /**
     * Formats meter tags as Prometheus labels: {@code {key1="value1",key2="value2"}}.
     * Returns empty string if no tags.
     */
    private String formatLabels(Meter.Id id, NamingConvention convention) {
        List<Tag> tags = id.getConventionTags(convention);
        if (tags.isEmpty()) {
            return "";
        }
        StringBuilder sb = new StringBuilder("{");
        for (int i = 0; i < tags.size(); i++) {
            if (i > 0) {
                sb.append(',');
            }
            Tag tag = tags.get(i);
            sb.append(tag.getKey()).append("=\"")
                    .append(escapeLabelValue(tag.getValue())).append('"');
        }
        sb.append('}');
        return sb.toString();
    }

    /**
     * Formats labels with an additional {@code le} (less-than-or-equal) label
     * for histogram buckets. The {@code le} label is always last.
     */
    private String formatLabelsWithLe(Meter.Id id, NamingConvention convention,
            String le) {
        List<Tag> tags = id.getConventionTags(convention);
        StringBuilder sb = new StringBuilder("{");
        for (Tag tag : tags) {
            sb.append(tag.getKey()).append("=\"")
                    .append(escapeLabelValue(tag.getValue())).append("\",");
        }
        sb.append("le=\"").append(le).append("\"}");
        return sb.toString();
    }

    // -----------------------------------------------------------------------
    // Value formatting
    // -----------------------------------------------------------------------

    /**
     * Formats a double for Prometheus text format.
     * Handles special values (+Inf, NaN) and avoids unnecessary ".0" for integers.
     */
    static String formatDouble(double value) {
        if (Double.isNaN(value)) {
            return "NaN";
        }
        if (value == Double.POSITIVE_INFINITY) {
            return "+Inf";
        }
        if (value == Double.NEGATIVE_INFINITY) {
            return "-Inf";
        }
        // Whole numbers: format without decimal point
        if (value == Math.floor(value) && !Double.isInfinite(value)
                && Math.abs(value) < 1e15) {
            return Long.toString((long) value);
        }
        return Double.toString(value);
    }

    private static String formatLong(long value) {
        return Long.toString(value);
    }

    // -----------------------------------------------------------------------
    // Escaping
    // -----------------------------------------------------------------------

    /**
     * Escapes a HELP line string: backslash and newline.
     */
    private String escapeHelp(String help) {
        return help.replace("\\", "\\\\").replace("\n", "\\n");
    }

    /**
     * Escapes a label value: backslash, double-quote, and newline.
     */
    private String escapeLabelValue(String value) {
        return value.replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n");
    }

    // -----------------------------------------------------------------------
    // Template Method factories
    // -----------------------------------------------------------------------

    @Override
    protected Counter newCounter(Meter.Id id) {
        return new PrometheusCounter(id);
    }

    @Override
    protected <T> Gauge newGauge(Meter.Id id, T obj, ToDoubleFunction<T> valueFunction) {
        return new DefaultGauge<>(id, obj, valueFunction);
    }

    @Override
    protected Timer newTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new PrometheusTimer(id, clock, distributionStatisticConfig);
    }

    @Override
    protected DistributionSummary newDistributionSummary(Meter.Id id, double scale,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new PrometheusDistributionSummary(id, clock, scale,
                distributionStatisticConfig);
    }

    @Override
    protected LongTaskTimer newLongTaskTimer(Meter.Id id,
            DistributionStatisticConfig distributionStatisticConfig) {
        return new DefaultLongTaskTimer(id, clock, getBaseTimeUnit());
    }

    @Override
    protected <T> FunctionCounter newFunctionCounter(Meter.Id id, T obj,
            ToDoubleFunction<T> countFunction) {
        return new CumulativeFunctionCounter<>(id, obj, countFunction);
    }

    @Override
    protected <T> FunctionTimer newFunctionTimer(Meter.Id id, T obj,
            ToLongFunction<T> countFunction, ToDoubleFunction<T> totalTimeFunction,
            TimeUnit totalTimeFunctionUnit) {
        return new CumulativeFunctionTimer<>(id, obj, countFunction,
                totalTimeFunction, totalTimeFunctionUnit, getBaseTimeUnit());
    }

    /**
     * Prometheus uses <b>seconds</b> as the base time unit — all duration values
     * (timer totals, max, histogram bucket boundaries) are exported in seconds.
     */
    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.SECONDS;
    }

    /**
     * Uses the config's step duration as the histogram expiry, matching it to
     * the Prometheus scrape interval so windowed statistics (like max) reflect
     * the most recent scrape period.
     */
    @Override
    protected DistributionStatisticConfig defaultHistogramConfig() {
        return DistributionStatisticConfig.builder()
                .expiry(prometheusConfig.step())
                .build()
                .merge(DistributionStatisticConfig.DEFAULT);
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusNamingConventionTest.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Meter;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PrometheusNamingConventionTest {

    private final PrometheusNamingConvention convention = new PrometheusNamingConvention();

    @Test
    void shouldConvertDotsToUnderscores() {
        String result = convention.name("http.server.requests", Meter.Type.COUNTER, null);
        assertThat(result).isEqualTo("http_server_requests");
    }

    @Test
    void shouldAppendSecondsForTimers() {
        String result = convention.name("http.request", Meter.Type.TIMER, null);
        assertThat(result).isEqualTo("http_request_seconds");
    }

    @Test
    void shouldNotDoubleAppendSecondsForTimers() {
        String result = convention.name("http.request.seconds", Meter.Type.TIMER, null);
        assertThat(result).isEqualTo("http_request_seconds");
    }

    @Test
    void shouldAppendSecondsForLongTaskTimers() {
        String result = convention.name("batch.import", Meter.Type.LONG_TASK_TIMER, null);
        assertThat(result).isEqualTo("batch_import_seconds");
    }

    @Test
    void shouldAppendBaseUnitForCounters() {
        String result = convention.name("data.transferred", Meter.Type.COUNTER, "bytes");
        assertThat(result).isEqualTo("data_transferred_bytes");
    }

    @Test
    void shouldNotDoubleAppendBaseUnitForCounters() {
        String result = convention.name("data.transferred.bytes", Meter.Type.COUNTER, "bytes");
        assertThat(result).isEqualTo("data_transferred_bytes");
    }

    @Test
    void shouldAppendBaseUnitForGauges() {
        String result = convention.name("jvm.memory.used", Meter.Type.GAUGE, "bytes");
        assertThat(result).isEqualTo("jvm_memory_used_bytes");
    }

    @Test
    void shouldAppendBaseUnitForDistributionSummaries() {
        String result = convention.name("http.request.size", Meter.Type.DISTRIBUTION_SUMMARY, "bytes");
        assertThat(result).isEqualTo("http_request_size_bytes");
    }

    @Test
    void shouldNotAppendUnitWhenNull() {
        String result = convention.name("http.requests", Meter.Type.COUNTER, null);
        assertThat(result).isEqualTo("http_requests");
    }

    @Test
    void shouldHandleOtherType() {
        String result = convention.name("custom.meter", Meter.Type.OTHER, null);
        assertThat(result).isEqualTo("custom_meter");
    }

    @Test
    void shouldSanitizeMetricNameStartingWithDigit() {
        String result = convention.name("1invalid.name", Meter.Type.GAUGE, null);
        assertThat(result).isEqualTo("_invalid_name");
    }

    @Test
    void shouldPreserveColonsInMetricName() {
        String result = convention.name("namespace:metric", Meter.Type.GAUGE, null);
        assertThat(result).isEqualTo("namespace:metric");
    }

    @Test
    void shouldConvertTagKeyDotsToUnderscores() {
        String result = convention.tagKey("http.method");
        assertThat(result).isEqualTo("http_method");
    }

    @Test
    void shouldSanitizeTagKeyStartingWithDigit() {
        String result = convention.tagKey("1key");
        assertThat(result).isEqualTo("_key");
    }

    @Test
    void shouldSanitizeTagKeyWithSpecialChars() {
        String result = convention.tagKey("tag-key");
        assertThat(result).isEqualTo("tag_key");
    }

    @Test
    void shouldNotModifyAlreadyValidTagKey() {
        String result = convention.tagKey("method");
        assertThat(result).isEqualTo("method");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusCounterTest.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.Tags;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PrometheusCounterTest {

    private final Meter.Id id = new Meter.Id("test.counter", Tags.empty(),
            Meter.Type.COUNTER, null, null);

    @Test
    void shouldStartAtZero() {
        PrometheusCounter counter = new PrometheusCounter(id);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldIncrementByOne() {
        PrometheusCounter counter = new PrometheusCounter(id);
        counter.increment();
        assertThat(counter.count()).isEqualTo(1.0);
    }

    @Test
    void shouldIncrementByAmount() {
        PrometheusCounter counter = new PrometheusCounter(id);
        counter.increment(5.5);
        assertThat(counter.count()).isEqualTo(5.5);
    }

    @Test
    void shouldAccumulateCumulatively() {
        PrometheusCounter counter = new PrometheusCounter(id);
        counter.increment(3.0);
        counter.increment(7.0);
        assertThat(counter.count()).isEqualTo(10.0);
    }

    @Test
    void shouldIgnoreNegativeIncrements() {
        PrometheusCounter counter = new PrometheusCounter(id);
        counter.increment(5.0);
        counter.increment(-1.0);
        assertThat(counter.count()).isEqualTo(5.0);
    }

    @Test
    void shouldIgnoreZeroIncrement() {
        PrometheusCounter counter = new PrometheusCounter(id);
        counter.increment(0.0);
        assertThat(counter.count()).isEqualTo(0.0);
    }

    @Test
    void shouldProduceCountMeasurement() {
        PrometheusCounter counter = new PrometheusCounter(id);
        counter.increment(42.0);

        assertThat(counter.measure()).hasSize(1);
        assertThat(counter.measure().iterator().next().getStatistic()).isEqualTo(Statistic.COUNT);
        assertThat(counter.measure().iterator().next().getValue()).isEqualTo(42.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusTimerTest.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.distribution.CountAtBucket;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class PrometheusTimerTest {

    private final Meter.Id id = new Meter.Id("test.timer", Tags.empty(),
            Meter.Type.TIMER, null, null);

    private PrometheusTimer createTimer(DistributionStatisticConfig config) {
        return new PrometheusTimer(id, Clock.SYSTEM, config);
    }

    @Test
    void shouldStartAtZero() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        assertThat(timer.count()).isEqualTo(0);
        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(0.0);
    }

    @Test
    void shouldRecordDuration() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        timer.record(100, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(100.0);
    }

    @Test
    void shouldAccumulateCumulatively() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(300, TimeUnit.MILLISECONDS);

        assertThat(timer.count()).isEqualTo(3);
        assertThat(timer.totalTime(TimeUnit.MILLISECONDS)).isEqualTo(600.0);
    }

    @Test
    void shouldReportTotalTimeInSeconds() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        timer.record(1500, TimeUnit.MILLISECONDS);

        assertThat(timer.totalTime(TimeUnit.SECONDS)).isEqualTo(1.5);
    }

    @Test
    void shouldHaveBaseTimeUnitOfSeconds() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        assertThat(timer.baseTimeUnit()).isEqualTo(TimeUnit.SECONDS);
    }

    @Test
    void shouldReturnEmptyHistogramWhenNoSlosConfigured() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        timer.record(100, TimeUnit.MILLISECONDS);

        HistogramSnapshot snapshot = timer.takeSnapshot();
        assertThat(snapshot.histogramCounts()).isEmpty();
        assertThat(snapshot.count()).isEqualTo(1);
    }

    @Test
    void shouldProduceCumulativeHistogramWithSlos() {
        DistributionStatisticConfig config = DistributionStatisticConfig.builder()
                .serviceLevelObjectives(
                        Duration.ofMillis(50).toNanos(),
                        Duration.ofMillis(100).toNanos(),
                        Duration.ofMillis(500).toNanos())
                .build()
                .merge(DistributionStatisticConfig.DEFAULT);

        PrometheusTimer timer = createTimer(config);

        // Record: 30ms, 80ms, 200ms, 800ms
        timer.record(30, TimeUnit.MILLISECONDS);
        timer.record(80, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);
        timer.record(800, TimeUnit.MILLISECONDS);

        HistogramSnapshot snapshot = timer.takeSnapshot();
        CountAtBucket[] counts = snapshot.histogramCounts();

        assertThat(counts).hasSize(3);

        // 30ms <= 50ms bucket -> cumulative count 1
        assertThat(counts[0].count()).isEqualTo(1);
        // 30ms, 80ms <= 100ms bucket -> cumulative count 2
        assertThat(counts[1].count()).isEqualTo(2);
        // 30ms, 80ms, 200ms <= 500ms bucket -> cumulative count 3
        assertThat(counts[2].count()).isEqualTo(3);
        // 800ms exceeds all buckets

        assertThat(snapshot.count()).isEqualTo(4);
    }

    @Test
    void shouldAccumulateHistogramAcrossMultipleRecordings() {
        DistributionStatisticConfig config = DistributionStatisticConfig.builder()
                .serviceLevelObjectives(
                        Duration.ofMillis(100).toNanos())
                .build()
                .merge(DistributionStatisticConfig.DEFAULT);

        PrometheusTimer timer = createTimer(config);

        timer.record(50, TimeUnit.MILLISECONDS);
        HistogramSnapshot snap1 = timer.takeSnapshot();
        assertThat(snap1.histogramCounts()[0].count()).isEqualTo(1);

        timer.record(50, TimeUnit.MILLISECONDS);
        HistogramSnapshot snap2 = timer.takeSnapshot();
        // Cumulative: should now be 2, not reset to 1
        assertThat(snap2.histogramCounts()[0].count()).isEqualTo(2);
    }

    @Test
    void shouldRecordDurationViaRecordRunnable() {
        PrometheusTimer timer = createTimer(DistributionStatisticConfig.DEFAULT);
        timer.record(() -> {
            // Simulate work
            try {
                Thread.sleep(1);
            }
            catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        assertThat(timer.count()).isEqualTo(1);
        assertThat(timer.totalTime(TimeUnit.NANOSECONDS)).isGreaterThan(0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusDistributionSummaryTest.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.distribution.CountAtBucket;
import dev.linhvu.micrometer.distribution.DistributionStatisticConfig;
import dev.linhvu.micrometer.distribution.HistogramSnapshot;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PrometheusDistributionSummaryTest {

    private final Meter.Id id = new Meter.Id("test.summary", Tags.empty(),
            Meter.Type.DISTRIBUTION_SUMMARY, null, null);

    private PrometheusDistributionSummary createSummary(DistributionStatisticConfig config) {
        return new PrometheusDistributionSummary(id, Clock.SYSTEM, 1.0, config);
    }

    @Test
    void shouldStartAtZero() {
        PrometheusDistributionSummary summary = createSummary(DistributionStatisticConfig.DEFAULT);
        assertThat(summary.count()).isEqualTo(0);
        assertThat(summary.totalAmount()).isEqualTo(0.0);
    }

    @Test
    void shouldRecordValues() {
        PrometheusDistributionSummary summary = createSummary(DistributionStatisticConfig.DEFAULT);
        summary.record(100.0);
        summary.record(200.0);

        assertThat(summary.count()).isEqualTo(2);
        assertThat(summary.totalAmount()).isEqualTo(300.0);
    }

    @Test
    void shouldReturnEmptyHistogramWhenNoSlosConfigured() {
        PrometheusDistributionSummary summary = createSummary(DistributionStatisticConfig.DEFAULT);
        summary.record(42.0);

        HistogramSnapshot snapshot = summary.takeSnapshot();
        assertThat(snapshot.histogramCounts()).isEmpty();
        assertThat(snapshot.count()).isEqualTo(1);
    }

    @Test
    void shouldProduceCumulativeHistogramWithSlos() {
        DistributionStatisticConfig config = DistributionStatisticConfig.builder()
                .serviceLevelObjectives(100.0, 500.0, 1000.0)
                .build()
                .merge(DistributionStatisticConfig.DEFAULT);

        PrometheusDistributionSummary summary = createSummary(config);

        summary.record(50.0);   // <= 100
        summary.record(200.0);  // <= 500
        summary.record(750.0);  // <= 1000
        summary.record(2000.0); // > 1000

        HistogramSnapshot snapshot = summary.takeSnapshot();
        CountAtBucket[] counts = snapshot.histogramCounts();

        assertThat(counts).hasSize(3);
        assertThat(counts[0].count()).isEqualTo(1);  // <= 100
        assertThat(counts[1].count()).isEqualTo(2);  // <= 500 (cumulative)
        assertThat(counts[2].count()).isEqualTo(3);  // <= 1000 (cumulative)

        assertThat(snapshot.count()).isEqualTo(4);
        assertThat(snapshot.total()).isEqualTo(3000.0);
    }

    @Test
    void shouldAccumulateHistogramAcrossMultipleRecordings() {
        DistributionStatisticConfig config = DistributionStatisticConfig.builder()
                .serviceLevelObjectives(100.0)
                .build()
                .merge(DistributionStatisticConfig.DEFAULT);

        PrometheusDistributionSummary summary = createSummary(config);

        summary.record(50.0);
        assertThat(summary.takeSnapshot().histogramCounts()[0].count()).isEqualTo(1);

        summary.record(50.0);
        // Cumulative: should now be 2
        assertThat(summary.takeSnapshot().histogramCounts()[0].count()).isEqualTo(2);
    }

    @Test
    void shouldApplyScaleFactor() {
        PrometheusDistributionSummary summary = new PrometheusDistributionSummary(
                id, Clock.SYSTEM, 2.0, DistributionStatisticConfig.DEFAULT);

        summary.record(100.0);

        assertThat(summary.count()).isEqualTo(1);
        assertThat(summary.totalAmount()).isEqualTo(200.0);
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/prometheus/PrometheusMeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.prometheus;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Timer;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

class PrometheusMeterRegistryTest {

    private PrometheusMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
    }

    @Test
    void shouldScrapeCounter() {
        Counter counter = registry.counter("http.requests", "method", "GET");
        counter.increment(42);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE http_requests_total counter");
        assertThat(scrape).contains("http_requests_total{method=\"GET\"} 42");
    }

    @Test
    void shouldScrapeCounterWithDescription() {
        Counter.builder("api.calls")
                .description("Total API calls")
                .register(registry)
                .increment(5);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# HELP api_calls_total Total API calls");
        assertThat(scrape).contains("# TYPE api_calls_total counter");
        assertThat(scrape).contains("api_calls_total 5");
    }

    @Test
    void shouldNotDoubleAppendTotal() {
        Counter counter = Counter.builder("requests.total").register(registry);
        counter.increment();

        String scrape = registry.scrape();
        assertThat(scrape).contains("requests_total ");
        assertThat(scrape).doesNotContain("requests_total_total");
    }

    @Test
    void shouldScrapeGauge() {
        AtomicLong value = new AtomicLong(42);
        registry.gauge("jvm.memory.used", value);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE jvm_memory_used gauge");
        assertThat(scrape).contains("jvm_memory_used 42");
    }

    @Test
    void shouldScrapeTimerAsSummary_WhenNoHistogram() {
        Timer timer = Timer.builder("http.request")
                .description("HTTP request duration")
                .register(registry);

        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# HELP http_request_seconds HTTP request duration");
        assertThat(scrape).contains("# TYPE http_request_seconds summary");
        assertThat(scrape).contains("http_request_seconds_count 2");
        assertThat(scrape).contains("http_request_seconds_sum 0.3");
        assertThat(scrape).contains("# TYPE http_request_seconds_max gauge");
    }

    @Test
    void shouldScrapeTimerAsHistogram_WhenSlosConfigured() {
        Timer timer = Timer.builder("http.request")
                .serviceLevelObjectives(
                        Duration.ofMillis(50),
                        Duration.ofMillis(100),
                        Duration.ofMillis(500))
                .register(registry);

        timer.record(30, TimeUnit.MILLISECONDS);
        timer.record(80, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE http_request_seconds histogram");
        assertThat(scrape).contains("http_request_seconds_bucket{le=\"0.05\"} 1");
        assertThat(scrape).contains("http_request_seconds_bucket{le=\"0.1\"} 2");
        assertThat(scrape).contains("http_request_seconds_bucket{le=\"0.5\"} 3");
        assertThat(scrape).contains("http_request_seconds_bucket{le=\"+Inf\"} 3");
        assertThat(scrape).contains("http_request_seconds_count 3");
        assertThat(scrape).contains("http_request_seconds_sum");
    }

    @Test
    void shouldScrapeTimerWithTags() {
        Timer timer = Timer.builder("http.request")
                .tag("method", "GET")
                .tag("status", "200")
                .register(registry);

        timer.record(50, TimeUnit.MILLISECONDS);

        String scrape = registry.scrape();

        assertThat(scrape).contains(
                "http_request_seconds_count{method=\"GET\",status=\"200\"} 1");
    }

    @Test
    void shouldScrapeDistributionSummaryAsSummary() {
        DistributionSummary summary = DistributionSummary.builder("http.request.size")
                .baseUnit("bytes")
                .register(registry);

        summary.record(1024);
        summary.record(2048);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE http_request_size_bytes summary");
        assertThat(scrape).contains("http_request_size_bytes_count 2");
        assertThat(scrape).contains("http_request_size_bytes_sum 3072");
    }

    @Test
    void shouldScrapeDistributionSummaryAsHistogram_WhenSlosConfigured() {
        DistributionSummary summary = DistributionSummary.builder("http.request.size")
                .baseUnit("bytes")
                .serviceLevelObjectives(1024, 4096)
                .register(registry);

        summary.record(512);
        summary.record(2048);
        summary.record(8192);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE http_request_size_bytes histogram");
        assertThat(scrape).contains("http_request_size_bytes_bucket{le=\"1024\"} 1");
        assertThat(scrape).contains("http_request_size_bytes_bucket{le=\"4096\"} 2");
        assertThat(scrape).contains("http_request_size_bytes_bucket{le=\"+Inf\"} 3");
        assertThat(scrape).contains("http_request_size_bytes_count 3");
    }

    @Test
    void shouldScrapeFunctionCounter() {
        AtomicLong count = new AtomicLong(100);
        registry.more().counter("cache.evictions",
                java.util.Collections.emptyList(), count, AtomicLong::doubleValue);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE cache_evictions_total counter");
        assertThat(scrape).contains("cache_evictions_total 100");
    }

    @Test
    void shouldScrapeFunctionTimer() {
        Object source = new Object();
        registry.more().timer("pool.wait",
                java.util.Collections.emptyList(),
                source,
                obj -> 10L,
                obj -> 5000.0,
                TimeUnit.MILLISECONDS);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE pool_wait_seconds summary");
        assertThat(scrape).contains("pool_wait_seconds_count 10");
        assertThat(scrape).contains("pool_wait_seconds_sum 5");
    }

    @Test
    void shouldScrapeLongTaskTimer() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import")
                .register(registry);

        LongTaskTimer.Sample sample = ltt.start();

        String scrape = registry.scrape();

        assertThat(scrape).contains("# TYPE batch_import_seconds_active_count gauge");
        assertThat(scrape).contains("batch_import_seconds_active_count 1");
        assertThat(scrape).contains("# TYPE batch_import_seconds_duration_sum gauge");
        assertThat(scrape).contains("# TYPE batch_import_seconds_max gauge");

        sample.stop();
    }

    @Test
    void shouldGroupMetersByConventionName() {
        registry.counter("http.requests", "method", "GET").increment(10);
        registry.counter("http.requests", "method", "POST").increment(5);

        String scrape = registry.scrape();

        assertThat(scrape).contains("http_requests_total{method=\"GET\"} 10");
        assertThat(scrape).contains("http_requests_total{method=\"POST\"} 5");
    }

    @Test
    void shouldOmitHelpWhenDescriptionsDisabled() {
        PrometheusConfig noDescConfig = key -> {
            if ("prometheus.descriptions".equals(key)) {
                return "false";
            }
            return null;
        };
        PrometheusMeterRegistry reg = new PrometheusMeterRegistry(noDescConfig);

        Counter.builder("api.calls")
                .description("Total API calls")
                .register(reg)
                .increment();

        String scrape = reg.scrape();

        assertThat(scrape).doesNotContain("# HELP");
        assertThat(scrape).contains("# TYPE api_calls_total counter");
    }

    @Test
    void shouldFormatWholeNumbersWithoutDecimalPoint() {
        assertThat(PrometheusMeterRegistry.formatDouble(42.0)).isEqualTo("42");
        assertThat(PrometheusMeterRegistry.formatDouble(0.0)).isEqualTo("0");
    }

    @Test
    void shouldFormatDecimalNumbers() {
        assertThat(PrometheusMeterRegistry.formatDouble(3.14)).isEqualTo("3.14");
        assertThat(PrometheusMeterRegistry.formatDouble(0.001)).isEqualTo("0.001");
    }

    @Test
    void shouldFormatSpecialValues() {
        assertThat(PrometheusMeterRegistry.formatDouble(Double.NaN)).isEqualTo("NaN");
        assertThat(PrometheusMeterRegistry.formatDouble(Double.POSITIVE_INFINITY)).isEqualTo("+Inf");
        assertThat(PrometheusMeterRegistry.formatDouble(Double.NEGATIVE_INFINITY)).isEqualTo("-Inf");
    }

    @Test
    void shouldScrapeEmptyRegistry() {
        assertThat(registry.scrape()).isEmpty();
    }

    @Test
    void shouldEscapeQuotesInLabelValues() {
        registry.counter("test", "path", "/api/\"quoted\"").increment();

        String scrape = registry.scrape();
        assertThat(scrape).contains("path=\"/api/\\\"quoted\\\"\"");
    }

    @Test
    void shouldEscapeBackslashesInLabelValues() {
        registry.counter("test", "path", "C:\\Users").increment();

        String scrape = registry.scrape();
        assertThat(scrape).contains("path=\"C:\\\\Users\"");
    }

    @Test
    void shouldExposeContentType() {
        assertThat(PrometheusMeterRegistry.CONTENT_TYPE_004)
                .isEqualTo("text/plain; version=0.0.4; charset=utf-8");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/integration/PrometheusRegistryIntegrationTest.java` [NEW]

```java
package dev.linhvu.micrometer.integration;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.config.MeterFilter;
import dev.linhvu.micrometer.prometheus.PrometheusConfig;
import dev.linhvu.micrometer.prometheus.PrometheusMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests that verify the Prometheus registry works correctly
 * with other Micrometer features: naming conventions, meter filters,
 * and mixed meter types.
 */
class PrometheusRegistryIntegrationTest {

    private PrometheusMeterRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
    }

    @Test
    void shouldProduceCompletePrometheusOutput_WhenMultipleMeterTypesRegistered() {
        Counter counter = Counter.builder("http.requests")
                .description("Total HTTP requests")
                .tag("method", "GET")
                .register(registry);
        counter.increment(100);

        Timer timer = Timer.builder("http.request.duration")
                .description("HTTP request duration")
                .serviceLevelObjectives(
                        Duration.ofMillis(50),
                        Duration.ofMillis(100))
                .tag("method", "GET")
                .register(registry);
        timer.record(30, TimeUnit.MILLISECONDS);
        timer.record(80, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        DistributionSummary summary = DistributionSummary.builder("http.request.size")
                .baseUnit("bytes")
                .register(registry);
        summary.record(1024);

        String scrape = registry.scrape();

        assertThat(scrape).contains("# HELP http_requests_total Total HTTP requests");
        assertThat(scrape).contains("# TYPE http_requests_total counter");
        assertThat(scrape).contains("http_requests_total{method=\"GET\"} 100");
        assertThat(scrape).contains("# TYPE http_request_duration_seconds histogram");
        assertThat(scrape).contains(
                "http_request_duration_seconds_bucket{method=\"GET\",le=\"0.05\"} 1");
        assertThat(scrape).contains(
                "http_request_duration_seconds_bucket{method=\"GET\",le=\"0.1\"} 2");
        assertThat(scrape).contains(
                "http_request_duration_seconds_bucket{method=\"GET\",le=\"+Inf\"} 3");
        assertThat(scrape).contains(
                "http_request_duration_seconds_count{method=\"GET\"} 3");
        assertThat(scrape).contains("# TYPE http_request_size_bytes summary");
        assertThat(scrape).contains("http_request_size_bytes_count 1");
    }

    @Test
    void shouldApplyCommonTags_WhenMeterFilterConfigured() {
        registry.meterFilter(MeterFilter.commonTags(Tags.of("app", "myapp")));

        registry.counter("requests").increment();

        String scrape = registry.scrape();
        assertThat(scrape).contains("app=\"myapp\"");
    }

    @Test
    void shouldDenyMeters_WhenDenyFilterConfigured() {
        registry.meterFilter(MeterFilter.denyNameStartsWith("internal"));

        registry.counter("internal.counter").increment();
        registry.counter("external.counter").increment();

        String scrape = registry.scrape();
        assertThat(scrape).doesNotContain("internal");
        assertThat(scrape).contains("external_counter_total");
    }

    @Test
    void shouldRenameMeters_WhenMapFilterConfigured() {
        registry.meterFilter(MeterFilter.renameTag("http.requests", "status", "http_status"));

        registry.counter("http.requests", "status", "200").increment();

        String scrape = registry.scrape();
        assertThat(scrape).contains("http_status=\"200\"");
    }

    @Test
    void shouldApplyPrometheusNamingConvention() {
        Timer timer = Timer.builder("my.timer").register(registry);
        timer.record(1, TimeUnit.SECONDS);

        Counter counter = Counter.builder("my.counter").register(registry);
        counter.increment();

        String scrape = registry.scrape();

        assertThat(scrape).contains("my_timer_seconds");
        assertThat(scrape).contains("my_counter_total");
    }

    @Test
    void shouldShowCumulativeValues_AcrossMultipleScrapes() {
        Counter counter = registry.counter("requests");
        counter.increment(10);

        String scrape1 = registry.scrape();
        assertThat(scrape1).contains("requests_total 10");

        counter.increment(5);

        String scrape2 = registry.scrape();
        assertThat(scrape2).contains("requests_total 15");
    }

    @Test
    void shouldShowCumulativeHistogram_AcrossMultipleScrapes() {
        Timer timer = Timer.builder("latency")
                .serviceLevelObjectives(Duration.ofMillis(100))
                .register(registry);

        timer.record(50, TimeUnit.MILLISECONDS);
        String scrape1 = registry.scrape();
        assertThat(scrape1).contains("latency_seconds_bucket{le=\"0.1\"} 1");

        timer.record(50, TimeUnit.MILLISECONDS);
        String scrape2 = registry.scrape();
        assertThat(scrape2).contains("latency_seconds_bucket{le=\"0.1\"} 2");
    }

    @Test
    void shouldConvertHistogramBoundariesToSeconds() {
        Timer timer = Timer.builder("request.duration")
                .serviceLevelObjectives(
                        Duration.ofMillis(5),
                        Duration.ofMillis(10),
                        Duration.ofMillis(25))
                .register(registry);

        timer.record(3, TimeUnit.MILLISECONDS);

        String scrape = registry.scrape();

        assertThat(scrape).contains("le=\"0.005\"");
        assertThat(scrape).contains("le=\"0.01\"");
        assertThat(scrape).contains("le=\"0.025\"");
    }

    @Test
    void shouldReportTimerTotalsInSeconds() {
        Timer timer = registry.timer("my.timer");
        timer.record(1500, TimeUnit.MILLISECONDS);

        String scrape = registry.scrape();
        assertThat(scrape).contains("my_timer_seconds_sum 1.5");
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Pull model** | The monitoring server (Prometheus) fetches metrics by calling `scrape()` — the application is passive, no scheduler needed |
| **Cumulative histogram** | Each bucket count includes ALL observations since application start; Prometheus computes rates server-side from two consecutive scrapes |
| **Text exposition format** | Human-readable `# TYPE`/`# HELP` headers followed by `metric{labels} value` lines; simple to debug with `curl` |
| **`_total` suffix** | Prometheus convention for counters — signals that the value only goes up |
| **`le` label** | "Less than or equal" — used in histogram bucket lines to define the upper boundary |
| **`+Inf` bucket** | Mandatory catch-all histogram bucket containing the total count of all observations |

**Next: Chapter 21 — Datadog Registry** — A concrete HTTP push registry that serializes metrics as JSON and POSTs them to the Datadog API, demonstrating the step-based push model with real-world API integration.
